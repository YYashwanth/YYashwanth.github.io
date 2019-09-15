---
layout: post
title: "Layer 4 Load Balancing based on Source IP through HAProxy"
description: "Setup and configure HAProxy on AWS to facilitate a layer 4 load balancer"
date: 2019-08-12 11:30:18
comments: true
description: "Layer 4 Load Balancing"
keywords: "AWS, HAProxy, Load Balancer, IP Address, EC2, Source IP"
categories:
    - HAProxy
    - Load Balancer
    - Proxies
tags:
- AWS
- HAProxy
- EC2
- Load Balancing
---
While playing with Load Balancers during my preparation for AWS Certifications, I found that only Network Load Balancer supports source IP based routing. The only drawback is that it also includes source port in the routing algorithm while making a routing decision. I wanted a load balancer which solely routes using source IP. 

I looked out in the market to see which product can help me. I started comparing HAProxy and NGINX as they've got quite a good reputation in the market. I thought NGINX would meet the requirements but it routes based only on the first 2 octets. The enterprise version of NGINX supports routing based on complete source IP. But the open source version of HAProxy supports routing based on source IP. So I chose HAProxy to PoC a layer 4 load balancer. HAProxy only supports open source operating systems and we had to go with Amazon Linux 2 operating system.

<h2> Installing HAProxy </h2>

Once you login into the linux server, run: 

```bash
sudo yum install -y haproxy
 ```
This installs haproxy in the server.

<h2> Configuring HAProxy to balance load on source IP </h2>

Open the HAProxy configuration haproxy.cfg located in /etc/haproxy/haproxy.cfg. To open the configuration file run:

```bash
cd /etc/haproxy
sudo vi haproxy.cfg
 ```
 Delete the content inside haproxy.cfg or the easier way is to delete the existing file and create a new file:

```bash
sudo rm -f haproxy.cfg
sudo vi haproxy.cfg
 ```


<h2>Defining the global variables</h2>

There are certain variables that could be defined to be used globally across the configuration. Copy the below the lines in your /etc/haproxy/haproxy.cfg.


```yaml

global
    log         127.0.0.1 local0          #redrecting the log messages to the local0
    chroot      /var/lib/haproxy          #change current executing directory to haproxy
    pidfile     /var/run/haproxy.pid      #creating a pid file to store process id
    maxconn     50000                     #maximum connections
    user        haproxy                   #username
    group       haproxy                   #user group which contains the user haproxy
    daemon
    stats socket ipv4@127.0.0.1:9999 level admin #runtime interactive API of haproxy
    stats socket /var/run/haproxy.sock mode 666 level admin #haproxy runtime API
    stats timeout 2m                      #timeout of the stats

```

<h2>Defining the default variables</h2>

There are certain variables which would not be changed across the configuration and those variables can be defined together under the defaults section of the config. Copy the below lines after the global section.


```yaml

defaults
    mode                    tcp                 #the mode of transport protocol
    log                     global              # what services are being logged.
    option                  dontlognull         #logging options
    retries                 9999               
    timeout queue           60m
    timeout connect         30s
    timeout client          60m
    timeout server          60m
    timeout check           60s
    maxconn                 15000

```

<h2>Defining the health dashboard</h2>

We can have a dashboard for this server where we can see the health of the HAProxy server and the health of the backend servers. Copy the below lines after the defaults section


```yaml

frontend stats                  #name of the frontend
    bind *:8080                 #Binds the port number to access health dashboard
    mode http                   #protocol mode for displaying the stats
    stats enable                # enabling the stats
    stats hide-version
    stats uri /stats            #defining the uri endpoint path

```
The dashboard looks like this once we hit the server

<img class="mb-2" src="{{site.baseurl}}/assets/images/haproxy-health.png" alt="" height="400" width="900">

<h2>Defining the front end to accept incoming connections</h2>

We need to create a front end to for the users to send requests to. So, copy the lines below after the stats frontend

```yaml
frontend haproxy_frontend                   #name of the frontend
    bind *:80                               #listening on port 80
    default_backend haproxy_backend         #routes to defined backend
    option tcplog                           #logging format of the server

```
<h2>Before going to back-end configuration, lets read about Stick Tables</h2>

