# Distributed Computing with the Raspberry Pi 4
Project to design a Raspberry Pi 4 Cluster using Spark for Distributed Machine Learning.

## Part 1: Hardware and Setup.
### 1. Hardware for my implementation:
 - (4) Raspberry Pi 4, 4GB Version
 - (4) 32GB MicroSD Card
 - (4) USB-C Power Supply
 - (4) 1ft Ethernet cable
 - (1) Raspberry Pi Cluster Case
 - (1) Gigabit Ethernet Switch
 - (1) Keyboard+Mouse combination
 - (1) HDMI to Micro-HDMI Cable
 - (1) HDMI Monitor
 
### 2. Preliminary Setup
 - Follow the Raspberry Pi Foundation's [Official Guide](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up/3) to install the Raspian OS.
 - After setting up one Raspberry Pi fully, [clone the SD card](https://beebom.com/how-clone-raspberry-pi-sd-card-windows-linux-macos/) to each of the others (after formatting each new Micro-SD).
 - Physically assemble cluster.
![My Setup](/pictures/setup.jpeg)

## Part 2: Passwordless SSH.
### 1. (Starting with Pi #1) Setup wireless internet connection.

### 2. Assign static IP address to the ethernet interface.
```console
pi@raspberrypi:~$ sudo mousepad /etc/dhcpcd.conf
```
- Uncomment: <br />
`interface eth0` <br />
`static ip_address=192.168.0.10X/24` <br />
- Where X is the respective Raspberry Pi # (e.g. 1, 2, 3, 4)

### 3. Enable SSH.
- From Raspberry Pi dropdown menu (Top left corner of desktop): Preferences -> Config -> Interfaces -> Enable SSH

### 4. Modify `/etc/hosts` to include hostnames and IPs of each Raspberry Pi node.
```console
pi@raspberrypi:~$ sudo mousepad /etc/hosts
```
- Change `raspberrypi` to `piX`
- Add all IPs and hostnames to bottom of file <br />
`192.168.0.101  pi1` <br />
`192.168.0.102  pi2` <br />
`192.168.0.103  pi3` <br />
`192.168.0.104  pi4` <br />

### 5. Change the Pi's hostname to be its respective Pi #.
```console
pi@raspberrypi:~$ sudo mousepad /etc/hostname
```
- Change `raspberrypi` to `piX`

### 6. Reboot node.
```console
pi@raspberrypi:~$ reboot
```
- Upon reopening the command prompt you should see the updated hostname.
```console
pi@piX:~$ 
```

### 7. (Only on Pi #1) Generate `ssh` config, then modify.
- The `ssh` config file is generated after the first time `ssh` is run, so just `ssh` into the current pi node.
```console
pi@pi1:~$ ssh pi1
```
- Then exit the shell.
```console
pi@pi1:~$ exit
```
- Now the file has been generated and can be modified.
```console
pi@pi1:~$ sudo mousepad ~/.ssh/config
```
- Add hostname, user, and IP address for each node in the network (repeated 4x in my case). <br />
`Host  piX` <br />
`User  pi` <br />
`Hostname 192.168.0.10X` <br />

### 8. Create authentication key pairs for `ssh`.
```console
pi@piX:~$ ssh-keygen -t ed25519
```

### 9. Repeat steps 1-8 on the rest of the Pis.
- Once completed up to creating authentication pairs, append each Pi's public key to pi1's `authorized_keys` file.
```console
pi@piX:~$ cat ~/.ssh/id_ed25519.pub >> | ssh pi@192.168.0.101 'cat >> .ssh/authorized_keys'
```

### 10. (Back on Pi1) Append this Pi's public key to the file as well.
```console
pi@pi1:~$ cat ~/.ssh/id_ed25519.pub >> .ssh/authorized_keys
```

### 11. Copy `authorized_keys` file and `ssh` configuration to all other nodes (2-4) in the cluster.
```console
pi@pi1:~$ scp ~/.ssh/authorized_keys piX:~/.ssh/authorized_keys
pi@pi1:~$ scp ~/.ssh/config piX:~/.ssh/config
```
- Now all devices in the cluster will be able to `ssh` into each other without requiring a password.

### 12. Create additional functions to improve the ease of use of the cluster.
- To do this, we will create some new shell functions within the `.bashrc` file.
```console
pi@pi1:~$ sudo mousepad ~/.bashrc
```
#### `otherpis`
- Add to the end of the file:
```shell
function otherpis {
  grep "pi" /etc/hosts | awk '{print $2}' | grep -v $(hostname)
}
```
- This function will find and print the hostnames of all other nodes in the cluster (be sure to source `.bashrc` before running). 
```console
pi@pi1:~$ source ~/.bashrc
pi@pi1:~$ otherpis
pi2
pi3
pi4
```
#### `clustercmd`
- This command will run the specified command on all other nodes in the cluster, and then itself.
- In `~/.bashrc`:
```shell
function clustercmd {
  for pi in $(otherpis); do ssh $pi "$@"; done
  $@
}
```
```console
pi@pi1:~$ source ~/.bashrc
pi@pi1:~$ clustercmd date
Sun 26 Jan 2020 12:56:58 PM EST
Sun 26 Jan 2020 12:56:58 PM EST
Sun 26 Jan 2020 12:56:58 PM EST
Sun 26 Jan 2020 12:56:58 PM EST
```


#### `clusterreboot` and `clustershutdown`
- Reboot and shutdown all nodes in the cluster.
```shell
function clusterreboot {
  clustercmd sudo shutdown -r now
}

function clustershutdown {
  clustercmd sudo shutdown now
}
```

#### `clusterscp`
- Copies files from one device to every other node in the cluster.
```shell
function clusterscp { 
  for pi in $(otherpis); do
    cat $1 | ssh $pi "sudo tee $1" > /dev/null 2>&1
  done
}
```
### 13. Copy `.bashrc` file to all other nodes in the network.
- First source it if you haven't already, then copy.
```console
pi@pi1:~$ source ~/.bashrc
pi@pi1:~$ clusterscp ~/.bashrc
```

## Part 3: Installing Hadoop.
### 1. Install Java 8 on each node, make this each node's default Java.
- The latest Raspian (Buster) comes with Java 11 pre-installed. However, the latest Hadoop version (3.2.1) that 
we will be using only supports Java 8. To resolve this issue we will install OpenJDK 8 and make this the default 
Java that will run on each device.
```console
pi@piX:~$ sudo apt-get install openjdk-8-jdk
pi@piX:~$ sudo update-alternatives --config java    // Select number corresponding to Java 8
pi@piX:~$ sudo update-alternatives --config javac   // Select number corresponding to Java 8
```

### 2. Download Hadoop, unpack, and give `pi` ownership.
```console
pi@pi1:~$ cd && wget https://www-us.apache.org/dist/hadoop/common/hadoop-3.2.1/hadoop-3.2.1.tar.gz
pi@pi1:~$ sudo tar -xvf hadoop-3.2.1.tar.gz -C /opt/
pi@pi1:~$ rm hadoop-3.2.1.tar.gz && cd /opt
pi@pi1:/opt$ sudo mv hadoop-3.2.1 hadoop
pi@pi1:/opt$ sudo chown pi:pi -R /opt/hadoop
```

### 3. Configure Hadoop Environment variables.
```console
pi@pi1:~$ sudo mousepad ~/.bashrc
```
- Add (insert at top of file):
```shell
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-armhf/
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export LD_LIBRARY_PATH=$HADOOP_HOME/lib/native:$LD_LIBRARY_PATH
```

### 4. Initialize `JAVA_HOME` for Hadoop environment.
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/hadoop-env.sh
```
```shell
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-armhf/
```

### 5. Validate Hadoop install.
```console
pi@pi1:~$ source ~/.bashrc
pi@pi1:~$ cd && hadoop version | grep Hadoop
Hadoop 3.2.1
```

## Part 4: Setting up Hadoop Cluster
### 1. Setup Hadoop Distributed File System (HDFS) configuration files (Single Node Setup to start).
- All of the following files are located within `/opt/hadoop/etc/hadoop`.

#### `core-site.xml`
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/core-site.xml
```
- Modify end of file to be:
```shell
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://pi1:9000</value>
  </property>
</configuration>
```

#### `hdfs-site.xml`
```shell
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///opt/hadoop_tmp/hdfs/datanode</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///opt/hadoop_tmp/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration> 
```

#### `mapred-site.xml`
```shell
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

#### `yarn-site.xml`
```shell
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.auxservices.mapreduce.shuffle.class</name>  
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
</configuration> 
```

### 2. Create Datanode and Namenode directories.
```console
pi@pi1:~$ sudo mkdir -p /opt/hadoop_tmp/hdfs/datanode
pi@pi1:~$ sudo mkdir -p /opt/hadoop_tmp/hdfs/namenode
pi@pi1:~$ sudo chown pi:pi -R /opt/hadoop_tmp
```

### 3. Format HDFS.
```console
pi@pi1:~$ hdfs namenode -format -force
```

### 4. Boot HDFS and verify functionality.
```console
pi@pi1:~$ start-dfs && start-yarn.sh
```
- Verify the setup by using the `jps` command.
```console
pi@pi1:~$ jps
```
- This command lists all of the Java processes running on the machine, of which there should be at least 6:
1. `NameNode`
2. `DataNode`
3. `NodeManager`
4. `ResourceManager`
5. `SecondaryNameNode`
6. `jps`

- Create temporary directory to test the file system:
```console
pi@pi1:~$ hadoop fs -mkdir /tmp
pi@pi1:~$ hadoop fs -ls /
```

- Stop the single node cluster using:
```console
pi@pi1:~$ stop-dfs && stop-yarn.sh
```

### 5. Silence Warnings (as a result of 32-bit Hadoop build w/ 64-bit OS)
- Modify Hadoop environment configuration:
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/hadoop-env.sh
```
- Change:
```shell
# export HADOOP_OPTS="-Djava.net.preferIPv4Stack=true"
```
- To:
```shell
export HADOOP_OPTS="-XX:-PrintWarnings –Djava.net.preferIPv4Stack=true"
```

- Now in the `~/.bashrc`, add to the bottom:
```shell
export HADOOP_HOME_WARN_SUPPRESS=1
export HADOOP_ROOT_LOGGER="WARN,DRFA" 
```

- Source `~/.bashrc`:
```console
pi@pi1:~$ source ~/.bashrc
```

- Copy `.bashrc` to other nodes in the cluster:
```console
pi@pi1:~$ clusterscp ~/.bashrc
```

### 6. Create Hadoop Cluster directories (Multi-node Setup).
```console
pi@pi1:~$ clustercmd sudo mkdir -p /opt/hadoop_tmp/hdfs
pi@pi1:~$ clustercmd sudo chown pi:pi –R /opt/hadoop_tmp
pi@pi1:~$ clustercmd sudo mkdir -p /opt/hadoop
pi@pi1:~$ clustercmd sudo chown pi:pi /opt/hadoop
```

### 7. Copy Hadoop files to the other nodes.
```console
pi@pi1:~$ for pi in $(otherpis); do rsync -avxP $HADOOP_HOME $pi:/opt; done
```
-Verify install on other nodes:
```console
pi@pi1:~$ clustercmd hadoop version | grep Hadoop
Hadoop 3.2.1
Hadoop 3.2.1
Hadoop 3.2.1
Hadoop 3.2.1
```

### 8. Modify Hadoop configuration files for cluster setup.
#### `core-site.xml`
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/core-site.xml
```
```shell
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://pi1:9000</value>
  </property>
</configuration>
```

#### `hdfs-site.xml`
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/hdfs-site.xml
```
```shell
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/opt/hadoop_tmp/hdfs/datanode</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/opt/hadoop_tmp/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>4</value>
  </property>
</configuration> 
```

#### `mapred-site.xml`
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/mapred-site.xml
```
```shell
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
      <value>yarn</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
      <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>mapreduce.map.env</name>
      <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>mapreduce.reduce.env</name>
      <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
      <value>512</value>
  </property>
  <property>
    <name>mapreduce.map.memory.mb</name>
      <value>256</value>
  </property>
  <property>
    <name>mapreduce.reduce.memory.mb</name>
      <value>256</value>
  </property>
</configuration>
```

#### `yarn-site.xml`
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/yarn-site.xml
```
```shell
<configuration>
  <property>
    <name>yarn.acl.enable</name>
    <value>0</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname</name>
      <value>pi1</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
      <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
      <value>1536</value>
  </property>
  <property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
      <value>1536</value>
  </property>
  <property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
      <value>128</value>
  </property>
  <property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
      <value>false</value>
  </property>
</configuration> 
```

### 9. Clean `datanode` and `namenode` directories.
```console
pi@pi1:~$ clustercmd rm -rf /opt/hadoop_tmp/hdfs/datanode/*
pi@pi1:~$ clustercmd rm -rf /opt/hadoop_tmp/hdfs/namenode/*
```

### 10. Create/edit `master` amd `worker` files.
```console
pi@pi1:~$ cd $HADOOP_HOME/etc/hadoop
pi@pi1:/opt/hadoop/etc/hadoop$ mousepad master
```
- Add single line to file:
```shell
pi1
```
```console
pi@pi1:/opt/hadoop/etc/hadoop$ mousepad workers
```
- Add other pi hostnames to the file:
```shell
pi2
pi3
pi4
```

### 11. Edit `hosts` file.
```console
pi@pi1:~$ sudo mousepad /etc/hosts
```
- Remove the line (All nodes will have identical host configuration):
```shell
127.0.1.1 pi1
```
- Copy updated file to the other cluster nodes:
```console
pi@pi1:~$ clusterscp /etc/hosts
```

- Now reboot the cluster:
```console
pi@pi1:~$ clusterreboot
```

### 12. Format and start multi-node cluster.
```console
pi@pi1:~$ hdfs namenode -format -force
pi@pi1:~$ start-dfs.sh && start-yarn.sh
```
- Now since we have configured Hadoop on a multi-node cluster, when we use `jps` on the master node (pi1), only the following processes will be running:
1. `Namenode`
2. `SecondaryNamenode`
3. `ResourceManager`
4. `jps`
- With the following having been offloaded to the datanodes, as you'll see if you `ssh` into and perform a `jps`:
1. `Datanode`
2. `NodeManager`
3. `jps`

### 13. Modify `clusterreboot` and `clustershutdown` to shutdown Hadoop cluster gently.
#### `clusterreboot`
```shell
function clusterreboot {
  stop-yarn.sh && stop-dfs.sh && \
  clustercmd sudo shutdown -r now
}
```
#### `clustershutdown`
```shell
function clustershutdown {
  stop-yarn.sh && stop-dfs.sh && \
  clustercmd sudo shutdown now
}
```

## Part 5: Testing the Hadoop Cluster (Wordcount Example)
### 1. Start cluster, if not active already.
```console
pi@pi1:~$ start-hdfs.sh && start-yarn.sh
```

### 2. Make data directories.
- To test the Hadoop cluster, we will deploy a sample wordcount job to count word frequencies from several books obtained from the [Gutenberg Project](https://www.gutenberg.org/).
- First, make the HDFS directories for the data.
```console
pi@pi1:~$ hdfs dfs -mkdir -p /user/pi
pi@pi1:~$ hdfs dfs -mkdir books
```

### 3. Download books files.
```console
pi@pi1:~$ cd /opt/hadoop
pi@pi1:/opt/hadoop$ wget -O alice.txt https://www.gutenberg/org/files/11/11-0.txt
pi@pi1:/opt/hadoop$ wget -O holmes.txt https://www.gutenberg/org/files/1661/1661-0.txt
pi@pi1:/opt/hadoop$ wget -O frankenstein.txt https://www.gutenberg/org/files/84/84-0.txt
```

### 4. Upload book files to the HDFS.
```console
pi@pi1:/opt/hadoop$ hdfs dfs -put alice.txt holmes.txt frankenstein.txt books
pi@pi1:/opt/hadoop$ hdfs dfs -ls books
```

### 5. Read one of the books from the HDFS.
```console
pi@pi1:/opt/hadoop$ hdfs dfs -cat books/alice.txt
```

### 6. Monitor status of cluster and jobs.
- You can monitor the status of all jobs deployed to the cluster via the YARN web UI: http://pi1:8088
![Yarn Web UI](/pictures/yarn-ui.png)

- And the status of the cluster in general via the HDFS web UI: http://pi1:9870
![HDFS Web UI](/pictures/hdfs-ui.png)

### 7. Deploy sample MapReduce job to cluster.
```console
pi@pi1:/opt/hadoop$ yarn jar /opt/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar wordcounts "books/*" output
```

### 8. View output of job.
```console
pi@pi1:/opt/hadoop$ hdfs dfs -ls output
pi@pi1:/opt/hadoop$ hdfs dfs -cat output/part-r-0000 | less
```
![Hadoop Test](/pictures/hadoop-test.png)

## Part 6: Install Spark on the Cluster.
### 1. Download Spark, unpack, and give `pi` ownership.
```console
pi@pi1:~$ cd && wget https://www-us.apache.org/dist/spark/spark-2.4.4/spark-2.4.4-bin-hadoop-2.7.tgz
pi@pi1:~$ sudo tar -xvf spark-2.4.4-bin-hadoop-2.7.tgz -C /opt/
pi@pi1:~$ rm spark-2.4.4-bin-hadoop-2.7.tgz && cd /opt
pi@pi1:~$ sudo mv spark-2.4.4-bin-hadoop-2.7 spark
pi@pi1:~$ sudo chown pi:pi -R /opt/spark
```

### 2. Configure Spark Environment variables.
```console
pi@pi1:~$ sudo mousepad ~/.bashrc
```
- Add (insert at top of file):
```shell
export SPARK_HOME=/opt/spark
export PATH=$PATH:$SPARK_HOME/bin
```

### 3. Validate Spark install.
```console
pi@pi1:~$ source ~/.bashrc
pi@pi1:~$ spark-shell --version
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.4
      /_/
                       
Using Scala version 2.11.12, OpenJDK Client VM, 1.8.0_212
Branch
Compiled by user  on 2019-08-27T21:21:38Z
Revision
Url
Type --help for more information.
```

## Part 7: Test Spark on the Cluster (Approximating Pi).
### 1. Configure Spark job monitoring.
- Similar to Hadoop, Spark also offers the ability to monitor the jobs you deploy. However, with Spark we will have to manually configure the monitoring options.
- Generate then modify the Spark default configuration file:
```console
pi@pi1:~$ cd $SPARK_HOME/conf
pi@pi1:/opt/spark/conf$ sudo mv spark-defaults.conf.template spark-defaults.conf
pi@pi1:/opt/spark/conf$ mousepad spark-defaults.conf
```
- Add the following lines:
```shell
spark.master                       yarn
spark.driver.memory                512m
spark.yarn.am.memory               512m
spark.executor.memory              512m
spark.executor.cores               4

spark.eventLog.enabled             true
spark.eventLog.dir                 hdfs://pi1:9000/spark-logs
spark.history.provider             org.apache.spark.deploy.history.FsHistoryProvider
spark.history.fs.logDirectory      hdfs://pi1:9000/spark-logs
spark.history.fs.update.interval   10s
spark.history.ui.port              18080
```

### 2. Make logging directory on HDFS.
```console
pi@pi1:/opt/spark/conf$ cd
pi@pi1:~$ hdfs dfs -mkdir /spark-logs
```

### 3. Start Spark history server.
```console
pi@pi1:~$ $SPARK_HOME/sbin/start-history-server.sh
```
- The Spark history server UI can be accessed at: http://pi1:18080
![Spark History UI](/pictures/spark-history-ui.png)

### 4. Run sample job.
```console
pi@pi1:~$ spark-submit --deploy-mode client --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples_2.11-2.4.4.jar 7
```
![Spark Test](/pictures/spark-test.png)

## Part 8: Acquiring [Sloan Digital Sky Survey (SDSS)](https://www.sdss.org/) Data.
- The data I will be using to train and test a machine learning classifier is from the Sloan Digital Sky Survey (Data Release 16), a major multi-spectral imaging and spectroscopic redshift survey of different galactic masses (Stars, Galaxies, Quasars).
<br />
![SDSS Telescope](/pictures/sdss_gaulme1.jpg)
<br />
- 


## Part 9: Installing Python Packages and Jupyter Notebook on Master node.



## Part 10: Stars, Galaxies, and Quasars: Using PySpark to Classify SDSS Data.


