# docker-swarm-mysql-masterslave-failover

MySQL (minicluster) with a Master and a Slave, MySqlFailover and MySqlRouter.

This repository describes how to setup above components so MySQL (5.7) replication can be achieved.

## Notes

At this point we have not yet produced an "all-in-one" compose file. So this repository is made up of 3 compose files:

1. docker-compose-mysqlcluser-master.yml
2. docker-compose-mysqlcluser-slave.yml
3. docker-compose-mysqlcluser-connection.yml

As soon as we get to the point of proper health checks inside the images we intend to make 1 general compose file to be able to setup the cluster in 1 go.

At the moment we have not yet implemented the "secrets" offered by Docker. This means that the root password has to be put in the compose files.

Keep an eye on this repository since it is still under development and subject to change.

## Prerequisites

- Ubuntu (16.04 or higher) or RHEL host(s)
- Docker v1.13.1 (minimum)
    - Experimental Mode must be set to true (to be able to use "docker deploy" with compose v3 files)
    - Must run in Swarm Mode
    - Preferably at least 2 nodes (Manager + Worker) to run Master and Slave on
    - 1 overlay network to run MySQL Stack in  

## Used components

#### MySQL Stack

| Service | Purpose |
| ------ | ----- |
| [MySQL](https://hub.docker.com/r/voogd/mysql_repl/) | Main database server for the Master and Slave |
| [MySQLFailover](https://hub.docker.com/r/voogd/mysqlfailover/) | Service to failover to Slave in case Master dies  |
| [MySQLRouter](https://hub.docker.com/r/voogd/mysqlrouter/) | Router to balance client connections to active Master |


## Schema of the stacks
![stackflow](https://raw.githubusercontent.com/robinong79/docker-swarm-mysql-masterslave-failover/master/MySQL_Failover_Stack.png "Monitoring Logging Stack")

## Preparation

#### Directory Structure
The following directories need to be present on the host or on a central storage solution.
These directories will me used for connecting volumes to to services/containers.

| Component | Directory | Remarks |
| ----- | ----- | ----- | 
| mysql01 (Master) | /var/dockerdata/mysqlcluster/mysql01/conf.d | Directory for storing configuration files|
|  | /var/dockerdata/mysqlcluster/mysql01/initdb.d | Directory with sql initialisation files for fresh DB|
|  | /var/dockerdata/mysqlcluster/mysql01/lib | Data storage for MySQL instance|
|  | /var/dockerdata/mysqlcluster/mysql01/log | Directory where bin-log files for replication will be stored |
| mysql02 (Slave) | /var/dockerdata/mysqlcluster/mysql01/conf.d | Directory for storing configuration files|
|  | /var/dockerdata/mysqlcluster/mysql01/initdb.d | Directory with sql initialisation files for fresh DB|
|  | /var/dockerdata/mysqlcluster/mysql01/lib | Data storage for MySQL instance|
|  | /var/dockerdata/mysqlcluster/mysql01/log | Directory where bin-log files for replication will be stored |
| mysqlrouter | /var/dockerdata/mysqlcluster/mysqlrouter/log | Directory where log will be written|

#### Docker

```
$ docker swarm init
$ docker network create -d overlay mysqlcluster
```

Join 1 or more nodes to the cluster according to Docker documentation.

#### Compose files

Make sure to look through the compose files for the volume mappings.
In this example all is mapped to /var/dockerdata/mysqlcluser/<directories>. Adjust this to your own liking or create the same structure as used in this example (Preparation).

In this basic setup the <root> user is used in all communication. The password for this user has to be put in the compose files.

#### Config Files

This setup will create a custom config file with all the needed info to setup replication for the Master and the Slave.
The config file can be viewed after creation of the services in their respective config.d volumes.

## Installation

#### Step 1 - Initialize the Master

Database wise there are 2 options:
 - Create a fresh DB from the initdb.d directory
 - Copy an existing database to the \lib\ directory

When either of those is setup run below compose file:

```
$ docker deploy --compose-file docker-compose-mysql-cluster-master.yml mysqlcluster
```

Now wait till the master is up and running and accepting client connections

#### Step 2 - Initialize the Slave

```
$ docker deploy --compose-file docker-compose-mysql-cluster-slave.yml mysqlcluster
```

The replication will now be setup and the Master database will get copied to the Slave.
Wait till the process is completed and the Slave is accepting client connections

#### Step 3 - Initialize MySQLFailover and MySQLRouter

```
$ docker deploy --compose-file docker-compose-mysql-cluster-connect.yml mysqlcluster
```

## Connecting to the cluster

Now that everything is up and running you can connect to the cluster over port 3306 (MySQLRouter) or by accessing the Master and Slave directly over port 3307 and port 3308.

## Credits and License

Nicomak blog (https://github.com/nicomak) has been used as a base on how to setup replication between 2 MySQL servers.

The files are free to use and you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation.