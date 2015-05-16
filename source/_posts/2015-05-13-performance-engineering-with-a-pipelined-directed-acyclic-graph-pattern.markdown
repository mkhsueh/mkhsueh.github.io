---
layout: post
title: "Performance Engineering with a Pipelined Directed Acyclic Graph Pattern"
date: 2015-05-13 17:17:47 -0700
comments: true
categories: [concurrency, performance engineering, Java]
published: true
---

On a recent large-scale data migration project, my team encountered throughput issues for a service being developed in our data pipeline. This particular service drove our automated migration, and we needed to scale by orders of magnitude in order to meet the launch schedule. Beginning with addressing bottlenecks and tuning runtime parameters, various performance improvements were made; an optimization that helped us go the distance was rethinking single-threaded sequential execution as a pipelined directed acyclic graph (DAG) execution.

<!-- more -->

The particular service under discussion works by continuously streaming and processing data in batches. Up until the point of considering architectural redesign, various parts of the application had been tuned or otherwise parallelized as much as our network/hardware could support. There would be little to no performance benefit from throwing more threads at the issue in a brute-force manner. The key observation at this point was that we would need our slowest I/O-bound processes to run not only as quickly as possible, but as much of the time as possible.
 
One main loop consisting of several major tasks was executing sequentially. By multithreading the entire main process, we could process multiple batches simultaneously. Conveniently, the main process was already encapsulated as a job. A single threaded execution was done via <a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/concurrent/ThreadPoolTaskExecutor.html">ThreadPoolTaskExecutor.scheduleWithFixedDelay()</a> to allow the entire batch to finish before repeating. What we needed to do here was first change the ThreadPoolTaskExecutor to use scheduleAtFixedRate() and allow concurrent executions. To guard against excessive memory usage by queued tasks, a discard policy was configured for this top-level executor. 

Now, what does that accomplish? We're adding more parallelism at the task level, but as stated before, parallelism for individual stages was already achieved by other optimizations. The next step was to actually "gate" each stage in the process to force pipelining and parallelization of different tasks. To do this, some changes needed to happen:
</p>
**1)** Each stage was encapsulated inside a Callable. Callables for intermediate stages were further wrapped inside an <a href="http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/util/concurrent/AsyncFunction.html">AsyncFunction</a>.

</p>

**2)** A separate <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> was used to run each stage. Native Java ExecutorService instances were converted into Guava's ListeningExecutorService to support successive chaining of tasks (via <a href="http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/util/concurrent/ListenableFuture.html">ListenableFuture</a> instances returned). Using separate executors for each task allowed different types of tasks to run   in parallel. 

Simply running multiple threads through the execution of all tasks at once might have produced this behavior. However, using that approach does not guarantee whether executions end up staggered across I/O intensive stages or all within the same stage.

**3)** Splitting sequential tasks into a graph:
{% img center /images/DAG_conversion.png 'Conversion of Sequential Tasks into Task Graph' 'Conversion of Sequential Tasks into Task Graph'%}

Pipelining in itself is certainly an improvement. Now that tasks were encapsulated as Callables, independent tasks could be run in parallel. Various stages are kicked off and joined using ```transform()``` and ```get()```; intermediate stages are represented as an AsyncFunction used to transform results from dependency tasks. The image above shows before/after arrangement of tasks.

``` java fork-join example using ListenableFuture

 private final ListeningExecutorService executorServiceA = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(2));
 private final ListeningExecutorService executorServiceB = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(2));

// ...

        ListenableFuture<List<Result>> taskAFuture = executorServiceA
                .submit(new Callable<List<Result>>() {
                    public List<Result> call() {
                        final List<Result> result = attemptOperation();
                        // business logic, return result
                    }
                });

        AsyncFunction<List<Result>, List<Integer>> functionB = new AsyncFunction<List<Result>, List<Integer>>() {

            @Override
            public ListenableFuture<List<Integer>> apply(
                    final List<Result> input) throws Exception {
                return executorServiceB
                        .submit(new Callable<List<Integer>>() {

                            @Override
                            public List<Integer> call() throws Exception {
                                // business logic, return result
                            }
                        });
            }
        };

        ListenableFuture<List<Integer>> intermediateFuture = Futures.transform(taskAFuture, functionB);

        // other tasks are initialized, with varying degrees of fanout and joining

        Futures.allAsList(finalFutureX, finalFutureY).get();

```

