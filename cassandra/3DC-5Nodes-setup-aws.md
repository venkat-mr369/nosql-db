**Apache Cassandra 5.0.1** for your **3 regions (us, west, europe)** with **5 nodes each (total 15)** named `amscluster`, using the IP ranges you gave:

* `us` = `10.10.166.1` → `10.10.166.5`
* `west` = `10.10.144.1` → `10.10.144.5`
* `europe` = `10.10.133.1` → `10.10.133.5`

I can’t browse the web from here (so I can’t pull the *very latest* distro repo lines), but below is a complete, practical, version-appropriate plan (install, config, start order, checks) plus OpsCenter management node + agent deployment — ready to paste into your automation or run manually. If you want I will also produce per-node files / systemd units after this.

---

# Overview (high level)

* Cluster name: `amscluster`
* Cassandra version: **5.0.1** (tarball / package install)
* Replication strategy: `NetworkTopologyStrategy` — recommended RF = **3 per DC** (examples below).
* Snitch: `GossipingPropertyFileSnitch` (multi-DC) with `cassandra-rackdc.properties` set per node.
* Seeds: choose **2 seeds per DC** (good practice) → 6 seeds total (see list below).
* JVM: **OpenJDK 17** recommended for Cassandra 5.x (confirm in your environment).
* OpsCenter: management node (separate VM) + agent on each Cassandra node.

---

# Inventory / hostnames mapping (suggested)

Map IPs to simple hostnames to keep configs readable:

```
us:
  10.10.166.1  ams-us-1
  10.10.166.2  ams-us-2
  10.10.166.3  ams-us-3
  10.10.166.4  ams-us-4
  10.10.166.5  ams-us-5

west:
  10.10.144.1  ams-west-1
  10.10.144.2  ams-west-2
  10.10.144.3  ams-west-3
  10.10.144.4  ams-west-4
  10.10.144.5  ams-west-5

europe:
  10.10.133.1  ams-eu-1
  10.10.133.2  ams-eu-2
  10.10.133.3  ams-eu-3
  10.10.133.4  ams-eu-4
  10.10.133.5  ams-eu-5
```

Add those to `/etc/hosts` on each node (or keep DNS records).

---

# Seeds (recommended)

Pick `.1` and `.2` from each DC as seeds:

```
SEEDS="10.10.166.1,10.10.166.2,10.10.144.1,10.10.144.2,10.10.133.1,10.10.133.2"
```

Start *seed* nodes first (but you can start all seeds in parallel if network is up).

---

# Pre-reqs (per node)

1. Create OS user (optional): `cassandra` or run as root.

2. Install Java 17 (OpenJDK 17). Example (Debian/Ubuntu):

   ```bash
   sudo apt update
   sudo apt install -y openjdk-17-jdk
   java -version
   ```

   RHEL/CentOS:

   ```bash
   sudo dnf install -y java-17-openjdk
   ```

   Set `JAVA_HOME` if needed.

3. Ensure time sync (chrony/ntpd), hostname correct, and `/etc/hosts` populated.

4. Ports (open in firewall between nodes and OpsCenter):

   * Cassandra internode (plain): **7000**
   * Cassandra internode (TLS): **7001** (if you enable SSL)
   * JMX: **7199**
   * CQL (native transport): **9042**
   * OpsCenter web UI: **8888** (default)
   * Ensure agent/opscenter ports per your OpsCenter version (see OpsCenter docs).

---

# Install Cassandra 5.0.1 (tarball method — portable & universal)

(Use package repos if you have them; tarball is safe if you cannot access repo info.)

On each node:

```bash
# switch to /opt
sudo mkdir -p /opt/cassandra
cd /opt/cassandra

# download tarball (replace with actual URL if using mirror)
# example filename: apache-cassandra-5.0.1-bin.tar.gz
# curl -O <download_url>

sudo tar xzf apache-cassandra-5.0.1-bin.tar.gz
sudo ln -s apache-cassandra-5.0.1 cassandra-current
sudo chown -R cassandra:cassandra /opt/cassandra
```

Set env variables (e.g., in `/etc/profile.d/cassandra.sh`):

```bash
export CASSANDRA_HOME=/opt/cassandra/cassandra-current
export PATH=$PATH:$CASSANDRA_HOME/bin
```

Create data/log dirs (owned by cassandra user):

