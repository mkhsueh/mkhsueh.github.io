---
layout: post
title: "Performance Engineering with a Pipelined Directed Acyclic Graph Pattern"
date: 2015-05-13 17:17:47 -0700
comments: true
categories: [concurrency, performance engineering, Java]
published: true
---

On a recent large-scale data migration project, my team encountered throughput issues for a service being developed in our data pipeline. This particular service drove our automated migration, and we needed to scale by orders of magnitude in order to meet the launch schedule. Beginning with addressing bottlenecks and tuning runtime parameters, various performance improvements were made; an optimization that helped us go the distance was rethinking single-threaded sequential execution as a pipelined directed acyclic graph execution.

<!-- more -->

The particular service under discussion works by continuously streaming and processing data in batches. Up until the point of considering architectural redesign, various parts of the application had been tuned or otherwise parallelized as much as our network/hardware could support. There would be little to no performance benefit from throwing more threads at the issue in a brute-force manner. The key observation at this point was that we would need our slowest I/O-bound processes to run not only as quickly as possible, but as much of the time as possible.
 
One main loop consisting of several major tasks was executing sequentially. By multithreading the entire main process, we could process multiple batches simultaneously. Conveniently, the main process was already encapsulated as a job. A single threaded execution was done via <a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/concurrent/ThreadPoolTaskExecutor.html">ThreadPoolTaskExecutor.scheduleWithFixedDelay()</a> to allow the entire batch to finish before repeating. What we needed to do here was first change the ThreadPoolTaskExecutor to use scheduleAtFixedRate() and allow concurrent executions. To guard against excessive memory usage by queued tasks, a discard policy was configured for this top-level executor. 

Now, what does that change accomplish? We've achieved more parallelism, but as stated before, parallelism for individual stages was already achieved. The next step was to actually "gate" each stage in the process to force pipelining. To do this, a couple things needed to happen:
</p>
1) each stage was encapsulated as a new task </p>
// talk about Guava, futures, chaining, transform, async function

2) a separate <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> was used to run each stage

//TODO talk about performance improvement
// picture of sequentuial tasks

// picture of actual task graph
// why pick 2 threads?

// pipeline to always get the critical path moving as much as possible. no downtime
// reasons that tasks can't be in parallel? 

// pipelining, not parallel

// how to break into pipeline? not just sequential, can be made into graph for more parallelism

// two threads running stuff. some executor stages allow 2 threads, the critical ones allow 1

     


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
            /* use dependencies */
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
        
        /* handle independent task */
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
