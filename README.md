# hadoop-docker
Docker images for Apache Hadoop

Build:

```
$ HADOOP_VERSION=3.2.0 OS=ubuntu bash build.sh
```

This creates multiple images, all based on the same Hadoop installation:

 - single-service images: datanode, namenode, nodemanager, resourcemanager,
   historyserver

 - all-in-one, pseudo-distributed image that runs all daemons (`hadoop`)

 - client image (just hangs indefinitely)

Usage example for the all-in-one image:

```
docker run --name hadoop -p 8020:8020 -p 9864:9864 -p 9870:9870 -p 8088:8088 -p 8042:8042 -p 19888:19888 -d crs4/hadoop
docker exec -it hadoop bash
hdfs dfs -mkdir -p "/user/$(whoami)"
hdfs dfs -put entrypoint.sh
export V=$(hadoop version | head -n 1 | awk '{print $2}')
hadoop jar /opt/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-${V}.jar wordcount entrypoint.sh wc_out
hdfs dfs -get wc_out
grep hadoop wc_out/part*
```

The docker-compose file shows how to set up an HDFS cluster with three
services (namenode, datanode, client).

```
docker-compose up
docker exec -it hadoop-docker_client_1 bash
hdfs dfs -mkdir -p "/user/$(whoami)"
[...]
```

## Changing the Hadoop version and/or the base image

```
$ HADOOP_VERSION=2.9.2 OS=alpine bash build.sh
```

Note that some ports are different in Hadoop 2:

```
docker run --name hadoop -p 8020:8020 -p 50075:50075 -p 50070:50070 -p 8088:8088 -p 8042:8042 -p 19888:19888 -d crs4/hadoop:2.9.2-ubuntu
```

## Custom configuration

Images come with a simple default configuration (in `/opt/hadoop/etc/hadoop`)
that should be good enough for a test cluster. If the `HADOOP_CUSTOM_CONF_DIR`
environment variable is set to the path of a directory visible by the
container, the entrypoint links any files found there into the Hadoop
configuration directory, forcing a replacement if necessary. For instance, to
use a custom `hdfs-site.xml`, you can save it to a `/tmp/hadoop` directory and
run the container as follows:

```
docker run ${PORT_FW} -v /tmp/hadoop:/hadoop_custom_conf -e HADOOP_CUSTOM_CONF_DIR=/hadoop_custom_conf -d crs4/hadoop
```

Another way to customize the configuration is to override the entire directory
with a bind mount. For instance, to change the HDFS block size:

```
# Copy the default config to /tmp/hadoop
docker run --rm --entrypoint /bin/bash crs4/hadoop -c "tar -c -C /opt/hadoop/etc hadoop" | tar -x -C /tmp
# Add a property
$ sed -i "s|</configuration>|<property><name>dfs.blocksize</name><value>$((256*2**20))</value></property></configuration>|" /tmp/hadoop/hdfs-site.xml
# Run the container, overriding the configuration directory
$ docker run ${PORT_FW} -v /tmp/hadoop:/opt/hadoop/etc/hadoop -d crs4/hadoop
```

## Automatic hostname setting

The entrypoint automatically replaces "localhost", if found in the value of
the `fs.defaultFS` property, with the running container's hostname (this
allows to contact the name node from outside the container). Since this
substitution happens before HADOOP_CUSTOM_CONF_DIR is processed, it has no
effect if `${HADOOP_CUSTOM_CONF_DIR}/core-site.xml` is provided. Bind mounts,
however, take effect before the entrypoint is executed, so use one or the
other configuration method depending on whether you want the automatic
hostname setting to take place or not. If the `NAMENODE_HOSTNAME` env var is
provided, "localhost" is replaced with its value instead. This can be useful
when composing single-service containers (see the sample docker-compose file).
