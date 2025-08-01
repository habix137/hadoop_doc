# Hadoop Installation Guide (with NameNode and DataNode Roles)

## Step 1: Download and Extract Hadoop

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
tar xvf hadoop-3.4.1.tar.gz
mv hadoop-3.4.1 /opt/hadoop
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

### On DataNode:

```xml
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/data/hadoop/datanode</value>
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
