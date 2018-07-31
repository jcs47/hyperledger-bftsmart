# BFT ordering service for Hyperledger Fabric v1.1

This is a Byzantine fault-tolerant (BFT) ordering service for Hyperledger Fabric (HLF) v1.1. This BFT ordering service is a wrapper around [BFT-SMaRt](https://github.com/bft-smart/library), a Java open source BFT library maintained by the LaSIGE research unit at the University of Lisbon. 

## Overview

This ordering service has a similar architecture to the Kafka-based ordering service already provided by HLF. It is comprised by a set of `3f+1` ordering nodes and an arbitrary number of frontends, as depicted in the figure bellow. 

![Architecture](https://github.com/jcs47/hyperledger-bftsmart/blob/master/bft-os.png?raw=true)

The ordering nodes are equivalent to the Kafka-cluster with a Zookeeper ensemble, in the sense that they also execute a distributed consensus protocol responsible for establishing a total order on transactions. Furthermore, the frontends are equivalent to the ordering service nodes (OSNs) used by the Kafka-based ordering service, in the sense that they also relay the transactions issued by clients into the the consensus protocol. However, some key differences between this service and the Kafka-based service are:

1) The ordering nodes execute a BFT consensus protocol, which means malicious nodes are unable to disrupt the service (as long as they do not exceed `f` nodes out of a total of `3f+1`);
2) Whereas Kafka's OSNs receive a stream of ordered transactions, this service's frontends receive a stream of pre-generated blocks containing ECDSA signatures from `2f+1` ordering nodes;
3) Whereas Kafka's OSNs are logically comprised by a single Go process, this service's frontends are comprised by two processes (a Java and a Go component).

