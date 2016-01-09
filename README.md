Creating additional containers for ActiveMQ and Services in your Fabric (part2)
================================================================================
Author: Matt Robson

Technologies: Fuse, Fabric8, ActiveMQ, fabric8-maven-plugin, Profiles

Product: Fuse 6.2.1, ActiveMQ 6.2.1

Breakdown                                                                                                                     
---------                                                                                                                     
This is a command and profile based example to demonstrate how to build on the Fabric ensemble you created in part 1, 'fuse-fabric8-getting-started'.  We will explore how to create additional containers using fabric SSH and then provision them as AMQ Brokers and to host some services.

For more information see:

* <https://access.redhat.com/site/documentation/JBoss_Fuse/> for more information about Red Hat JBoss Fuse
* <http://www.jboss.org/products/fuse/overview/> for more information about the upstream community
* <http://fabric8.io/> for more information about fabric8

System Requirements
-------------------
Before building out your Fabric, you will need:
* Java 1.7 or 1.8
* JBoss Fuse 6.2.1
* JBoss ActiveMQ 6.2.1

Prerequisites
-------------
* A working Fabric as outlined in part 1, 'fuse-fabric8-getting-started' <https://github.com/mrobson/fuse-fabric8-getting-started>

The Build Out                                                                                                             
-------------

1) The first thing we're going to do is access our root container and open up a client session from the bin/ directory:

	./client

2) Now we will create a remote container using the ssh method.  This container will server as an AMQ Broker.

	JBossFuse:admin@root> fabric:container-create-ssh --host fusefabric3.lab.com --user root --password password --new-user admin --new-user-password admin --resolver manualip --manual-ip fusefabric3.lab.com amq-broker1
	The following containers have been created successfully:
	Container: amq-broker1.

On the root node in the karaf.log, you will see the operation complete successfully.

	2016-01-08 18:14:32,396 | INFO  | iner amq-broker1 | FabricServiceImpl                | 138 - io.fabric8.fabric-core - 1.2.0.redhat-621084 | The container amq-broker1 has been successfully created

On the remote host, you will see a new directory created called containers/ in the home directory of the user you specified above.  In my case, I am simply using root.  This is a brand new Fuse install for your new container.

	/root/containers/amq-broker1/fabric8-karaf-1.2.0.redhat-621084/

3) Create another 2 ssh containers on a different server to act as a second and third AMQ node.

	JBossFuse:admin@root> fabric:container-create-ssh --host fusefabric4.lab.com --user root --password password --new-user admin --new-user-password admin --resolver manualip --manual-ip fusefabric4.lab.com amq-broker2
	The following containers have been created successfully:
	Container: amq-broker2.

	JBossFuse:admin@root> fabric:container-create-ssh --host fusefabric5.lab.com --user root --password password --new-user admin --new-user-password admin --resolver manualip --manual-ip fusefabric5.lab.com amq-broker3
	The following containers have been created successfully:
	Container: amq-broker3.

4) Finally, we can create a third node which we can use to host some services.

	JBossFuse:admin@root> fabric:container-create-ssh --host fusefabric6.lab.com --user root --password password --new-user admin --new-user-password admin --resolver manualip --manual-ip fusefabric6.lab.com fuse-services1
	The following containers have been created successfully:
	Container: fuse-services1.

5) Running a container list will now display your 3 new nodes, connected to the Fabric, with the default profile.

	JBossFuse:admin@root> container-list 
	[id]                           [version] [connected] [profiles]                                         [provision status]
	amq-broker1                    1.0       true        default                                            success
	amq-broker2                    1.0       true        default                                            success
	amq-broker3                    1.0       true        default, mq-broker-redHatBrokerNetwork.redhat-broker3 success
	fuse-services1                 1.0       true        default                                            success
	root*                          1.0       true        fabric, jboss-fuse-full, fabric-ensemble-0001-1    success
	root2                          1.0       true        fabric, fabric-ensemble-0001-2                     success
	root3                          1.0       true        fabric, fabric-ensemble-0001-3                     success

