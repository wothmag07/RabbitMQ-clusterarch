# RabbitMQ Cluster Setup Guide

RabbitMQ, a resilient open-source message broker, streamlines communication among distributed systems. This comprehensive guide outlines precise steps to establish a cross-site RabbitMQ cluster, enhancing latency mitigation, and scalability. In this configuration, we opt for QUORUM_QUEUE over STATIC_QUEUE, as QUORUM_QUEUE excels in durability and fault tolerance through quorum replication, while STATIC_QUEUE represents a durable queue with conventional persistence, suitable for moderate scalability.

## Prerequisites

Before starting the installation, ensure that you have the following prerequisites in place:

- Linux CentOS 7/higher environment.
- Minimum of 2 nodes from each site.
- Java package installed on all nodes (`yum install java.1.8*`).

## Diagram

This cross-site cluster architecture diagram depicts RabbitMQ implementation, where each server operates RabbitMQ, a key open-source message broker enabling smooth communication and data exchange among applications. In the primary Site A, Server 1 handles production loads, while Server 2 serves as a standby or secondary broker. At the disaster recovery Site B, Server 1 mirrors Site A's Server 1, ready for failover, and Server 2 manages additional load. RabbitMQ replication, utilizing features like federation or shovel, ensures data synchronization between sites. High availability configurations, clustered setups, mirrored queues, and a robust disaster recovery strategy collectively strengthen the architecture, ensuring resilience and uninterrupted message processing.

![RabbitMQ Diagram](./RabbitMQ-clusterarch/images/rabbitmq.png)


## Steps

### 1. Cluster Configuration

When configuring the RabbitMQ cluster, consider the following:

- If you are installing RabbitMQ package version greater than 3.9, there is no need to install the Erlang package.
- Else you have to install erlang package along with rabbitmq installation.

### 2. Host Configuration

Update the host entries on each server node to include the hostnames of all servers considered as part of the cluster:

```bash
vim /etc/hosts
# Example:
# Hostname1 10.X.X.X
# Hostname2 10.X.X.X
# Hostname3 10.X.X.X
# Hostname4 10.X.X.X
```

### 3. RabbitMQ Installation

Install RabbitMQ Server packages from the EPEL repository:

```bash
yum -y install epel-release
yum install rabbitmq-server-3.12.12-1.el8.noarch.rpm
```
### 4. Start & Enable Services

Once the installation is finished, initiate the RabbitMQ service, configure it to launch automatically on system boot, and regularly check its status on all nodes.

```bash
systemctl start rabbitmq-server.service
systemctl status rabbitmq-server.service
systemctl enable rabbitmq-server.service

```
### 5. Firewall Configuration

Open the necessary ports for RabbitMQ to run:

|Port     | Description |
|---------| ---------------|
| 4369    | Erlang Port Mapper Daemon (epmd) for node name resolution. Nodes must reach each other and epmd for clustering to work.|
| 15672   | RabbitMQ Management Console port for versions 3.x and above.|
| 5672    | RabbitMQ main port for AMQP communication.|