Let's say you are trying to create a highly available HAProxy load balancer and wanted to make sure that even if one HAProxy server goes down, you can still reach your back end targets. That's a fair requirement, right? But how are the HAProxy servers going to remember which backend server were sent to earlier? If there are 2 HAProxy servers HA-1 and HA-2 and HA-1 went down, how will HA-2 know that HA-1 sent you to BackendServer-A earlier? 

To help answer this, HAProxy provides a feature called Stick Tables where they store the source IP and the destination server that IP has been routed to. This stick table can be shared with other HAProxy servers, which are part of the same cluster, so that the routing can be consistent.

If you would like to read more about stick tables and different types of routing algorithms, please refer to <a href="https://www.haproxy.com/blog/introduction-to-haproxy-stick-tables/">Sticktables on HAProxy</a>

![HAProxy-Server-A]({{site.baseurl}}/assets/images/HAP-ServerA.png)

![HAProxy-Server-B]({{site.baseurl}}/assets/images/HAP-ServerB.png)

In the above images, you can see that the entries are same in two different servers. Let me explain the most relevant entries from left to right.
<ul>
<li>The first entry on the left is the address id allocated to store the entry</li>
<li>The <b>Key</b> entries are where the IP address is stored </li>
<li>The <b>exp</b> field shows a counter which servers a TTL. As soon as it reaches 0, the entry is removed from the stick table</li>
<li>The <b>server_id</b> field shows the destination server to which the source request was routed to.</li>
</ul>

If you would like to read more about defining HAProxy clusters or peering configuration please refer to <a href="http://haproxy.tech-notes.net/3-5-peers/">HAProxy Peers</a>


<h2>Defining the back-end to route the incoming requests</h2>

We need to create a backend definition in the config for the HAProxy to know where to route the incoming requests to. HAProxy uses stick tables to store the session affinity information and we can share that table with its peers so that no matter which HAProxy server the incoming request is routed to, the request is sent to the same backend server to which the affinity is present.

```yaml
backend haproxy_backend                    #name of the backend
    mode tcp                               #mode of the traffic tcp or http
    balance source                         #name of the routing algorithm. 
    stick-table type ip size 20k expire 8h peers haproxy-cluster  #stick table,naming the peers
    stick on src                           #store the source IP addresses into the stick table
    server <backend server name1> <IP of backend server1>:80 check port 80 maxconn 5000
    server <backend server name2> <IP of backend server2>:80 check port 80 maxconn 5000    
```


<h2>Defining the Peers</h2>

We define the HAProxy peers section by adding the below code block.

```yaml
peers hapeers
    peer <name of peer 1> <IP of peer 1>:1024
    peer <name of peer 2> <IP of peer 2>:1024
```

<h2>Final Config</h2>

Once the above contents are defined, the final config file should look like below:

```yaml
global
    log         127.0.0.1 local0
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     50000
    user        haproxy
    group       haproxy
    daemon
    stats socket ipv4@127.0.0.1:9999 level admin
    stats socket /var/run/haproxy.sock mode 666 level admin
    stats timeout 2m
 
defaults
    mode                    tcp
    log                     global
    option                  dontlognull
    retries                 9999
    timeout queue           60m
    timeout connect         30s
    timeout client          60m
    timeout server          60m
    timeout check           60s
    maxconn                 15000
 
frontend stats
    bind *:8080
    mode http
    stats enable
    stats hide-version
    stats uri /stats
 
frontend haproxy_frontend
    bind *:80
    default_backend haproxy_backend
    option tcplog
 
backend haproxy_backend
    mode tcp
    balance source
    stick-table type ip size 20k expire 8h peers hapeers
    stick on src
    server <backend server name1> <IP of backend server1>:80 check port 80 maxconn 5000
    server <backend server name2> <IP of backend server2>:80 check port 80 maxconn 5000
 
peers hapeers
    peer <name of peer 1> <IP of peer 1>:1024
    peer <name of peer 2> <IP of peer 2>:1024
```

I know the article is pretty lengthy and I will not hold you up any longer. I will continue writing about the stick tables configured above and how to test if the stick tables are configured correctly in a different post.

As always, if you can help me out with an alternative approach to solve my problem stated above, please shoot an email at yyellapragada@gmail.com.

Always happy to learn!