Now that our containers are created, we can move on to adding some functionality!  We will start the by creating 3 ActiveMQ brokers configured as a Network of Brokers.  We will also generate the associated client configuration.  To do this, we have packaged the process into repeatable profiles.  There is a base profile, applicable to any environment, along with 3 detailed profiles specific to the current environment.  This allows for convenient usage in Fabric across environments.  The detailed profile inherits from the base profile to reduce the amount of required configuration.

6) clone the project

        git clone https://github.com/mrobson/fuse-fabric8-ssh-containers.git

7) change to project directory

        cd fuse-fabric8-ssh-containers

8) build and deploy the projects

	mvn fabric8:deploy -Dfabric8.jolokiaUrl=http://fusefabric1.lab.com:8181/jolokia

This will deploy all 4 profiles.

	fabric-broker-base-profile
	fabric-redhat-broker1-profile
	fabric-redhat-broker2-profile
	fabric-redhat-broker3-profile
	fabric-redhat-broker-client-profile

The first, 'fabric-broker-base-profile' is the generic base profile that provides the broker.xml and default configurations.  The second, third and fourth profiles, 'fabric-redhat-broker1-profile', 'fabric-redhat-broker2-profile' and 'fabric-redhat-broker3-profile' inherit from the 'fabric-broker-base-profile' and reconfigure the necessary defaults specific to our lab for broker1, broker2 and broker3.  The fourth profile, 'fabric-redhat-broker-client-profile' is the client profile used by the services to connect to the brokers over the fabric8 amq: protocol.