```bash
sudo firewall-cmd --add-port=15672/tcp --permanent
sudo firewall-cmd --add-port=5672/tcp --permanent
sudo firewall-cmd --add-port={4369/tcp,25672/tcp} --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### 6. Connectivity Checks

For checking the port connectivity, install the network tools package which is present in yum repository.

```bash
yum install net-tools
```
Check port connectivity between nodes:

```bash
curl -v telnet://ip:port
netstat -tnulp|grep rabbit
```

### 7. RabbitMQ Management Plugins

Enable RabbitMQ management plugins for viewing the monitoring dashboards and restart the RabbitMQ service:

```bash
rabbitmq-plugins enable rabbitmq_management
sudo systemctl restart rabbitmq-server
```

### 8. Cluster Joining

Repeat Steps 1 to 7 on the remaining nodes. To establish a cluster, ensure that the Erlang cookies are consistent across all nodes; if not, synchronize them to be identical on all nodes.

```bash
cat /var/lib/rabbitmq/.erlang.cookie
scp /var/lib/rabbitmq/.erlang.cookie root@node02:/var/lib/rabbitmq/
```

Restart RabbitMQ service on both nodes, check the status and stop the application in the slave (node 2).

```bash
systemctl restart rabbitmq-server
systemctl status rabbitmq-server
rabbitmqctl stop_app
```	
After stopping the application, switch the log and mnesia folder to some user defined mounts because, if the mount storage gets full, the application might get stopped because of this.
Default location of logs and mnesia folder:
```directory path
“/etc/rabbitmq/logs”
“etc/rabbitmq/mnesia”
```



Join the cluster on node01 and start the application:

```bash
rabbitmqctl join_cluster rabbit@node01
rabbitmqctl start_app
```

### 8. User Management

Delete the default user 'guest' and add new users:

```bash
rabbitmqctl delete_user guest
rabbitmqctl add_user admin (__password_______)
rabbitmqctl add_user org
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_user_tags org administrator
rabbitmqctl set_permissions -p / org ".*" ".*" ".*"
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
rabbitmqctl list_users
```

### 9. Queue Mirroring Setup

It is essential to set up a 'High Availability (HA) policy' cluster for queue mirroring and replication across all nodes in the RabbitMQ cluster. In the event of a failure on the node hosting the queue master, the oldest mirror will be promoted to the new master, provided it is synchronized. This depends on the 'ha-mode' and 'ha-params' policies.

#### Setting up 'ha-all' Policy

Ensure that all queues on the RabbitMQ cluster mirror to every node in the cluster:

```bash
rabbitmqctl set_policy ha-all ".*" '{"ha-mode":"all"}'
```

#### Executing Policies with 'ha-mode: nodes'

Execute policies with 'ha-mode: nodes' for specific queues:

```bash
rabbitmqctl set_policy ha-nodes "QUEUE_NAME" -p /etchost '{"ha-mode":"nodes","ha-params":["rabbit@HOSTNAME1","rabbit@HOSTNAME2"],"ha-sync-mode":"automatic"}' --priority 1
```

#### Other Scenarios:

- Setting up 'ha-two' Policy: Mirroring queues starting with 'two.' to two nodes in the cluster:

```bash
sudo rabbitmqctl set_policy ha-two "^two\." '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

- Setting up 'ha-nodes' Policy: Mirroring queues starting with 'nodes.' to specific nodes 'node02' and 'node03':

```bash
sudo rabbitmqctl set_policy ha-nodes "^nodes\." '{"ha-mode":"nodes","ha-params":["rabbit@node02", "rabbit@node03"]}'
```

To view all available policies, use the following command:

```bash
rabbitmqctl list_policies
```

If you wish to clear a specific policy, for example, 'ha-two,' use:

```bash
rabbitmqctl clear_policy ha-two
```

This will remove the specified policy from the RabbitMQ configuration. Adjust the policy name as needed.

### 10. Viewing RabbitMQ Dashboard

To access the RabbitMQ dashboard and view essential information such as cluster details, status, and existing queues along with their request counts, follow these steps:

1. Open your web browser.
2. Enter the IP address of the node along with port '15672' in the address bar.

Example:
[http://ip:15672/](http://ip:15672/)

This will lead you to the RabbitMQ dashboard, providing comprehensive insights into the cluster's health and the status of queues. Explore the dashboard to monitor and manage your RabbitMQ server efficiently.

### 11. Queue Creation

To establish queues for your application to transmit messages, you have two options: either create them through the RabbitMQ dashboard or utilize the command-line interface (CLI) with the following command:

```bash
rabbitmq-queues grow “rabbit@HOSTNAME” “all” –vhost-pattern “/host” –queue-pattern “QUEUE-NAME”
```

### 12. Testing the Setup

For testing purposes, create a Java service producer and consumer designed to send and receive messages, respectively, using a designated messaging queue (e.g., QUORUM_QUEUE). Follow these steps:

1. Push Messages Using Producer JAR:
   - Utilize the producer JAR to send messages.
   - Monitor the RabbitMQ dashboard to observe an increment in the queue requests count.

2. Fetch Messages Using Consumer JAR:
   - Utilize the consumer JAR to retrieve messages.
   - Observe a gradual decrement in the queue requests count on the RabbitMQ dashboard.

This testing procedure ensures that your RabbitMQ setup is functioning as expected, and the messaging queue is handling messages appropriately. Adjust the queue names and configurations based on your application's requirements.

Congratulations! Your RabbitMQ cluster is now set up and ready for use.
