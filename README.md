Creating additional containers for Active-MQ and Services in your Fabric (part2)
================================================================================
Author: Matt Robson

Technologies: Fuse, Fabric8, ActiveMQ, fabric8-maven-plugin, Profiles

Product: Fuse 6.1, Active-MQ 6.1

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
* Java 1.7
* JBoss Fuse 6.1
* JBoss Active-MQ 6.1

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

	2015-06-08 14:31:03,908 | INFO  | iner amq-broker2 | FabricServiceImpl                | ric8.service.FabricServiceImpl$1  487 | 65 - io.fabric8.fabric-core - 1.0.0.redhat-431 | The container amq-broker1 has been successfully created

On the remote host, you will see a new directory created called containers/ in the home directory of the user you specified above.  In my case, I am simply using root.  This is a brand new Fuse install for your new container.

	/root/containers/amq-broker1/fabric8-karaf-1.0.0.redhat-431/

3) Create another ssh container on a different server to act as a second AMQ node.

	JBossFuse:admin@root> fabric:container-create-ssh --host fusefabric4.lab.com --user root --password password --new-user admin --new-user-password admin --resolver manualip --manual-ip fusefabric4.lab.com amq-broker2
	The following containers have been created successfully:
	Container: amq-broker2.

4) Finally, we can create a third node which we can use to host some services.

	JBossFuse:admin@root> fabric:container-create-ssh --host fusefabric5.lab.com --user root --password password --new-user admin --new-user-password admin --resolver manualip --manual-ip fusefabric5.lab.com fuse-services
	The following containers have been created successfully:
	Container: fuse-services.

5) Running a container list will now display your 3 new nodes, connected to the Fabric, with the default profile.

	JBossFuse:admin@root> container-list 
	[id]                           [version] [connected] [profiles]                                         [provision status]
	amq-broker1                    1.5       true        default                                            success
	amq-broker2                    1.5       true        default                                            success
	fuse-services                  1.5       true        default                                            success
	root*                          1.5       true        fabric, jboss-fuse-full, fabric-ensemble-0001-1    success
	root2                          1.5       true        fabric, fabric-ensemble-0001-2                     success
	root3                          1.5       true        fabric, fabric-ensemble-0001-3                     success
	root4                          1.5       true        fabric, fabric-ensemble-0001-4                     success
	root5                          1.5       true        fabric, fabric-ensemble-0001-5                     success

Now that our containers are created, we can move on to adding some functionality!  We will start the by creating 2 ActiveMQ brokers configured as a Network of Brokers.  We will also generate the associated client configuration.  To do this, we have packaged the process into repeatable profiles.  There is a base profile, applicable to any environment, along with 2 detailed profiles specific to the current environment.  This allows for convenient usage in Fabric across environments.  The detailed profile inherits from the base profile to reduce the amount of required configuration.

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
	fabric-redhat-broker-client-profile

The first, 'fabric-broker-base-profile' is the generic base profile that provides the broker.xml and default configurations.  The second and third profiles, 'fabric-redhat-broker1-profile' and 'fabric-redhat-broker2-profile' inherit from the 'fabric-broker-base-profile' and reconfigure the necessary defaults specific to our lab for broker1 and broker2.  The fourth profile, 'fabric-redhat-broker-client-profile' is the client profile used by the services to connect to the brokers.

