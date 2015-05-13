---
layout: post
title: "Environment Namespacing with MyBatis and Spring"
date: 2015-01-26 19:16:47 -0800
comments: true
categories: mybatis,spring,namespace,namespaces
---

While working with a recent customer, a set of database tables had been namespaced differently in the QA and production environments. Ideally, the namespaces should mirror one another. Unfortunately the DB replication topology was constrained to one slave host, and there would have been a namespace collision. Shucks! In this post, I'll explain how we managed namespaces for different environments using the MyBatis mapping layer with Spring configuration.

<!-- more -->

By decomposing the XML configuration, it's possible to select desired namespaces via Spring profiles. For reference, Spring profiles effectively group bean definitions, which are instantiated when the profiles are invoked.

We start by examining sample MyBatis mapper files. Let's say we want to handle multiple namespaces for the "object" table:

```xml common-methods.xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.mkhsueh.example.mapper">

	<sql id="excludeFailed">
        LEFT JOIN pp.object o
        ON (failureCandidates.customer_id = o.source_id)
        LEFT JOIN pp.object_type ot
        ON ot.object_type_id = o.object_type_id 
        WHERE o.source_id IS NULL OR
        (ot.type = 'Purchase' AND o.failure_count = 0)
	</sql>

</mapper>

```

A <sql> tag denotes SQL fragments that can be reused within the same file, or any other file loaded by the same session factory. The first thing we do is abstract the namespace into a SQL fragment, such that we can refer to it via the refid.


Represent the namespace fragments in their own configuration files:

```xml namespace-qa.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.mkhsueh.example.mapper.namespace">

    <sql id="purchase">
        pq.
    </sql>

</mapper>
```
 
```xml namespace-prod.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.mkhsueh.example.mapper.namespace">

    <sql id="purchase">
        pp.
    </sql>

</mapper>
```

Now we substitute the hardcoded namespace with references:

```xml common-methods.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.mkhsueh.example.mapper">

	<sql id="excludeFailed">
        LEFT JOIN <include refid="com.mkhsueh.example.mapper.namespace.purchase" />object o
        ON (failureCandidates.customer_id = o.source_id)
        LEFT JOIN <include refid="com.mkhsueh.example.mapper.namespace.purchase" />object_type ot
        ON ot.object_type_id = o.object_type_id 
        WHERE o.source_id IS NULL OR
        (ot.type = 'Purchase' AND o.failure_count = 0)
	</sql>

</mapper>
```

At this step, the key observation is that our mapping files are encapsulated in MyBatis session factories, and those factories can be tied to profiles.

```xml session-factories.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!-- imports -->
    <import resource="classpath:/spring/spring-datasource.xml"/>
    
    <!-- dao; by default, use prod namespace -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="mapperLocations">
            <list>
                <value>classpath:/mapper/service-methods.xml</value>
                <value>classpath:/mapper/common-methods.xml</value>
                <value>classpath:/mapper/namespace-prod.xml</value>
            </list>
        </property>
        <property name="typeHandlers">
            <list>
                <bean class="com.mkhsueh.example.DateTimeTypeHandler" />
            </list>
        </property>
    </bean>
    
    <!-- profile to use prod namespace -->
    <beans profile="prod">
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="mapperLocations">
            <list>
                <value>classpath:/mapper/service-methods.xml</value>
                <value>classpath:/mapper/common-methods.xml</value>
                <value>classpath:/mapper/namespace-prod.xml</value>
            </list>
        </property>
        <property name="typeHandlers">
            <list>
                <bean class="com.mkhsueh.example.DateTimeTypeHandler" />
            </list>
        </property>
    </bean>
    </beans>  
    
    <!-- profile to use qa namespace -->
    <beans profile="qa">
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="mapperLocations">
            <list>
                <value>classpath:/mapper/service-methods.xml</value>
                <value>classpath:/mapper/common-methods.xml</value>
                <value>classpath:/mapper/namespace-qa.xml</value>
            </list>
        </property>
        <property name="typeHandlers">
            <list>
                <bean class="com.mkhsueh.example.DateTimeTypeHandler" />
            </list>
        </property>
    </bean>
    </beans>  
</beans>
```

Supply the session factory bean to the DAO layer:

```xml spring-dao.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	<!-- imports -->
	<import resource="classpath:/spring/spring-datasource-sessionFactory.xml"/>
    

    <bean id="purchaseMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
        <property name="mapperInterface" value="com.mkhsueh.example.mapper.CustomerMapper"/>
        <property name="sqlSessionFactory" ref="sqlSessionFactory"/>
    </bean>

    <bean id="purchaseMapperCommon" class="org.mybatis.spring.mapper.MapperFactoryBean">
        <property name="mapperInterface" value="com.mkhsueh.example.common.mapper.CustomerMapper"/>
        <property name="sqlSessionFactory" ref="sqlSessionFactory"/>
    </bean>
 
</beans>
```

Now when the application is invoked with VM args {% codeblock %} -Dspring.profiles.active="prod" {% endcodeblock %} the session factory that loads namespace-prod.xml gets constructed and injected into the DAO object. 




  