The profiles are built and deployed using the fabric8 maven plugin.  Check out how in the POM and see the broker.xml and PID properties file being deployed in src/main/fabric8/ directory.

	JBossFuse:admin@root> profile-display mq-redhat
	Profile id: mq-redhat
	Version   : 1.0
	Attributes: 
		abstract: false
		parents: karaf
	Containers: 

	Container settings
	----------------------------
	Features : 
		mq-fabric
		mq-fabric-http-discovery

	Configuration details
	----------------------------

	PID: io.fabric8.mq.fabric.template
	  broker.username admin
	  standby.pool default
	  broker.password password
	  broker.nob.transport.uri tcp://hosta:61617
	  broker.host.name fusefabric1
	  data ${karaf.base}/data/${broker-name}
	  broker.client.transport.uri tcp://hosta:61616
	  broker.client.transport.name redhat-broker-client
	  kind StandAlone
	  connectors ${broker.client.transport.name}
	  broker.nob.transport.name redhat-broker-nob
	  config profile:redhat-broker.xml
	  broker.network.name.2 redhat-broker2
	  broker.name ${broker-name}
	  group default
	  broker.network.name.1 redhat-broker1
	  broker.network.uri.1 static:(tcp://hostb:61617)
	  broker.network.uri.2 static:(tcp://hostb:61617)

	Other resources
	----------------------------
	Resource: redhat-broker.xml
	Resource: README.md

	JBossFuse:admin@root> profile-display mq-broker-redHatBrokerNetwork.redhat-broker1
	Profile id: mq-broker-redHatBrokerNetwork.redhat-broker1
	Version   : 1.0
	Attributes: 
		abstract: false
		parents: mq-redhat
	Containers: 

	Container settings
	----------------------------

	Configuration details
	----------------------------
	PID: io.fabric8.mq.fabric.server-redhat-broker1
	  broker.username admin
	  standby.pool default
	  broker.data.dir ${data}/kahadb
	  broker.password admin
	  broker.nob.transport.uri ssl://fusefabric3.lab.com:61617?keepAlive=true
	  broker-name redhat-broker1
	  broker.host.name fusefabric3.lab.com
	  data ${karaf.base}/data/redhat-broker1
	  broker.client.transport.uri ssl://fusefabric3.lab.com:61616
	  broker.client.transport.name redhat-broker-client
	  kind StandAlone
	  connectors ${broker.client.transport.name}
	  broker.nob.transport.name redhat-broker-nob
	  config profile:redhat-broker.xml
	  broker.network.name.2 redhat-broker3
	  broker.name ${broker-name}
	  group redHatBrokerNetwork
	  broker.network.name.1 redhat-broker2
	  broker.network.uri.1 static:(ssl://fusefabric4.lab.com:61617)
	  broker.network.uri.2 static:(ssl://fusefabric5.lab.com:61617)

	JBossFuse:admin@root> profile-display mq-client-redHatBrokerNetwork
	Profile id: mq-client-redHatBrokerNetwork
	Version   : 1.0
	Attributes: 
		abstract: false
		parents: karaf
	Containers: 

	Container settings
	----------------------------

	Configuration details
	----------------------------
	PID: io.fabric8.mq.fabric.cf
	  brokerUrl discovery://(fabric:redHatBrokerNetwork)
	  group redHatBrokerNetwork
	  password admin
	  user admin

9) Now that the profiles have been deployed into the fabric, you can assign them to the appropriate containers.

	JBossFuse:admin@root> container-add-profile amq-broker1 mq-broker-redHatBrokerNetwork.redhat-broker1
	JBossFuse:admin@root> container-add-profile amq-broker2 mq-broker-redHatBrokerNetwork.redhat-broker2
	JBossFuse:admin@root> container-add-profile amq-broker3 mq-broker-redHatBrokerNetwork.redhat-broker3

10) You will see a number of connection errors as the default need to be updated
	
	Caused by: java.io.IOException: Failed to bind to server socket: tcp://fusefabric5.lab.com:61616 due to: java.net.BindException: Cannot assign requested address
	
11) You can update the defaults by editing the properties file in HawtIO or by executing profile-edit commands from the Karaf Shell

	profile-edit --pid io.fabric8.mq.fabric.server-redhat-broker1/broker.host.name=tcp://fusefabric4.lab.com:61616 mq-broker-mattsBrokerNetwork.redhat-broker1
	profile-edit --pid io.fabric8.mq.fabric.server-redhat-broker1/broker.client.transport.uri=tcp://fusefabric4.lab.com:61616 mq-broker-mattsBrokerNetwork.redhat-broker1
	profile-edit --pid io.fabric8.mq.fabric.server-redhat-broker1/broker.nob.transport.uri=tcp://fusefabric4.lab.com:61617 mq-broker-mattsBrokerNetwork.redhat-broker1
	profile-edit --pid io.fabric8.mq.fabric.server-redhat-broker1/broker.network.uri.1=static:(tcp://fusefabric5.lab.com:61617) mq-broker-mattsBrokerNetwork.redhat-broker1
	profile-edit --pid io.fabric8.mq.fabric.server-redhat-broker1/broker.network.uri.2=static:(tcp://fusefabric5.lab.com:61617) mq-broker-mattsBrokerNetwork.redhat-broker1
	
12) You will see redhat-broker1 start and then start failing to connect to the second server in the NoB.
	
	2016-01-08 18:33:07,383 | INFO  | df-d0ba065086ac) | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Found zookeeper service for broker redhat-broker1.
	2016-01-08 18:33:07,383 | INFO  | df-d0ba065086ac) | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Broker redhat-broker1 is waiting to become the master
	2016-01-08 18:33:07,384 | INFO  | df-d0ba065086ac) | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Broker redhat-broker1 added to pool default.
	2016-01-08 18:33:07,411 | INFO  | ZooKeeperGroup-0 | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Broker redhat-broker1 is slave
	2016-01-08 18:33:07,413 | INFO  | ZooKeeperGroup-0 | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Broker redhat-broker1 is slave
	2016-01-08 18:33:07,422 | INFO  | ZooKeeperGroup-0 | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Broker redhat-broker1 is now the master, starting the broker.
	2016-01-08 18:33:07,422 | INFO  | ZooKeeperGroup-0 | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | Broker redhat-broker1 is being started.
	2016-01-08 18:33:07,453 | INFO  | AMQ-3-thread-1   | ActiveMQServiceFactory           | 133 - io.fabric8.mq.mq-fabric - 1.2.0.redhat-621084 | booting up a broker from: profile:redhat-broker.xml
	2016-01-08 18:33:07,716 | INFO  | AMQ-3-thread-1   | DiscoveryNetworkConnector        | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Establishing network connection from vm://redhat-broker1?async=false&network=true to tcp://fusefabric5.lab.com:61617
	2016-01-08 18:33:07,790 | INFO  | AMQ-3-thread-1   | TransportConnector               | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Connector vm://redhat-broker1 started
	2016-01-08 18:33:07,826 | INFO  | -broker1] Task-2 | DemandForwardingBridgeSupport    | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | redhat-broker1 Shutting down redhat-broker2
	2016-01-08 18:33:07,832 | INFO  | -broker1] Task-2 | DemandForwardingBridgeSupport    | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | redhat-broker1 bridge to Unknown stopped
	2016-01-08 18:33:07,834 | INFO  | AMQ-3-thread-1   | TransportConnector               | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Connector vm://redhat-broker1 stopped
	2016-01-08 18:33:07,835 | WARN  | AMQ-3-thread-1   | DiscoveryNetworkConnector        | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Could not start network bridge between: vm://redhat-broker1?async=false&network=true and: tcp://fusefabric5.lab.com:61617 due to: Connection refused

