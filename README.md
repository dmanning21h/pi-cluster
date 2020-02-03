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
- Add all IPs and hostnames to bottom of file
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
- Add hostname, user, and IP address for each node in the network.
