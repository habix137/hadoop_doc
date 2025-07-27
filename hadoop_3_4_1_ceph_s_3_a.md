# Hadoop 3.4.1 ↔ Ceph RGW (S3A) Setup Guide

> **Goal:** Make Hadoop and Spark treat a Ceph object‑storage bucket as a normal Hadoop filesystem using the `s3a://` scheme.

---

## 1  Prerequisites

| Component     | Version / Notes                                              |
| ------------- | ------------------------------------------------------------ |
| **Hadoop**    | 3.4.1 binary tarball, unpacked in `/opt/hadoop`              |
| **Java LTS**  | OpenJDK 17 (or 11) – set `JAVA_HOME` accordingly             |
| **Ceph RGW**  | Reef 18.2.x, endpoint `172.27.8.77:80`, bucket `sparkbucket` |
| **Ceph user** | `sparkuser` – access `OUET4GER69OVXEN91ZMM`, secret `71X…`   |

---

## 2  Required client JARs (copy to `$HADOOP_HOME/share/hadoop/tools/lib/`)

```bash
wget -P /opt/hadoop/share/hadoop/tools/lib/ \
  https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.4.1/hadoop-aws-3.4.1.jar \
  https://repo1.maven.org/maven2/software/amazon/awssdk/bundle/2.25.60/bundle-2.25.60.jar
```

*Exactly one* `hadoop-aws‑3.4.1.jar` and one `bundle‑2.25.60.jar` must be on the classpath.  Remove older SDK v1 bundles if present.

---

## 3  Hadoop environment tweaks (edit `hadoop-env.sh`)

```bash
# use OpenJDK 17
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

# load every jar we drop in tools/lib
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/opt/hadoop/share/hadoop/tools/lib/*

# bypass any corporate proxy for RGW traffic
export no_proxy=172.27.8.77
```

*(Add the same exports to **``** if you run Hadoop commands manually.)*

---

## 4  `core-site.xml`

Place this file in `` on every node:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- Ceph RGW endpoint -->
  <property>
    <name>fs.s3a.endpoint</name>
    <value>http://172.27.8.77:80</value>
  </property>

  <!-- Plain HTTP -->
  <property>
    <name>fs.s3a.connection.ssl.enabled</name>
    <value>false</value>
  </property>

  <!-- Ceph needs path‑style URLs -->
  <property>
    <name>fs.s3a.path.style.access</name>
    <value>true</value>
  </property>

  <!-- Non‑empty region avoids 301 redirects -->
  <property>
    <name>fs.s3a.endpoint.region</name>
    <value>us-east-1</value>
  </property>

  <!-- Credentials embedded for test; use JCEKS in prod -->
  <property>
    <name>fs.s3a.aws.credentials.provider</name>
    <value>org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider</value>
  </property>
  <property>
    <name>fs.s3a.access.key</name>
    <value>OUET4GER69OVXEN91ZMM</value>
  </property>
  <property>
    <name>fs.s3a.secret.key</name>
    <value>71Xsy8IgPePr50DK1yOQ42Eiy9af5d9LJyRmVdda</value>
  </property>

  <!-- Time‑outs (must be integers w/ SDK v2) -->
  <property><name>fs.s3a.connection.request.timeout</name><value>60000</value></property>
  <property><name>fs.s3a.connection.establish.timeout</name><value>60000</value></property>
  <property><name>fs.s3a.connection.timeout</name><value>60000</value></property>
  <property><name>fs.s3a.socket.timeout</name><value>120000</value></property>

  <!-- Retry policy -->
  <property><name>fs.s3a.attempts.maximum</name><value>3</value></property>
  <property><name>fs.s3a.retry.limit</name><value>3</value></property>
</configuration>
```

> **Note:** underscores are illegal in URIs, so we use the raw IP. If you need a hostname, register one without `_`.

---

## 5  Service restarts

```bash
# reload env vars in your shell
exec bash