The profiles are built and deployed using the fabric8 maven plugin.  Check out how in the POM and see the broker.xml and PID properties file being deployed in src/main/fabric8/ directory.

	JBossFuse:admin@root> profile-display mq-redhat 
	Profile id: mq-redhat
	Version   : 1.5
	Attributes: 
		parents: karaf
	Containers: 

	Container settings
	----------------------------
	Repositories : 
		mvn:org.apache.activemq/activemq-karaf/${version:activemq}/xml/features
	Features : 
		mq-fabric
		mq-fabric-http-discovery
	Configuration details
	----------------------------
	PID: org.fusesource.mq.fabric.template
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
	  broker.name ${broker-name}
	  group default
	  broker.network.name.1 redhat-broker
	  broker.network.uri.1 static:(tcp://hostb:61617)
	Other resources
	----------------------------
	Resource: redhat-broker.xml
	Resource: README.md

	JBossFuse:admin@root> profile-display mq-broker-redHatBrokerNetwork.redhat-broker1 
	Profile id: mq-broker-redHatBrokerNetwork.redhat-broker1
	Version   : 1.5
	Attributes: 
		parents: mq-redhat
	Containers: amq-broker1

	Container settings
	----------------------------
	Configuration details
	----------------------------
	PID: org.fusesource.mq.fabric.server-redhat-broker2
	  broker.username admin
	  standby.pool default
	  broker.password admin
	  broker.nob.transport.uri tcp://fusefabric3.lab.com:61617
	  broker-name redhat-broker1
	  broker.host.name fusefabric3.lab.com
	  data ${karaf.base}/data/redhat-broker1
	  broker.client.transport.uri tcp://fusefabric3.lab.com:61616
	  broker.client.transport.name redhat-broker-client
	  kind StandAlone
	  connectors ${broker.client.transport.name}
	  broker.nob.transport.name redhat-broker-nob
	  config profile:redhat-broker.xml
	  broker.name ${broker-name}
	  group redHatBrokerNetwork
	  broker.network.name.1 redhat-broker
	  broker.network.uri.1 static:(tcp://fusefabric4.lab.com:61617)

	JBossFuse:admin@root> profile-display mq-client-redHatBrokerNetwork 
	Profile id: mq-client-redHatBrokerNetwork
	Version   : 1.5
	Attributes: 
		parents: karaf
	Containers: fuse-services

	Container settings
	----------------------------
	Configuration details
	----------------------------
	PID: org.fusesource.mq.fabric.cf
	  brokerUrl discovery://(fabric:redHatBrokerNetwork)
	  group redHatBrokerNetwork
	  password admin
	  user admin

9) Now that the profiles have been deployed into the fabric, you can assign them to the appropriate containers.

	JBossFuse:admin@root> container-add-profile amq-broker1 mq-broker-redHatBrokerNetwork.redhat-broker1
	
You will see redhat-broker1 start and then start failing to connect to the second server in the NoB.

	2015-06-13 18:35:09,996 | INFO  | pool-47-thread-1 | ActiveMQServiceFactory           | q.fabric.ActiveMQServiceFactory$   65 | 150 - org.jboss.amq.mq-fabric - 6.1.1.redhat-431 | Broker redhat-broker1 has started.

	2015-06-13 18:35:10,960 | INFO  | ActiveMQ Task-1  | DiscoveryNetworkConnector        | etwork.DiscoveryNetworkConnector  120 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Establishing network connection from vm://redhat-broker1?async=false&network=true to tcp://fusefabric4.lab.com:61617
	2015-06-13 18:35:10,962 | INFO  | ActiveMQ Task-1  | TransportConnector               | tivemq.broker.TransportConnector  260 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Connector vm://redhat-broker1 started
	2015-06-13 18:35:10,971 | INFO  | -broker1] Task-5 | DemandForwardingBridgeSupport    | rk.DemandForwardingBridgeSupport 1008 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | redhat-broker1 Shutting down
	2015-06-13 18:35:10,974 | INFO  | -broker1] Task-5 | DemandForwardingBridgeSupport    | rk.DemandForwardingBridgeSupport  288 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | redhat-broker1 bridge to Unknown stopped
	2015-06-13 18:35:10,975 | INFO  | ActiveMQ Task-1  | TransportConnector               | tivemq.broker.TransportConnector  291 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Connector vm://redhat-broker1 stopped
	2015-06-13 18:35:10,977 | WARN  | ActiveMQ Task-1  | DiscoveryNetworkConnector        | etwork.DiscoveryNetworkConnector  156 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Could not start network bridge between: vm://redhat-broker1?async=false&network=true and: tcp://fusefabric4.lab.com:61617 due to: java.net.ConnectException: Connection refused

Assign the broker2 profile to amq-broker2 now.

	JBossFuse:admin@root> container-add-profile amq-broker2 mq-broker-redHatBrokerNetwork.redhat-broker2

You will see the profile provision and the second broker start.

	2015-06-13 18:37:05,881 | INFO  | pool-42-thread-1 | ActiveMQServiceFactory           | q.fabric.ActiveMQServiceFactory$   65 | 150 - org.jboss.amq.mq-fabric - 6.1.1.redhat-431 | Broker redhat-broker2 has started.
	2015-06-13 18:37:05,944 | INFO  | redhat-broker2#0 | DemandForwardingBridgeSupport    | rk.DemandForwardingBridgeSupport  475 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Network connection between vm://redhat-broker2#0 and tcp://fusefabric3.lab.com/10.10.183.205:61617@33596 (redhat-broker1) has been established.

On broker1, you will now see if connect successfully to broker2.

	2015-06-13 18:37:22,666 | INFO  | ActiveMQ Task-4  | DiscoveryNetworkConnector        | etwork.DiscoveryNetworkConnector  120 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Establishing network connection from vm://redhat-broker1?async=false&network=true to tcp://fusefabric4.lab.com:61617
	2015-06-13 18:37:22,667 | INFO  | ActiveMQ Task-4  | TransportConnector               | tivemq.broker.TransportConnector  260 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Connector vm://redhat-broker1 started
	2015-06-13 18:37:22,693 | INFO  | edhat-broker1#22 | DemandForwardingBridgeSupport    | rk.DemandForwardingBridgeSupport  475 | 131 - org.apache.activemq.activemq-osgi - 5.9.0.redhat-611431 | Network connection between vm://redhat-broker1#22 and tcp://fusefabric4.lab.com/10.10.183.206:61617@52059 (redhat-broker2) has been established.

10) The last piece is to provision the client profile to our services node where we will later deploy some example code to test everything out.

	JBossFuse:admin@root> container-add-profile fuse-services mq-client-redHatBrokerNetwork

11) Verify everyone was successfully provisioned

	JBossFuse:admin@root> container-list 
	[id]                           [version] [connected] [profiles]                                            [provision status]
	amq-broker1                    1.5       true        default, mq-broker-redHatBrokerNetwork.redhat-broker1 success
	amq-broker2                    1.5       true        default, mq-broker-redHatBrokerNetwork.redhat-broker2 success
	fuse-services                  1.5       true        default, mq-client-redHatBrokerNetwork                success
	root*                          1.5       true        fabric, fabric-ensemble-0001-1                        success
	root2                          1.5       true        fabric, fabric-ensemble-0001-2                        success
	root3                          1.5       true        fabric, fabric-ensemble-0001-3                        success
	root4                          1.5       true        fabric, fabric-ensemble-0001-4                        success
	root5                          1.5       true        fabric, fabric-ensemble-0001-5                        success

Done!

To add some services to your container that utilize your brokers, continue on to part 3 <https://github.com/mrobson/fuse-fabric8-amq>.
