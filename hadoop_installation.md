# Hadoop Installation Guide

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

## Step 3: Configure `core-site.xml`

```bash
cd /opt/hadoop/etc/hadoop
nano core-site.xml
```

Insert the following inside `<configuration>`:

```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://<master_ip>:9000</value>
</property>
```

## Step 4: Configure `hdfs-site.xml`

Create directories and set permissions:

```bash
sudo mkdir -p /data/hadoop/namenode
sudo mkdir -p /data/hadoop/datanode
sudo chown konect:konect /data/hadoop/namenode
sudo chown konect:konect /data/hadoop/datanode
```

Edit file:

```bash
nano /opt/hadoop/etc/hadoop/hdfs-site.xml
```

Insert:

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/opt/hadoop/hdfs/datanode</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/opt/hadoop/hdfs/namenode</value>
  </property>
</configuration>
```

## Step 5: Set `JAVA_HOME` in `hadoop-env.sh`

Edit file:

```bash
nano /opt/hadoop/etc/hadoop/hadoop-env.sh
```

Add:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_SSH_OPTS="-i ~/.ssh/spark -o StrictHostKeyChecking=no"
```

## Step 6: Format the Namenode

```bash
/opt/hadoop/bin/hdfs namenode -format
```

## Step 7: Setup SSH and Workers List

Add worker hostnames to:

```bash
nano /opt/hadoop/etc/hadoop/workers
```

Example content:

```
worker1
worker2
worker3
```

Ensure `/etc/hosts` contains the worker IPs and hostnames.

## Step 8: Start HDFS Services

On master:

```bash
/opt/hadoop/sbin/start-dfs.sh
```

On each worker (optional):

```bash
/opt/hadoop/sbin/start-dfs.sh
```

Verify status:

```bash
/opt/hadoop/bin/hdfs dfsadmin -report
```

## Step 9: Setup Hadoop on Client Node

Repeat Hadoop installation steps. Then:

```bash
nano ~/.bashrc
```

Add:

```bash
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
```

Edit `core-site.xml`:

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://<NameNode-IP>:9000</value>
  </property>
</configuration>
```

## Step 10: HDFS File Operations

Create directory:

```bash
hdfs dfs -mkdir -p /user/hadoop/
```

Check ownership:

```bash
hdfs dfs -ls /user
```

Change ownership:

```bash
hdfs dfs -chown ali /user/hadoop
hdfs dfs -chmod 775 /user/hadoop
```

Upload file:

```bash
hdfs dfs -put healthcare_dataset.csv /user/hadoop/
```

Check file replication:

```bash
hdfs fsck /user/hadoop/healthcare_dataset.csv -files -blocks -locations
```

## Verify Hadoop Version

```bash
hadoop version
```