# or restart daemons on each node
a) sbin/stop-dfs.sh  # if already running
b) sbin/start-dfs.sh
```

“Unable to load native-hadoop” and “Cannot set priority …” warnings are harmless.

---

## 6  Smoke test

```bash
hadoop fs -ls s3a://sparkbucket/
# ➜ Found 0 items   (or list of objects)
```

### Upload & verify

```bash
echo "Hello Ceph" > /tmp/hello.txt
hadoop fs -mkdir -p s3a://sparkbucket/test/
hadoop fs -put /tmp/hello.txt s3a://sparkbucket/test/
hadoop fs -cat s3a://sparkbucket/test/hello.txt
```

---

## 7  Troubleshooting cheatsheet

| Symptom                                 | Fix                                                                    |
| --------------------------------------- | ---------------------------------------------------------------------- |
| `ClassNotFoundException: S3AFileSystem` | Missing `hadoop-aws.jar` or SDK bundle.                                |
| `NumberFormatException: "60s"`          | A timeout property still contains a suffix; use pure integers.         |
| `URISyntaxException`                    | Endpoint must be `http://host:port` with no underscores.               |
| `503 Service Unavailable` via curl      | RGW rejects Host header – use correct hostname, or IP; bypass proxies. |
| `AccessDenied`                          | Wrong key/secret or bucket policy; verify with `s3cmd`.                |

---

## 8  Spark integration (Spark 4.0.0 preview, Scala 2.13)

### 8.1  Copy the two required jars to every worker

```
cp /opt/hadoop/share/hadoop/tools/lib/hadoop-aws-3.4.1.jar \
   /opt/spark/jars/
cp /opt/hadoop/share/hadoop/tools/lib/bundle-2.25.60.jar \
   /opt/spark/jars/
```

### 8.2  Remove *all* conflicting AWS SDK jars

```bash
cd /opt/spark/jars
sudo rm aws-java-sdk-bundle-1.12.*.jar              # old v1 bundle
sudo rm *-2.27.0.jar                                # modular v2 jars shipped with Spark preview
```

(Listed in full: `auth, aws-core, cloudwatch, http-client-spi, identity-spi, netty-nio-client, profiles, regions, s3, s3-transfer-manager, sdk-core, third-party-jackson-core, utils`.)

### 8.3  Fix the Jackson‑Scala / Scala version mismatch

Spark 4.0.0 uses **Scala 2.13**; ensure the matching Jackson jar is active:

```bash
cd /opt/spark/jars
mv jackson-module-scala_2.12-*.jar jackson-module-scala_2.12-*.jar.old   # deactivate 2.12 build
mv jackson-module-scala_2.13-*.jar.notuse jackson-module-scala_2.13-*.jar
```

### 8.4  `spark-defaults.conf`

Put S3A properties (no `spark.jars` line is necessary **if** the two jars live under `$SPARK_HOME/jars/`).  Either:

- **Preferred:** leave the jars in the `jars/` directory **and delete** any existing

  ```
  spark.jars ...
  ```

  line.  Spark automatically picks up every jar in that folder.

- **Alternate:** if you keep the jars elsewhere, keep a single `spark.jars` line **that lists only** the two good jars, e.g.

  ```properties
  spark.jars /opt/spark/jars/hadoop-aws-3.4.1.jar,/opt/spark/jars/bundle-2.25.60.jar
  ```

  and make sure no conflicting AWS jars appear later on the class‑path.

Minimal working example (preferred):

```properties
spark.hadoop.fs.s3a.endpoint                 http://172.27.8.77:80
spark.hadoop.fs.s3a.path.style.access        true
spark.hadoop.fs.s3a.connection.ssl.enabled   false
spark.hadoop.fs.s3a.endpoint.region          us-east-1
spark.hadoop.fs.s3a.aws.credentials.provider org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider
spark.hadoop.fs.s3a.access.key               OUET4GER69OVXEN91ZMM
spark.hadoop.fs.s3a.secret.key               71Xsy8IgPePr50DK1yOQ42Eiy9af5d9LJyRmVdda
```

### 8.5  Restart Spark  Restart Spark

```bash
/opt/spark/sbin/stop-all.sh
/opt/spark/sbin/start-all.sh
```

### 8.6  Smoke test

```bash
/opt/spark/bin/spark-submit --deploy-mode client read_head5.py
```

Should print first five rows without `NoClassDefFoundError`.

---

## 9  Production hardening (next steps)

- Store keys in a **JCEKS credential provider** (`hadoop credential create …`).

- Enable **HTTPS** (terminating TLS at a reverse proxy or directly in RGW).

- Add **native libraries** (`libhadoop.so`, compression codecs) for better I/O performance.

- Tune retries/time‑outs and consider enabling the **AWS CRT S3 client** (Hadoop 3.4+).   Production hardening (next steps)

- Store keys in a **JCEKS credential provider** (`hadoop credential create …`).

- Enable **HTTPS** (terminating TLS at a reverse proxy or directly in RGW).

- Add **native libraries** (`libhadoop.so`, compression codecs) for better I/O performance.

- Tune retries/time‑outs and consider enabling the **AWS CRT S3 client** (Hadoop 3.4+).