For more information regarding this project, check out the technical report available [here](http://arxiv.org/abs/1709.06921)

## Pre-requisites

This ordering service was developed and tested under Ubuntu 16.04.2 LTS and HLF v1.1. The docker images provided in Docker Hub are compatible with Docker 1.13.1 and above.

Because this ordering service needs to be integrated into the HLF codebase, it requires the HLF fork repository available [here](https://github.com/jcs47/fabric) instead of the official repository. This means that all dependencies associated with the offical HLF codebase are still required for this fork. The pre-requisites associated to the HLF codebase can be found [here](http://hyperledger-fabric.readthedocs.io/en/release/prereqs.html).

Besides the aforementioned dependecies, this service also uses the JUDS library to provide UNIX sockets for communication between the Java and Go components. The codebase already includes the associated jar, but because it uses JNI to access native UNIX sockets interfaces, it is still necessary to download the source code from [here](https://github.com/mcfunley/juds) and go through the installation steps described in the README.

## Quick start

You can quickly launch a local Fabric network, comprised of 4 ordering nodes, a single frontend, and one peer by following the steps described bellow.

1. Make sure you are not executing any container. This is because the images are already configured for containers allocated to the addresses 172.17.0.2-172.17.0.8. After stopping any container you may have running, download the images and create their respective containers in the following order (each one from a different terminal):

```
docker run -i -t -P bftsmart/fabric-orderingnode 0
docker run -i -t -P bftsmart/fabric-orderingnode 1
docker run -i -t -P bftsmart/fabric-orderingnode 2
docker run -i -t -P bftsmart/fabric-orderingnode 3
docker run -i -t -P bftsmart/fabric-frontend
docker run -i -t -P -v /var/run/:/var/run/ hyperledger/fabric-peer:x86_64-1.1.1
docker run -i -t -P bftsmart/fabric-tools
```
You have now the whole network booted up, using the `SampleOrg` organization provided in the `sampleconfig` directory of the Fabric codebase for both clients, peers, and the ordering service.

As you may have noticed, you can use the ordering service with the official peer image provided by the Hyperledger project. You can also use it with the official CLI image, but the one provided by us is already configured for this demontration. You will also need to use the `configtxgen` tool provided with the image if you decide to setup another network different than the one configured in the images.

2. Switch to the terminal where you launched fabric-tools. You should have access to the container's command line. The rest of the commands should be issued from within it. Generate the transactions to create a new channel named "channel47" and to update its anchor peers as follows:

```
configtxgen -profile SampleSingleMSPChannel -outputCreateChannelTx channel.tx -channelID channel47
configtxgen -profile SampleSingleMSPChannel -outputAnchorPeersUpdate anchor.tx -channelID channel47 -asOrg SampleOrg
```
Notice we are not generating the genesis block for the system channel because the images already come with one generated. The name of the system channel is "bftchannel".

3. If you were not executing any container before following this instructions, the frontend should have been assigned the address "172.17.0.6". Send the transactions to the orderng service by providing the frontend's entrypoint as follows:

```
peer channel create -o 172.17.0.6:7050 -c channel47 -f channel.tx 
peer channel update -o 172.17.0.6:7050 -c channel47 -f anchor.tx
```

You should now have a file named "channel47.block" in your current directory of the container. You can use it to make the peer join the channel as follows:

```
peer channel join -b channel47.block
```

4. Install and instantiate the example chaincode included in the container as follows:

```
peer chaincode install -n example02 -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
peer chaincode instantiate -o 172.17.0.6:7050 -C channel47 -n example02 -v 1.0 -c '{"Args":["init","a","100","b","200"]}'
```

5. You can now perform queries and invocations to the chaincode:

```
peer chaincode query -C channel47 -n example02 -v 1.0 -c '{"Args":["query","a"]}'

2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
Query Result: 100

peer chaincode invoke -C channel47 -n example02 -v 1.0 -c '{"Args":["invoke","a","b","10"]}'

2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] InitCmdFactory -> INFO 001 Get chain(channel47) orderer endpoint: 172.17.0.6:7050
2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default escc
2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default vscc
2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 004 Chaincode invoke successful. result: status:200 
2018-xx-xx xx:xx:xx.xx UTC [main] main -> INFO 005 Exiting.....

peer chaincode query -C channel47 -n example02 -v 1.0 -c '{"Args":["query","a"]}'

2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-xx-xx xx:xx:xx.xx UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
Query Result: 90
```

## Distributed deployment

You can configure the ordering nodes, frontends, peers and clients to operate in a true distributed setting with the scripts contained in the `./docker_images/` folder. The idea is to execute the `prepare_<TYPE>_defaults.sh` scripts to obtain the default configuration files for each type of principal (where `<TYPE>` is either `orderingnode`,`frontend`, `peer`, or `cli`). This will create a new folder named `./<TYPE>_material` where you can edit the configuration files before copying them into containers. Once the configuration files are ready, they can be copied into their respective containers using the `create_<TYPE>_container.sh` scripts. Bear in mind that not all scripts need to be invoked at all hosts (for instance, `prepare_peer_defaults.sh` and `create_peer_container.sh` only need to be executed at hosts executing peers).

If you wish to re-create the "`SampleOrg`" network from the previous section in a distributed scenario:

1. Login to each host machine that will comprise the network and download this repository using `git clone` or any other method of your preference. Execute the respective `prepare_<TYPE>_defaults.sh` scripts from within the `./docker_images` folder

2. In the machines that should run ordering nodes and the frontend, edit the `hosts.config` file with the IDs, IP addresses, and ports of each host running an ordering node. This file must be the same across all ordering nodes, and should look something like this:

```
#server id, address and port (the ids from 0 to n-1 are the service replicas) 
0 192.168.1.1 11000
1 192.168.1.2 11000
2 192.168.1.3 11000
3 192.168.1.4 11000
7001 192.168.1.100 11100
```

The last line in the file represents the ID for a trust BFT-SMaRt client that will be discussed in a later section.

3. In the machine running the peer, make sure that docker can be correctly accessed from inside the container, by checking the `vm->endpoint` parameter from `core.yaml`. If the peer is supposed to access docker using UNIX sockets, make sure the host machine is creating the socket file at `/var/run/docker.sock` folder and that the value of the parameter is set to `unix:///var/run/docker.sock`. This is because it is in that folder that `create_peer_container.sh` will attempt to mount a volume containing the socket file.

4. In the machine running the client, make sure to set `peer->address` from `core.yaml` to the correct endpoint. In the case of this "`SampleOrg`" network, the entrypoint corresponds to the IP address of the machine executing the peer at port `7051`. If the machine's IP address is `192.168.1.6`, then the value of the parameter should be `192.168.1.6:7051`. Alternatively, you can leave `core.yaml` unchanged and pass the endpoint later, when invoking `create_cli_container.sh`.

5. Once you are done configuring the files, execute the `create_<TYPE>_container.sh` script at each host. You will need to supply a name for the container as a parameter. Each script will output the following:

```
Container ID for <container name> is <container id>
Launch the <TYPE> by typing 'docker start -a <container name>'
Stop the tools by typing 'docker stop <container name>'
```

6. At this point you can launch the containers with the commands indicated by the output of the scripts. However, before you are able to create channels, there is one extra step that needs to be performed in the case of "`SampleOrg`". As previously said, the images already have the genesis block for the system channel created. Since the configuration contained within that block was prepared for a local enviroment, it contains an incorrect address for the frontend at this point (it is set to `172.17.0.6:7050`, the expected docker address for the local deployment). Nonetheless, the CLI image we provide contains a script to update the list of frontends. Assuming the entrypoint for the frontend is `192.168.1.5:7050`, launch the container and execute the following commands:

```
peer channel fetch config genesisblock -c bftchannel -o 192.168.1.5:7050
echo "[\"192.168.1.5:7050\"]" > addresses.json
update_frontend_entrypoint.sh genesisblock addresses.json update.tx
peer channel signconfigtx -f update.tx 
peer channel update -o 192.168.1.5:7050 -c bftchannel -f update.tx
```

From this point on, you should be able to create new channels and invoke the chaincode as in the previous section.

## Compiling

If you wish to compile this code, make sure to switch to the 'release-1.1' branch, both for this repository and for the aforementioned HLF fork. You can compile that fork the same way as the official repository. However, you must make sure to compile it outside of Vagrant, by executing:

```
cd $GOPATH/src/github.com/hyperledger/fabric/
sudo ./devenv/setupUbuntuOnPPC64le.sh
```
Notice that even though we are working with a fork instead of the original HLF repository, the code still needs to be stored in `$GOPATH/src/github.com/hyperledger/fabric/`. Following this, you can compile HLF as follows:

```
make dist-clean peer orderer configtxgen
```

Make also sure to set the `$FABRIC_CFG_PATH` environment variable to the absolute path of the `./sampleconfig` directory of the fork. You should be able to do this by typing `export FABRIC_CFG_PATH=$GOPATH/src/github.com/hyperledger/fabric/sampleconfig/` in the command line. If you are using multiple terminals, you might need to type this command in each one of them.

To compile the Java code provided by this repository, you can simply type `ant` in its main folder. To compile the images available at Docker Hub, enter the `docker_images` sub-directory and run the `create_docker_images.sh` script (you will need to provide the name for the system channel as an argument).

## Executing the compiled code without docker

### Launching 4 ordering nodes and a single frontend

The first step is to generate the genesis block for the ordering service. Generate the block as follows:

```
cd $GOPATH/src/github.com/hyperledger/fabric/
./build/bin/configtxgen -profile SampleSingleMSPBFTsmart -channelID <system channel ID> -outputBlock <path to genesis file>
```

The `<path to genesis file>` argument should match both the absolute path in the `GenesisFile` parameter in the `General` section in of the `./fabric/sampleconfig/orderer.yaml` configuration file and the `GENESIS` parameter in the `./hyperledger-bftmart/config/node.config`. Keep in mind that in the case of this ordering service, the `GenesisMethod` parameter must always be set to `file`. Otherwise the frontends will generate genesis blocks distinct from the one loaded by the ordering node, thus leading to inconsistencies.

After creating the genesis block, edit the  `./hyperledger-bftmart/config/hosts.config` file with the loopback address for your machine. It should look something like this:

```
#server id, address and port (the ids from 0 to n-1 are the service replicas) 
0 127.0.0.1 11000
1 127.0.0.1 11010
2 127.0.0.1 11020
3 127.0.0.1 11030
7001 127.0.0.1 11100
```

Next, edit the `./hyperledger-bftmart/config/node.config` file so that the `CERTIFICATE` parameter is set to the absolute path of the `./fabric/sampleconfig/msp/signcerts/peer.pem` file and that the `PRIVKEY` parameter is set to the absolute path of the `./fabric/sampleconfig/msp/keystore/key.pem` file. Following this, enter the main directory for this repository and execute the `startReplica.sh` script in 4 different terminals as follows:

```
./startReplica.sh 0
./startReplica.sh 1
./startReplica.sh 2
./startReplica.sh 3
```

Once all nodes have outputed the message `-- Ready to process operations`, you can launch the Java component of the frontend as follows:

```
./startFrontend.sh 1000 10 9999
```

The first argument is the ID of the frontend, and it should match one of the IDs specified in the `RECEIVERS` parameter in the `./hyperledger-bftmart/config/node.config` file. The second argument is the number of UNIX connections available in the pool between the Go and Java components, and it should match the `ConnectionPoolSize`parameter from the `BFTsmart` section in the `./fabric/sampleconfig/orderer.yaml` file. The third parameter is the TCP port from which the Java component delivers blocks to the Go component, and should match the `RecvPort` parameter in the previous section/file.

You can now launch the Go component as follows:

```
./build/bin/orderer start
```

### Running the example chaincode

To execute an example chaincode using this ordering service, generate the rest of the HLF artifacts as follows:

```
cd $GOPATH/src/github.com/hyperledger/fabric/
./build/bin/configtxgen -profile SampleSingleMSPChannel -outputCreateChannelTx <path to channel creation tx> -channelID <channel ID>
./build/bin/configtxgen -profile SampleSingleMSPChannel -outputAnchorPeersUpdate <path to anchor peer update tx> -channelID <channel ID> -asOrg SampleOrg
```

Note that the channel ID that you use for generating the channel creation transaction must be different that the one used to generate the genesis block. When creating the genesis block, you specify the ID for the *system channel*, whereas when generating the channel creating transaction, you are going to request a new *normal channel* to be created.

With the ordering service bootstrapped, you can now launch an endorsing peer. To do so, make sure that the `fileSystemPath` parameter of the `peer` section in the `./sampleconfig/core.yaml` configuration file is set to a directory where you have write previledges. Following this, start the endorsing peer process by executing:

```
./build/bin/peer node start

```

Use a client to join a channel and install/execute chaincode as follows:

```
./build/bin/peer channel create -o 127.0.0.1:7050 -c <channel ID> -f <path to channel creation tx>
./build/bin/peer channel join -b ./<channel ID>.block
./build/bin/peer channel update -o 127.0.0.1:7050 -c <channel ID> -f <path to anchor peer update tx>
./build/bin/peer chaincode install -n <chaincode ID> -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
./build/bin/peer chaincode instantiate -o 127.0.0.1:7050 -C <channel ID> -n <chaincode ID> -v 1.0 -c '{"Args":["init","a","100","b","200"]}'
./build/bin/peer chaincode query -C <channel ID> -n <chaincode ID> -v 1.0 -c '{"Args":["query","a"]}'
./build/bin/peer chaincode invoke -C <channel ID> -n <chaincode ID> -v 1.0 -c '{"Args":["invoke","a","b","10"]}'
./build/bin/peer chaincode query -C <channel ID> -n <chaincode ID> -v 1.0 -c '{"Args":["query","a"]}'
./build/bin/peer chaincode invoke -C <channel ID> -n <chaincode ID> -v 1.0 -c '{"Args":["invoke","a","b","-10"]}'
./build/bin/peer chaincode query -C <channel ID> -n <chaincode ID> -v 1.0 -c '{"Args":["query","a"]}'
```

### Running with the sample clients

To submit a heavy workload of representative transactions using the sample clients available in `./fabric/sample_clients`, execute the commands bellow in the following order:

Execute `go build`  at directories `./fabric/orderer/sample_clients/deliver_stdout`, `./fabric/orderer/sample_clients/broadcast_msg/` and `./fabric/orderer/sample_clients/broadcast_config/` to compile the sample clients

Launch a client to receive the generated blocks as follows:

```
cd $GOPATH/src/github.com/hyperledger/fabric/
./orderer/sample_clients/deliver_stdout/deliver_stdout --quiet --channelID <system channel ID>
```
  
Launch a client to submit transactions to the service as follows:

```
./orderer/sample_clients/broadcast_msg/broadcast_msg --channelID <system channel ID> --size <size of each transaction> --messages <number of transactions to send>
```
  
You can also create a new channel as follows:

```
./orderer/sample_clients/broadcast_config/broadcast_config --cmd newChain --chainID <channel ID>
  ```

### Running the code in a distributed setting

The HLF codebase provided by the fork can be configured and deployed in a distributed setting in the same way as the original. In order to make sure the distributed deployment still uses this ordering service, the genesis block must be generated from a profile that sets the ordering service type to `bftsmart`.

In order to deploy the ordering nodes across multiple hosts, edit the `hyperledger-bftmart/config/host.config` file with the ip address of the hosts running each ordering node. If you want to set up more than 4 nodes, you must also edit the `hyperledger-bftmart/config/system.config` file (you need to fiddle with the parameters `system.servers.num`, `system.servers.f` and `system.initial.view`). These files must be the same across all ordering nodes, as well as the frontend.

Any time you need to bootstrap the system from scratch, make sure you delete the `hyperledger-bftmart/config/currentView` file in each host. Otherwise, the ordering nodes will fetch information about the group of nodes from that file instead of the configuration files described previously. Make also sure that you launch each node in numerical order (node 0, node 1, node 2, etc).

Finally, keep in mind that the Go component needs to be in the same host as the Java component. Nonetheless, you only need to install the JUDS dependencies in the hosts that runs a frontend.

## State transfer and reconfiguration

Each ordering node can be re-started after a crash, as long as the total number of simultaneously crashed nodes do not exceed `f`. Furthermore, it is also possible to change the set of ordering nodes on-the-fly via BFT-SMaRt's reconfiguration protocol. In order to add a node to the group, start it as you would any other node with the `startReplica.sh` script. To make that node join the existing group, use the `reconfigure.sh` script as follows:

```
./reconfigure.sh <node id> <ip address> <port>
```

In order to remove a node from the group, use the same script specifying only the node id. Bear in mind that when doing this in a distributed setting, it is necessary to copy the ``./hyperledger-bftsmart/config/currentView``file into the hosts that are about to join the group before anything else is done. This is because this file specifies the set of nodes that comprise the most up-to-date group. You must also make sure that the host from which the `reconfigure.sh` script is executed is also given this file.

## Limitations

The current codebase is compatible with, and as been tested for Fabic v1.1.1, but not yet with v1.2. Addtionally, and as described above, the current implementation of this ordering service supports state transfer and on-the-fly reconfiguration of the set of ordering nodes. However, this does not extends to the set of frontends yet.