An initial solution was delivered under operational/timeline constraints with the above approach. Just imagine more stages with more interdependencies. Examined in isolation, the result was an increase in throughput, by a factor of approximately 1.83x. Keep in mind, an architectural reorganization can compound upon smaller changes; combined with other optimizations made to this app, we've seen up to a 132x peak gain in production throughput.

**An aside on tuning thread count and executor services for various stages:**

I believe in using threads judiciously (I know, it's tempting to add an extra 0 to that thread config). How one does this exactly depends on the application in question. For our particular case, there were two heavily I/O bound stages, with the others being negligible or relatively quick. The number of threads executing the graph were set to two-- allowing each expensive stage to run nearly continuously. Furthermore, the executors for those expensive stages were configured with a thread pool size of one. As a test, the executors' pool sizes were increased to two. This actually caused a degradation in performance by 0.877s for a sample dataset; To understand why this happens, let's look at the visualization of application state. The chart below (part of a monitoring dashboard built using Splunk) shows what happened when the two stages were constrained to one thread each. Tasks represented by the orange bars represent the second most time-intensive stage. Once our pipeline execution fully becomes staggered, we should expect to get the cost of stage-orange for free (on a timescale only), except for the initial occurrence. In other words, that task's execution is overshadowed by the most expensive stage. 

{% img center /images/flipper_state_diagram_one_thread.png 'Application State Transition Diagram'%}

Once thread count is increased, the two threads may concurrently attempt to perform the same stage, splitting available resources. If we examine the total cost of N-1 stage-orange runs, we see that they roughly add up to the one second increase in runtime.  

     estimated times for ops, not counting the initial execution:
     0.102s + 0.108s + 0.119s + 0.1s + 0.121s + 0.094s
     total = 0.644s

     difference in trial run times: 17.474-16.597 = 0.877s

     0.644s accounts for most of the difference in performance
     - of course, there may have been some variability due to network performance

**Generalizing the design:**

Manually stringing up a series of asynchronous tasks and transformations isn't exactly the picture of maintainability and extensibility we want to see in our applications. The next challenge was refactoring this process to make it reuseable and configurable. We can decompose this pattern into the following:
         
- specification of tasks as a graph
- chaining and joining of tasks

Each "stage" or task was rewritten as a <a href="http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/base/Function.html">Function</a><I, O> class. Specifying a task graph can be done by specifying the list of dependencies for each stage, along with the work encapsulated within a Function, and the executor service for each stage. That gives us all the specifications we need to initiate a stage. Essentially, we used a modified adjacency list representation for a graph.

To automatically initiate tasks in dependency order, a depth-first search was performed from each task in the graph. No task begins executing until dependencies complete, and results from the dependencies are passed into the function. We could have done a topological sort to determine dependency ordering. However, the implementation was simplified using the design assumption that crawling the graph and initializing tasks in memory is relatively inexpensive (on the order of milliseconds, and happens once per graph execution).

We also needed to validate the graph as an acyclic graph; cycles could cause tasks to block on execution, or even a stack overflow error from recursively initializing dependencies. The graph is first validated by a custom ```AcyclicGraph``` class, which is then consumed by a ```TaskGraphExecutor``` that executes the graph. Internally, the ```AcyclicGraph``` is backed by a task graph mapping (adjacency list-like representation discussed earlier). The task graph mapping is created in Spring via XML configuration as a map. Excluding writing new services, we can configure executor services for each stage, reconfigure the graph, or incorporate new tasks fairly flexibly-- all without touching any Java code.



The ```TaskGraphExecutor``` strings everything together. You'll notice that working with ```ListenableFuture``` gives us the added convenience of choosing an asynchronous or synchronous computation of the overall graph, exposed by ```computeGraphAsync()``` and ```computeGraphSynchronously()```.      

``` java executor for an acyclic task graph https://gist.github.com/mkhsueh/d75cd6bc2ed3b61a947f gist

package com.mkhsueh.example.pipeline;
 
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ExecutionException;
 
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
import com.google.common.collect.Lists;
import com.google.common.util.concurrent.AsyncFunction;
import com.google.common.util.concurrent.Futures;
import com.google.common.util.concurrent.ListenableFuture;
 
import static com.google.common.base.Preconditions.*;
 
/**
 * Executes a directed acyclic graph of tasks. Thread-safe. 
 *
 * @param <I> the return type of tasks
 * @author Michael K Hsueh
 */
public class TaskGraphExecutor <I> {
 
    private static final Logger LOG = LoggerFactory.getLogger(TaskGraphExecutor.class); 
    private final AcyclicGraph<TaskGraphTuple<List<I>, I>> taskGraph;
    
    /**
     * 
     * @param taskGraph
     *            represents a description of a directed acyclic graph of tasks.
     *            Keys are vertexes, and each {@link TaskGraphTuple} contains a
     *            listing of the dependencies for that vertex--edges flow from
     *            the dependencies to the current vertex. Task graphs with
     *            cycles are considered invalid and will not execute.
     */
    public TaskGraphExecutor(
            AcyclicGraph<TaskGraphTuple<List<I>, I>> taskGraph) {
        this.taskGraph = checkNotNull(taskGraph);
    }
 
    /**
     * Initializes task graph and returns an aggregate future. Caller should
     * block on future.get() to wait for completion.
     * 
     * @return aggregate future
     */
    public ListenableFuture<List<I>> computeGraphAsync() {
 
        Map<String, ListenableFuture<I>> futuresMapping = new HashMap<String, ListenableFuture<I>>();
        
        for (String key : taskGraph.keySet()) {
            populateFuture(key, futuresMapping);
        }
        
        return Futures.allAsList(futuresMapping.values());
    }
    
    /**
     * Returns final result list from the task graph once all tasks complete.
     * The actual graph execution can still run with parallelization as
     * configured.
     * 
     * @return aggregate results
     * @throws InterruptedException
     * @throws ExecutionException
     */
    public List<I> computeGraphSynchronously() throws InterruptedException, ExecutionException {
        return computeGraphAsync().get();
    }
    
    private ListenableFuture<I> populateFuture(String key, Map<String, ListenableFuture<I>> futuresMapping) {
        final TaskGraphTuple<List<I>, I> tuple = taskGraph.get(key);
        List<String> dependencyKeys = tuple.getDependencyKeys();
 
        if (notStarted(key, futuresMapping)) {
            // use dependencies
            AsyncFunction<List<I>, I> taskAsFxn = tuple.getAsyncFunction();
            try {
                ListenableFuture<List<I>> dependencies = startAndGetAggregateDependencies(dependencyKeys, futuresMapping);
                futuresMapping.put(key, Futures.transform(dependencies, taskAsFxn));
            } catch (ClassCastException e) {
                String msg = String.format("Unexpected input types from dependency tasks [%s] for task [%s]", dependencyKeys, key);
                LOG.error(msg, e);
                throw new RuntimeException(msg, e);
            } catch (NullPointerException e) {
                String msg = String.format("Null pointer given dependency tasks [%s] for task [%s]; check configuration to ensure required dependencies were specified for the task",
                                dependencyKeys, key);
                LOG.error(msg, e);
                throw new RuntimeException(msg, e);
            }
        }
        
        return futuresMapping.get(key);
    }
    
    private boolean notStarted(String key,
            Map<String, ListenableFuture<I>> futuresMapping) {
        return futuresMapping.get(key) == null;
    }
 
    private ListenableFuture<List<I>> startAndGetAggregateDependencies(List<String> dependencyKeys,
            Map<String, ListenableFuture<I>> futuresMapping) {
        List<ListenableFuture<I>> dependencyFutures = Lists.newArrayList();
        
        // handle independent task
        if (dependencyKeys.isEmpty()) {
            return Futures.immediateFuture(null); // null avoids cast errors;
                                                  // reasonably safe
                                                  // since independent tasks
                                                  // should not read inputs
        }
        
        for (String key : dependencyKeys) {
            if (notStarted(key, futuresMapping)) {
                populateFuture(key, futuresMapping);
            } 
        }
        
        for (String key : dependencyKeys) {
            ListenableFuture<I> dependencyFuture = futuresMapping.get(key);
            dependencyFutures.add(dependencyFuture);
        }
 
        return Futures.allAsList(dependencyFutures);     
    }
 
}
```

Below is the helper class ```TaskGraphTuple```, which encapsulates converting ```Function``` to ```AsyncFunction```.

```java TaskGraphTuple.java

package com.mkhsueh.example.pipeline;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;

import com.google.common.base.Function;
import com.google.common.util.concurrent.AsyncFunction;
import com.google.common.util.concurrent.ListenableFuture;
import com.google.common.util.concurrent.ListeningExecutorService;
import com.google.common.util.concurrent.MoreExecutors;

import static com.google.common.base.Preconditions.*;

public class TaskGraphTuple<I, O> {

    private final List<String> dependencyKeys;
    private final Function<I, O> function;
    private final ListeningExecutorService executorService;
    private AsyncFunction<I, O> asyncFunction;
    
    public TaskGraphTuple(List<String> dependencyKeys, Function<I, O> task, ExecutorService executorService) {
        this.dependencyKeys = checkNotNull(dependencyKeys);
        this.function = checkNotNull(task);
        this.executorService = MoreExecutors.listeningDecorator(checkNotNull(executorService));
        this.asyncFunction = createAsyncFunction(this.function, this.executorService);
    }

    private AsyncFunction<I, O> createAsyncFunction(
            final Function<I, O> fxn,
            final ListeningExecutorService executorService) {
        return new AsyncFunction<I, O>() {

            @Override
            public ListenableFuture<O> apply(final I input) throws Exception {
                return executorService.submit(new Callable<O>() {

                    @Override
                    public O call() throws Exception {
                        return fxn.apply(input);
                    }
                });
            }

        };
    }

    public List<String> getDependencyKeys() {
        return Collections.unmodifiableList(dependencyKeys);
    }

    public Function<I, O> getFunction() {
        return function;
    }

    public ListeningExecutorService getExecutorService() {
        return executorService;
    }
    
    public AsyncFunction<I, O> getAsyncFunction() {
        return asyncFunction;
    }

}

```

Lastly, here's an example configuration file. All the configuration is done in one file here for brevity.

```xml example Spring configuration file

<!-- pipeline stages -->

<bean id="serviceA" class="com.mkhsueh.example.pipeline.ServiceA">
      <constructor-arg ref="constructorArg1"/>
      <constructor-arg ref="constructorArg2"/>
</bean>

<bean id="serviceB" class="com.mkhsueh.example.pipeline.ServiceB">
      <constructor-arg ref="constructorArg1"/>
      <constructor-arg ref="constructorArg2"/>
</bean>

<!-- task graph setup -->

<bean id="executorA" class="com.mkhsueh.example.concurrent.ExecutorServiceFactory" factory-method="newFixedThreadPool">
      <constructor-arg value="${serviceA.jobThreadPoolCount}" />
      <constructor-arg value="${serviceA.jobThreadPoolName}" />
</bean>

<bean id="executorB" class="com.mkhsueh.example.concurrent.ExecutorServiceFactory" factory-method="newFixedThreadPool">
      <constructor-arg value="${serviceB.jobThreadPoolCount}" />
      <constructor-arg value="${serviceB.jobThreadPoolName}" />
</bean>

<bean id="tupleA"  class="com.mkhsueh.example.pipeline.TaskGraphTuple">
      <constructor-arg>
        <list></list>  <!-- dependency tasks go here -->
      </constructor-arg>
      <constructor-arg ref="serviceA" />
      <constructor-arg ref="executorA" />
</bean>

<bean id="tupleB" class="com.mkhsueh.example.pipeline.TaskGraphTuple">
      <constructor-arg>
        <list>
                <value>serviceA</value>
        </list>
      </constructor-arg>
      <constructor-arg ref="serviceB" />
      <constructor-arg ref="executorB" />
</bean>

<bean id="taskGraphMapping" class="java.util.HashMap">
      <constructor-arg>
        <map key-type="java.lang.String" value-type="com.mkhsueh.example.pipeline.TaskGraphTuple">
             <entry key="serviceA" value-ref="tupleA" />
             <entry key="serviceB" value-ref="tupleB" />
        </map>
      </constructor-arg>
</bean>

<bean id="acyclicTaskGraph" class="com.mkhsueh.example.pipeline.AcyclicTaskGraph">
      <constructor-arg ref="taskGraphMapping" />
</bean>

<bean id="taskGraphExecutor" class="com.mkhsueh.example.pipeline.TaskGraphExecutor">
      <constructor-arg ref="acyclicTaskGraph" />
</bean>
```
