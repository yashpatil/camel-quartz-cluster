# Camel Quartz cluster

Red Hat JBoss Fuse can be used to load balance and cluster using Fuse Fabric.
In the absence of Fabric, there are a few options to achieve this particular functionality. These options include:

* Camel Zookeeper - This component offers a RoutePolicy that can start/stop routes in master/slave fashion. The first route that gets the lock will be started where the remaining routes will be waiting to get the lock.
* Camel Quartz - This component has clustering support. If you are using quartz consumers, in clustered mode, you can have only one of the routes triggered at a time.
* Camel JGroups - Using JGroupsFilters, we can get master/slave capability

For background reference, see:

* [Master/Slave Failover for Camel Routes] (http://java.dzone.com/articles/masterslave-failover-camel)
* [Single instance of running route in the cluster] (http://camel.465427.n5.nabble.com/Single-instance-of-running-route-in-the-cluster-td5720846.html)

The aim of this project is to create a singleton route so that our Camel route runs only on one instance at any point in time.
Clustering is achieved by using database as a Persistence layer which can help with the failover. We are using PostgreSQL for this example.

```xml
<bean id="quartzDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="org.postgresql.Driver"/>
    <property name="url" value="jdbc:postgresql://localhost:5432/quartz"/>
    <property name="username" value="quartz"/>
    <property name="password" value="quartz"/>
</bean>
```

Camel Quartz component is initialized with a reference to the scheduler bean:

```xml
<bean id="quartz" class="org.apache.camel.component.quartz.QuartzComponent">
    <property name="scheduler" ref="scheduler"/>
</bean>
```

Note that this can also be done by providing a quartz properties file if so desired. A sample file quartzcluster.properties.notused (not fully configured) is added to this project under the META-INF/spring directory for quick reference purposes:

```xml
<bean id="quartz" class="org.apache.camel.component.quartz.QuartzComponent">
    <property name="propertiesFile" value="META-INF/spring/quartzcluster.properties"/>
</bean>
```

The key component from the scheduler definition is the `schedulerContextAsMap` section.
This hooks the camel context into the quartz configuration.
If this is not configured correctly, and if you reinstall your bundle in the container, you may see errors in your log like: ```org.quartz.JobExecutionException: No CamelContext could be found with name: 274-quartz-cluster``` where the bundle id shown would be the previous bundle-id of the deployed application bundle.

```xml
<property name="schedulerContextAsMap">
    <map>
        <entry key="CamelQuartzCamelContext" value-ref="quartzCluster"/>
    </map>
</property>
```

The `isClustered` flag turns on clustering, and the `PostgreSQLDelegate` sets the dialect:

```xml
<property name="quartzProperties">
    <props>
        <prop key="org.quartz.scheduler.instanceName">myscheduler</prop>
        <prop key="org.quartz.scheduler.instanceId">AUTO</prop>
        <prop key="org.quartz.scheduler.skipUpdateCheck">true</prop>
        <prop key="org.quartz.jobStore.driverDelegateClass">org.quartz.impl.jdbcjobstore.PostgreSQLDelegate</prop>
        <prop key="org.quartz.jobStore.isClustered">true</prop>
        <prop key="org.quartz.jobStore.clusterCheckinInterval">5000</prop>
    </props>
</property>
```

The camel context has been configured to fire off every 5 seconds and print a log message:

```xml
<camelContext xmlns="http://camel.apache.org/schema/spring" id="quartzCluster" managementNamePattern="#name#">
    <route id="clusteredRoute">
        <from uri="quartz://clustergroup/clusterTimerName?job.name=demoQuartzCluster&amp;cron=0/5+*+*+*+*+?"/>
        <log message="Fired quartz cluster trigger"/>
    </route>
</camelContext>
```


## Testing

### Install two versions of Red Hat JBoss Fuse (this test was done using version 6.0.0-redhat-60024)

If you plan to run them on the same host, the files to be modified to avoid port conflicts are:

${karaf.home}/etc/system.properties
------------------------------------
Instance 1:
org.osgi.service.http.port=8181
activemq.port=61616
activemq.jmx.url=service:jmx:rmi:///jndi/rmi://localhost:1099/karaf-${karaf.name}

Instance 2:
org.osgi.service.http.port=8281
activemq.port=61716
activemq.jmx.url=service:jmx:rmi:///jndi/rmi://localhost:1199/karaf-${karaf.name}

${karaf.home}/etc/jetty.xml
----------------------------
Instance 1:
<Property name="jetty.port" default="8181"/>
<Set name="confidentialPort">8443</Set>

Instance 2:
<Property name="jetty.port" default="8281"/>
<Set name="confidentialPort">8543</Set>

${karaf.home}/etc/org.ops4j.karaf.management.cfg
--------------------------------------------------
Instance 1:
rmiRegistryPort=1099
rmiServerPort=44444

Instance 2:
rmiRegistryPort=1199
rmiServerPort=44544

${karaf.home}/etc/org. apache.karaf.shell.cfg
-----------------------------------------------
Instance 1:
sshPort=8101

Instance 2:
sshPort=8201

### Start and install features
Run the following commands from the karaf command line:

`features:install quartz`

`features: install spring-jdbc`

### Install the PostgreSQL JDBC driver

The proper way to do it would be to install it in your local repository with the right felix-export and import options, and then install it in Fuse.
A quick workaround for testing purposes would be to do a `wrap` install as below:

`install -s wrap:file://<path_to_postgresql_jdbc.jar>`

### Compile and install this application bundle

`mvn clean install`

In the karaf console:

`install -s mvn:com.redhat.gps/quartzcluster/1.0.0-SNAPSHOT`

When you do this on both the servers, you will see that only one of them (the first one where the application bundle was installed) will start printing the log statements from our route:

`Fired quartz cluster trigger`

If you shut down this instance, the other instance should pick up the route, and you will start seeing the log statement being outputted in its logs.

