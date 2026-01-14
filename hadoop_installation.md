# Hadoop Installation Guide (with NameNode and DataNode Roles)

## Step 1: Download and Extract Hadoop

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
tar xvf hadoop-3.4.1.tar.gz
sudo mv hadoop-3.4.1 /opt/hadoop
```

## Step 2: Add Hadoop to PATH

Edit `~/.bashrc`:

```bash
nano ~/.bashrc
```

Append:

```bash
# Hadoop Environment Variables
export HADOOP_HOME=/opt/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
```

Apply changes:

```bash
source ~/.bashrc
```

---

```bash
cd /opt/hadoop/etc/hadoop
```

## Step 3: Configure `core-site.xml` (Common)

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://<namenode-hostname-or-ip>:9000</value>
  </property>
</configuration>
```

---

## Step 4: Configure `hdfs-site.xml`

### On NameNode:

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>

  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/data/hadoop/namenode</value>
  </property>
</configuration>
```
> replication should compatible with number of datanode for example if you have 2 datanode so your replication should be 2 

for web hdfs enable put it
```bash
<configuration>
  <!-- Keep your replication -->
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>

  <!-- Keep your NameNode metadata directory -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/data/hadoop/namenode</value>
  </property>

  <!-- Enable WebHDFS -->
  <property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>

  <!-- NameNode HTTP server (WebHDFS lives here) -->
  <property>
    <name>dfs.namenode.http-address</name>
    <value>master:9870</value>
  </property>

  <!-- (Recommended) Explicit NameNode RPC address for the default nameservice -->
  <property>
    <name>dfs.namenode.rpc-address</name>
    <value>master:8021</value>
  </property>
</configuration>
```

### On DataNode:

```xml
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/data/hadoop/datanode</value>
  </property>
</configuration>
```
for web hdfs 
```bash
<configuration>
  <!-- Keep your DataNode data directory -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/data/hadoop/datanode</value>
  </property>

  <!-- Enable WebHDFS -->
  <property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>

  <!-- DataNode service ports (these are the usual defaults, explicit is nice) -->
  <property>
    <name>dfs.datanode.address</name>
    <value>0.0.0.0:9866</value>
  </property>

  <property>
    <name>dfs.datanode.http.address</name>
    <value>0.0.0.0:9864</value>
  </property>

  <property>
    <name>dfs.datanode.ipc.address</name>
    <value>0.0.0.0:9867</value>
  </property>
</configuration>
```

Create directories and set permissions:

```bash
sudo mkdir -p /data/hadoop/namenode
sudo mkdir -p /data/hadoop/datanode
sudo chown -R $USER:$USER /data/hadoop/
```

---

## Step 5: Set `JAVA_HOME` in `hadoop-env.sh`

```bash
nano /opt/hadoop/etc/hadoop/hadoop-env.sh
```

Add:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

---

## Step 6: Format the NameNode (Only on NameNode)

```bash
hdfs namenode -format
```

---

## Step 7: Setup SSH and Workers List (Only on NameNode)

Edit:

```bash
nano /opt/hadoop/etc/hadoop/workers
```

Example:

```
worker1
worker2
worker3
```

Ensure `/etc/hosts` contains mappings for all nodes.

---

## Step 8: Start HDFS Services

On NameNode:

```bash
start-dfs.sh
```

On each DataNode (optional):

```bash
start-dfs.sh
```

Verify cluster:

```bash
hdfs dfsadmin -report
```

---

## Step 9: Client Node Setup (Optional)

Repeat installation and update `.bashrc`:

```bash
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
```

Update `core-site.xml` with:

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://<namenode-ip>:9000</value>
  </property>
</configuration>
```

---

## Step 10: HDFS File Operations

```bash
hdfs dfs -mkdir -p /user/hadoop/
hdfs dfs -ls /user
hdfs dfs -chown ali /user/hadoop
hdfs dfs -chmod 775 /user/hadoop

hdfs dfs -put healthcare_dataset.csv /user/hadoop/
hdfs fsck /user/hadoop/healthcare_dataset.csv -files -blocks -locations
```

---

## Verify Hadoop Version

```bash
hadoop version
```
