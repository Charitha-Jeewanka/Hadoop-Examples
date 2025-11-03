# 🧠 Hadoop WordCount Example (Docker + Maven + IntelliJ)

This repository demonstrates how to build and run a **Java MapReduce WordCount** program on a **Hadoop 3.2.1** multi-container cluster using **Docker Compose**.
The project is built with **Maven** (from IntelliJ IDEA) and executed on the Hadoop cluster running Java 8.

---

## 📦 1. Cluster Setup (using Docker Compose)

Save the following as `docker-compose.yml`:

```yaml
version: "3.8"

services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8
    container_name: namenode
    restart: always
    ports:
      - "9870:9870"   # NameNode UI
      - "9000:9000"   # HDFS RPC
    volumes:
      - hadoop_namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop.env
    networks:
      - hadoop

  datanode1:
    image: bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8
    container_name: datanode1
    restart: always
    depends_on:
      - namenode
    volumes:
      - hadoop_datanode1:/hadoop/dfs/data
    environment:
      SERVICE_PRECONDITION: "namenode:9870"
    env_file:
      - ./hadoop.env
    networks:
      - hadoop

  datanode2:
    image: bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8
    container_name: datanode2
    restart: always
    depends_on:
      - namenode
    volumes:
      - hadoop_datanode2:/hadoop/dfs/data
    environment:
      SERVICE_PRECONDITION: "namenode:9870"
    env_file:
      - ./hadoop.env
    networks:
      - hadoop

  resourcemanager:
    image: bde2020/hadoop-resourcemanager:2.0.0-hadoop3.2.1-java8
    container_name: resourcemanager
    restart: always
    depends_on:
      - namenode
      - datanode1
      - datanode2
    ports:
      - "8088:8088"   # YARN UI
    environment:
      SERVICE_PRECONDITION: "namenode:9000 namenode:9870 datanode1:9864 datanode2:9864"
    env_file:
      - ./hadoop.env
    networks:
      - hadoop

  nodemanager1:
    image: bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8
    container_name: nodemanager1
    restart: always
    depends_on:
      - resourcemanager
    environment:
      SERVICE_PRECONDITION: "resourcemanager:8088"
    env_file:
      - ./hadoop.env
    networks:
      - hadoop

  nodemanager2:
    image: bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8
    container_name: nodemanager2
    restart: always
    depends_on:
      - resourcemanager
    environment:
      SERVICE_PRECONDITION: "resourcemanager:8088"
    env_file:
      - ./hadoop.env
    networks:
      - hadoop

volumes:
  hadoop_namenode:
  hadoop_datanode1:
  hadoop_datanode2:

networks:
  hadoop:
    name: hadoop
    driver: bridge
```

Add a `hadoop.env` file:

```bash
CORE_CONF_fs_defaultFS=hdfs://namenode:9000
HDFS_CONF_dfs_replication=2
YARN_CONF_yarn_resourcemanager_hostname=resourcemanager
```

### Start the cluster

```bash
docker compose up -d
```

Check containers:

```bash
docker ps
```

Web UIs:

* NameNode → [http://localhost:9870](http://localhost:9870)
* ResourceManager → [http://localhost:8088](http://localhost:8088)

---

## 🧰 2. Maven Project (in IntelliJ)

### Create Project

1. **New Project → Maven → Java 8 SDK (Temurin 1.8)**
2. Group ID = `com.charitha.hadoop`
3. Artifact ID = `stackoverflow-wc`

### `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.charitha.hadoop</groupId>
  <artifactId>stackoverflow-wc</artifactId>
  <version>1.0-SNAPSHOT</version>

  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <hadoop.version>3.2.1</hadoop.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-common</artifactId>
      <version>${hadoop.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-mapreduce-client-core</artifactId>
      <version>${hadoop.version}</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>

      <!-- Fat JAR -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.4</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
              <createDependencyReducedPom>false</createDependencyReducedPom>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>com.charitha.hadoop.WordCount</mainClass>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### WordCount.java

```java
package com.charitha.hadoop;

import java.io.IOException;
import java.util.StringTokenizer;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static class TokenizerMapper
      extends Mapper<Object, Text, Text, IntWritable> {

    private final static IntWritable one = new IntWritable(1);
    private final Text word = new Text();

    @Override
    public void map(Object key, Text value, Context context)
        throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        String token = itr.nextToken().replaceAll("[^A-Za-z0-9]", "").toLowerCase();
        if (!token.isEmpty()) {
          word.set(token);
          context.write(word, one);
        }
      }
    }
  }

  public static class IntSumReducer
      extends Reducer<Text, IntWritable, Text, IntWritable> {

    private final IntWritable result = new IntWritable();

    @Override
    public void reduce(Text key, Iterable<IntWritable> values, Context context)
        throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) sum += val.get();
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: WordCount <input> <output>");
      System.exit(2);
    }

    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

---

## ⚙️ 3. Build the JAR

From IntelliJ’s Maven panel or a terminal:

```bash
mvn clean package
```

Output:
`target/stackoverflow-wc-1.0-SNAPSHOT-shaded.jar`

---

## 🚀 4. Run on the Docker Hadoop Cluster

### Copy JAR to namenode

```bash
docker cp target/stackoverflow-wc-1.0-SNAPSHOT-shaded.jar namenode:/root/stackoverflow-wc.jar
```

### Prepare input in HDFS

```bash
docker exec -it namenode bash
echo "hello hadoop hello docker" > /tmp/input.txt
hdfs dfs -mkdir -p /user/root/input
hdfs dfs -put -f /tmp/input.txt /user/root/input
```

### Run MapReduce job

```bash
hdfs dfs -rm -r -f /user/root/output
hadoop jar /root/stackoverflow-wc.jar /user/root/input /user/root/output
```

### View results

```bash
hdfs dfs -cat /user/root/output/part-r-00000
```

Output example:

```
docker 1
hadoop 1
hello 2
```

---

## 🧩 5. Common Issues

| Problem                             | Cause                                            | Fix                                                                                     |
| ----------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| `Cannot resolve symbol 'hadoop'`    | Maven deps not imported                          | Add `<dependency>` for `hadoop-common` and `hadoop-mapreduce-client-core`, reload Maven |
| `Usage: WordCount <input> <output>` | Extra class arg passed                           | Omit class name when running (`hadoop jar jarfile input output`)                        |
| `Cannot initialize Cluster`         | Running plain `java -cp` instead of `hadoop jar` | Always run with `hadoop jar` inside the cluster                                         |
| Version mismatch errors             | Host compiled with JDK > 8                       | Use Temurin 1.8 SDK and target 1.8 bytecode                                             |

---

## 🖥️ 6. Cluster Management

| Action           | Command                                        |
| ---------------- | ---------------------------------------------- |
| Start cluster    | `docker compose up -d`                         |
| Stop cluster     | `docker compose down`                          |
| Check logs       | `docker logs namenode`                         |
| Enter container  | `docker exec -it namenode bash`                |
| Open NameNode UI | [http://localhost:9870](http://localhost:9870) |
| Open YARN UI     | [http://localhost:8088](http://localhost:8088) |

---

## 📚 7. References

* [Apache Hadoop MapReduce Tutorial](https://hadoop.apache.org/docs/r3.2.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)
* [BDE Hadoop Docker Images](https://github.com/big-data-europe/docker-hadoop)
* [Apache Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/)