```bash
sudo mkdir -p /var/lib/cassandra/data /var/lib/cassandra/commitlog /var/log/cassandra
sudo chown -R cassandra:cassandra /var/lib/cassandra /var/log/cassandra
```

---

# cassandra.yaml changes (critical fields)

Edit `$CASSANDRA_HOME/conf/cassandra.yaml` on every node — only change the keys shown below (I show examples and per-node placeholders).

Key changes (replace `<NODE_IP>` and seeds string accordingly):

```yaml
cluster_name: 'amscluster'
num_tokens: 256                         # default for vnodes; keep consistent across nodes

# seeds (same on all nodes)
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "10.10.166.1,10.10.166.2,10.10.144.1,10.10.144.2,10.10.133.1,10.10.133.2"

# network
listen_address: "<NODE_IP>"            # the node's own IP, e.g. 10.10.166.1
# or listen_interface if using that mode
# RPC/native transport
rpc_address: 0.0.0.0                   # bind native transport to all; clients connect via broadcast_rpc_address
broadcast_rpc_address: "<NODE_IP>"

# internode broadcast (useful for NAT/cloud)
broadcast_address: "<NODE_IP>"

# storage paths
data_file_directories:
  - /var/lib/cassandra/data
commitlog_directory: /var/lib/cassandra/commitlog
saved_caches_directory: /var/lib/cassandra/saved_caches

endpoint_snitch: GossipingPropertyFileSnitch
```

**Important**: keep `seeds` identical on *all* nodes. Do **not** add more than ~2–3 seeds per DC in small clusters, but 6 total (2 per DC) is okay for your set.

---

# cassandra-rackdc.properties (per node)

Set DC and rack here — required for `GossipingPropertyFileSnitch`. Example file:

```
# location: $CASSANDRA_HOME/conf/cassandra-rackdc.properties
dc=us
rack=rack1
```

For our mapping:

* For `10.10.166.*` --> `dc=us`
* For `10.10.144.*` --> `dc=west`
* For `10.10.133.*` --> `dc=europe`

You can set `rack=rack1` for all, or map multiple racks if applicable (e.g., `rack=R1`, `R2`).

---

# JVM options (cassandra-env.sh or jvm.options)

Tuning short notes (place in `$CASSANDRA_HOME/conf/jvm.options` or `cassandra-env.sh`):

* Heap: set `-Xms` and `-Xmx` to 50% of machine RAM but ≤ 32GB (if you have >64GB RAM you can go higher but watch GC). Example for 32GB host:

  ```
  -Xms16G
  -Xmx16G
  ```
* Leave CMS/G1 flags as default for Cassandra 5.x (the distribution jvm.options is tuned; only edit if you know GC tuning).

---

# Start order & sanity checks

1. On all nodes set `/etc/hosts`, Java installed, cassandra files in place, configs edited.
2. Start the 6 seed nodes first (we recommended `.1` & `.2` in each DC) — or start seeds in each DC at roughly the same time.

   ```bash
   # as cassandra user
   $CASSANDRA_HOME/bin/cassandra -R      # start in foreground or background
   # or use systemd service if you create unit file
   ```
3. On each seed node, check:

   ```bash
   nodetool status
   # look for UN for the seed itself and peers when they join
   ```
4. Start non-seed nodes (the remaining 9).
5. Once all nodes started, run on any node:

   ```bash
   nodetool status
   ```

   You should see 15 nodes with tokens and their DC listed (UN = Up/Normal).

---

# Keyspace & replication example (RF=3 per DC)

Create a keyspace that keeps 3 copies in each region:

```sql
CREATE KEYSPACE app_ks WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'us': 3,
  'west': 3,
  'europe': 3
} AND durable_writes = true;
```

* Total replicas per row = 9 (3×3).
* `LOCAL_QUORUM` for reads/writes recommended for low-latency regional operations:

  * LOCAL_QUORUM per DC = floor(3/2) + 1 = 2

---

# Repairs & maintenance

* Run `nodetool repair -pr` per node regularly (per keyspace). For big clusters use incremental or repair tools (Reaper, Cassandra 4+ techniques).
* Backups: take `nodetool snapshot` and copy snapshots off-site.
* Seeds: keep the seed list stable; change seeds only with care.

---

# OpsCenter setup (management + agents)

