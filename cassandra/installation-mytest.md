sudo yum install java-1.8.0-openjdk-devel -y & yum install wget -y

sudo groupadd -g 9082 cassandra
sudo useradd -u 9081 -g 9082 -d /home/cassandra cassandra 

sudo mkdir -p /data/cassandra/data ## appending only
sudo mkdir -p /data/cassandra/saved_caches
sudo mkdir -p /data/cassandra/commitlog ## disaster recovery
sudo mkdir -p /data/cassandra/hints

sudo chown -R cassandra:cassandra /data/
sudo mkdir -p /var/log/cassandra
sudo mkdir -p /var/run/cassandra
sudo chown -R cassandra:cassandra /var/log/cassandra
sudo chown -R cassandra:cassandra /var/run/cassandra
sudo chown -R cassandra:cassandra /data/cassandra/hints
sudo chown -R cassandra:cassandra /data/cassandra/saved_caches
sudo chown -R cassandra:cassandra /data/cassandra/commitlog

sudo chown -R cassandra:cassandra /opt/

---this will get in sudoers & from root user 
usermod -aG wheel cassandra

passwd cassandra

su - cassandra

Old Versions
List of versions https://archive.apache.org/dist/cassandra/
wget http://archive.apache.org/dist/cassandra/2.1.16/apache-cassandra-2.1.16-bin.tar.gz (stable release)
wget http://archive.apache.org/dist/cassandra/3.11.2/apache-cassandra-3.11.2-bin.tar.gz
wget https://archive.apache.org/dist/cassandra/3.11.4/apache-cassandra-3.11.4-bin.tar.gz
wget https://archive.apache.org/dist/cassandra/3.11.5/apache-cassandra-3.11.5-bin.tar.gz


sudo wget https://archive.apache.org/dist/cassandra/4.1.7/apache-cassandra-4.1.7-bin.tar.gz

sudo wget https://archive.apache.org/dist/cassandra/5.0.1/apache-cassandra-5.0.1-bin.tar.gz

sudo wget https://archive.apache.org/dist/cassandra/5.0.3/apache-cassandra-5.0.3-bin.tar.gz

mv apache-cassandra-4.1.7-bin.tar.gz /opt/
cd

[cassandra@server2024 opt]$ tar -xvf apache-cassandra-4.1.7-bin.tar.gz

cassandra@centos07 conf]$ pwd
/opt/apache-cassandra-4.1.7/conf
[cassandra@centos07 conf]$ ls -ll cassandra.yaml 
-rw-r--r--. 1 cassandra cassandra 90055 Sep 18 17:30 cassandra.yaml


Two types of parameters

Local (They are exclusively set only one a current node) 

#Current Server IPaddress's
listen_address:
rpc_address: 


Global (This is set common across all the nodes in this cluster)

cluster_name: 'myappcluster'

data_file_directories:
     - /data/cassandra/data
     
commitlog_directory: /data/cassandra/commitlog
saved_caches_directory: /data/cassandra/saved_caches
- seeds: "10.166.0.4"

# for production snitch from simple to change Gossiping ..
endpoint_snitch: GossipingPropertyFileSnitch

***************************

[cassandra@c2021 ~]$ which java
/bin/java
[cassandra@c2021 ~]$ ls -ltr /bin/java
lrwxrwxrwx. 1 root root 22 Jan 14 01:31 /bin/java -> /etc/alternatives/java
[cassandra@c2021 ~]$ ls -ltr /etc/alternatives/java
lrwxrwxrwx. 1 root root 73 Jan 14 01:31 /etc/alternatives/java -> /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.275.b01-0.el7_9.x86_64/jre/bin/java

********************************

#Cassandra (.bash_profile)
#Updated Date 25-08-2025
export CASSANDRA_HOME=/opt/apache-cassandra-4.1.7/
export CASSANDRA_CONF=$CASSANDRA_HOME/conf
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.0.1.el9.x86_64/jre/
#export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk/jre
export CASSANDRA_LOG=/var/log/cassandra/system.log
export TMP=/tmp
export TMPDIR=$TMP
export PATH=$PATH:$HOME/bin:$JAVA_HOME/bin:$CASSANDRA_HOME/bin:/opt/cassandra-sstable-tools/bin:.
export CLASSPATH=$JAVA_HOME:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/jre/lib/ext:$CASSANDRA_HOME/lib:$CASSANDRA_CONF:$CASSANDRA_HOME/bin:.
alias casslog='tail -100f $CASSANDRA_LOG'

