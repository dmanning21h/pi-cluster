# Distributed Computing with the Raspberry Pi 4
Project to design a Raspberry Pi 4 Cluster using Spark for Distributed Machine Learning.

## Part 1: Setting up Passwordless SSH.

### 1. Setup wireless internet connection.

### 2. Assign static IP address to the ethernet interface.
```console
pi@raspberrypi:~$ sudo mousepad /etc/dhcpcd.conf
```
- Uncomment: <br />
`interface eth0` <br />
`static ip_address=192.168.0.10X/24` <br />
- Where X is the respective Raspberry Pi # (e.g. 1, 2, 3, 4)

### 3. Enable SSH.
- From Raspberry Pi dropdown menu: Preferences -> Config -> Interfaces -> Enable SSH

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
- Upon reopening the command prompt you should see the updated Pi hostname.
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
pi@piX:~$ cat ~/.ssh/id_ed25519.pub >> | ssh pi @ 192.168.0.101 'cat >> .ssh/authorized_keys'
```

### 10. (Back on Pi1) Append this Pi's public key to the file as well.
```console
pi@piX:~$ cat ~/.ssh/id_ed25519.pub >> .ssh/authorized_keys
```

### 11. Copy `authorized_keys` file and `ssh` configuration to all other nodes in the cluster.
```console
pi@piX:~$ scp ~/.ssh/authorized_keys piX:~/.ssh/authorized_keys
pi@piX:~$ scp ~/.ssh/config piX:~/.ssh/config
```
- Now all devices in the cluster will be able to `ssh` into each other without requiring a password.

### 12. Create additional functions to improve the ease of use of the cluster.
- To do this, we will create some new shell functions within the `.bashrc` file.
```console
pi@piX:~$ sudo mousepad ~/.bashrc
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
pi@pi1:~$ clustercmd data
```
<!-- PUT CLUSTERCMD DATE OUTPUT HERE -->

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

## Part 2: Installing Hadoop.

### 1. Install Java 8 on each node, make this each nodes default Java.
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
<!-- GET EXACT JAVA 8 PATH FOR MY CONFIGURATION -->
- Add:
```shell
export JAVA_HOME=?????????????
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

### 4. Initialize `JAVA_HOME` for Hadoop environment.
```console
pi@pi1:~$ sudo mousepad /opt/hadoop/etc/hadoop/hadoop-env.sh
```
```shell
export JAVA_HOME=??????????????
```

### 5. Validate Hadoop install.
```console
pi@pi1:~$ source ~/.bashrc
pi@pi1:~$ cd && hadoop version | grep Hadoop
Hadoop 3.2.1
```

## Part 3: Setting up and Testing Hadoop Cluster