You requested OpsCenter — set up a dedicated OpsCenter server (VM) and install OpsCenter there. Then install **OpsCenter Agent** on every Cassandra node so OpsCenter can manage/monitor the cluster.

**NOTE:** I can’t fetch the exact OpsCenter package URL for your environment right now, but here’s a robust workflow that works for most OpsCenter versions:

1. Pick a management VM (example `10.10.166.100`) — ensure it can reach all Cassandra nodes on required ports.
2. Install OpsCenter server on management VM:

   * If package available: `apt install datastax-opscenter` or `yum install opscenter` (your OS dependent).
   * Or download OpsCenter tarball, extract to `/opt/opscenter` and run the daemon.
3. Edit OpsCenter config (example file `/etc/opscenter/opscenterd.conf` or `/opt/opscenter/conf/opscenterd.conf`):

   ```ini
   [webserver]
   interface = 0.0.0.0
   port = 8888

   [cassandra]
   cluster_name = amscluster
   seeds = 10.10.166.1,10.10.166.2,10.10.144.1,10.10.144.2,10.10.133.1,10.10.133.2
   ```
4. Start opscenterd:

   ```bash
   sudo systemctl start opscenterd   # if packaged
   # or ./opscenterd -d if tarball-run
   ```
5. Install OpsCenter Agent on all cassandra nodes:

   * Agent package installed under `/var/lib/opscenter-agent` or `/opt/opscenter-agent`.
   * Configure agent to point at OpsCenter server (set `stomp_interface` or `opscenter_host` in agent config). Example `/etc/opscenter/agent.conf`:

     ```ini
     [agent]
     stomp_interface = 10.10.166.100   # OpsCenter server IP
     ```
   * Start the agent on each node:

     ```bash
     sudo systemctl start opscenter-agent
     ```
6. In OpsCenter web UI ([http://10.10.166.100:8888](http://10.10.166.100:8888)), add/verify cluster — agents should auto-register. Configure backups, alerts, repair scheduling from UI.

> If you want, I will generate the exact agent-install loop (bash) and the agent config template targeted for your 15-node IP list.

---

# Firewall/Ports checklist (minimum)

Open between nodes and OpsCenter:

* 7000 TCP (internode)
* 7001 TCP (internode TLS, if enabled)
* 9042 TCP (CQL)
* 7199 TCP (JMX)
* 8888 TCP (OpsCenter web)
  Also open whatever OpsCenter agent ports your OpsCenter version requires.

---

# Quick verification commands

On any node:

```bash
nodetool info
nodetool status
nodetool ring   # (status provides similar info)
nodetool netstats
tail -f /var/log/cassandra/system.log
```

From OpsCenter server: open web UI and confirm agents connected.

---

# Example automation snippets

1. Example `/etc/hosts` entry (append to all hosts):

```
10.10.166.1 ams-us-1
10.10.166.2 ams-us-2
...
10.10.133.5 ams-eu-5
```

2. Example loop to copy agent to all nodes (psuedo):

```bash
for ip in 10.10.166.{1..5} 10.10.144.{1..5} 10.10.133.{1..5}; do
  scp opscenter-agent.rpm root@$ip:/tmp/
  ssh root@$ip "rpm -ivh /tmp/opscenter-agent.rpm; sed -i 's/^stomp_interface.*/stomp_interface = 10.10.166.100/' /etc/opscenter/agent.conf; systemctl enable --now opscenter-agent"
done
```

---

# Notes, caveats & next steps

* I can’t pull exact package repo lines or OpsCenter package URLs right now (web access disabled here). Before production, confirm the Cassandra 5.0.1 tarball URL or package repo for your OS and the correct OpsCenter version compatibility with Cassandra 5.0.1 (OpsCenter versions and agent config options sometimes change across releases).
* If you’d like I will prepare:

  1. A per-node `cassandra.yaml` snippet with the exact `listen_address`, `broadcast_address`, and `broadcast_rpc_address` filled for each IP.
  2. `cassandra-rackdc.properties` files for all nodes.
  3. A systemd unit file for Cassandra and OpsCenter agent.
  4. A Bash automation script to install Java, unpack Cassandra, place configs, and start nodes in correct order.
 
----
<img width="1600" height="1000" alt="amscluster_diagram" src="https://github.com/user-attachments/assets/5673a0e4-eb73-495d-bebf-24145a629258" />