********************************

[cassandra@centos07 ~]$ pwd
/home/cassandra
[cassandra@centos07 ~]$ Cassandra

INFO  [main] 2025-01-04 22:48:42,060 Gossiper.java:2333 - Waiting for gossip to settle...
INFO  [main] 2025-01-04 22:48:50,063 Gossiper.java:2364 - No gossip backlog; proceeding
INFO  [main] 2025-01-04 22:48:50,192 CassandraDaemon.java:488 - Prewarming of auth caches is disabled


ps -ef|grep java 
ps -ef|grep cassandra

[cassandra@c2021 ~]$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.142.0.2  70.72 KiB  256          100.0%            93ad25e9-5a05-4046-90f2-2ea9beb03686  rack1


[cassandra@c2021 ~]$ nodetool info
ID                     : 93ad25e9-5a05-4046-90f2-2ea9beb03686
Gossip active          : true
Thrift active          : false
Native Transport active: true
Load                   : 70.72 KiB
Generation No          : 1610674169
Uptime (seconds)       : 145
Heap Memory (MB)       : 98.63 / 1004.00
Off Heap Memory (MB)   : 0.00
Data Center            : datacenter1
Rack                   : rack1
Exceptions             : 0
Key Cache              : entries 10, size 816 bytes, capacity 50 MiB, 49 hits, 65 requests, 0.754 recent hit rate, 14400 save period in seconds
Row Cache              : entries 0, size 0 bytes, capacity 0 bytes, 0 hits, 0 requests, NaN recent hit rate, 0 save period in seconds
Counter Cache          : entries 0, size 0 bytes, capacity 25 MiB, 0 hits, 0 requests, NaN recent hit rate, 7200 save period in seconds
Chunk Cache            : entries 9, size 576 KiB, capacity 219 MiB, 22 misses, 86 requests, 0.744 recent hit rate, 68.038 microseconds miss latency
Percent Repaired       : 100.0%
Token                  : (invoke with -T/--tokens to see all 256 tokens)

--error 
[cassandra@server2024 ~]$ cassandra
[cassandra@server2024 ~]$ OpenJDK 64-Bit Server VM warning: Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.

[root@server2024 ~]# yum install java-1.8.0-openjdk-devel

vi .bash_profile #in cassandra user

JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk/jre

then execute ..cassandra
------------------------------------
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
JRE_HOME=/usr/lib/jvm/java-1.8.0-openjdk/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

https://cassandra.apache.org/doc/4.1/cassandra/getting_started/java11.html


cat /opt/apache-cassandra-4.1.7/conf/cassandra.yaml |grep "seeds:"

cat cassandra.yaml | grep -E "seeds|listen_address|rpc_address|GossipingPropertyFileSnitch"


cqlsh hostname 9042
or
cqlsh ipaddress 


--for rack1 changing to vm02-rack1

---from vm02 
rm -rf /data/cassandra/data/system/*

---from vm01
[cassandra@vm01 ~]$ nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load        Tokens  Owns (effective)  Host ID                               Rack
UN  10.166.0.7  138.54 KiB  16      35.7%             d5ce6ec4-074f-455d-8cbd-5b2789966229  rack1
UN  10.166.0.4  258.83 KiB  16      32.7%             5e3fd185-eee5-4ece-a1c8-71fda9cc3f0e  rack1
DN  10.166.0.5  137.03 KiB  16      31.6%             2a7520fa-24c6-4a34-9c31-680f5375ffc4  rack1

Datacenter: dc2
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load        Tokens  Owns (effective)  Host ID                               Rack
UN  10.138.0.3  70.7 KiB    16      0.0%              67de3852-2b7c-4efb-99bd-da22c29d348a  vm05-rack1
-----

--- Host ID for vm02(10.166.0.5), after removing do changes then Restart
nodetool removenode 2a7520fa-24c6-4a34-9c31-680f5375ffc4