Once broker2 and broker 3 are also up and running, you will see the NoB form.

	2016-01-08 18:40:39,140 | INFO  | ActiveMQ Task-15 | DiscoveryNetworkConnector        | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Establishing network connection from vm://redhat-broker1?async=false&network=true to tcp://fusefabric5.lab.com:61617
	2016-01-08 18:40:39,218 | INFO  | edhat-broker1#76 | DemandForwardingBridgeSupport    | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Network connection between vm://redhat-broker1#76 and tcp://fusefabric5.lab.com/10.10.183.207:61617@36905 (redhat-broker2) has been established.
	2016-01-08 18:41:09,167 | INFO  | ActiveMQ Task-16 | DiscoveryNetworkConnector        | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Establishing network connection from vm://redhat-broker1?async=false&network=true to tcp://fusefabric6.lab.com:61617
	2016-01-08 18:41:09,219 | INFO  | edhat-broker1#80 | DemandForwardingBridgeSupport    | 137 - org.apache.activemq.activemq-osgi - 5.11.0.redhat-621084 | Network connection between vm://redhat-broker1#80 and tcp://fusefabric6.lab.com/10.10.183.201:61617@60885 (redhat-broker3) has been established.	

13) The last piece is to provision the client profile to our services node where we will later deploy some example code to test everything out.

	JBossFuse:admin@root> container-add-profile fuse-services1 mq-client-redHatBrokerNetwork

	2016-01-08 18:58:20,613 | INFO  | cache-1-thread-1 | ZkDataStoreImpl                  | 68 - io.fabric8.fabric-core - 1.2.0.redhat-621084 | Event CHILD_UPDATED detected on /fabric/configs/versions/1.0/containers/fuse-services1 with data default mq-client-redHatBrokerNetwork. Sending notification.

11) Verify everyone was successfully provisioned

	JBossFuse:admin@root> container-list 
	[id]                           [version] [connected] [profiles]                                            [provision status]
	amq-broker1                    1.0       true        default, mq-broker-redHatBrokerNetwork.redhat-broker1 success
	amq-broker2                    1.0       true        default, mq-broker-redHatBrokerNetwork.redhat-broker2 success
	amq-broker3                    1.0       true        default, mq-broker-redHatBrokerNetwork.redhat-broker3 success
	fuse-services1                 1.0       true        default, mq-client-redHatBrokerNetwork                success
	root*                          1.0       true        fabric, fabric-ensemble-0001-1                        success
	root2                          1.0       true        fabric, fabric-ensemble-0001-2                        success
	root3                          1.0       true        fabric, fabric-ensemble-0001-3                        success

Done!

To add some services to your container that utilize your brokers, continue on to part 3 <https://github.com/mrobson/fuse-fabric8-amq>.
