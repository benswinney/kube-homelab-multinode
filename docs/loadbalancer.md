# A Load Balancer for the Kubernetes API-Server

The Kubernetes API-Server should sit behind a load balancer in a round-robin method.

## Configure HAProxy and Heartbeat

On each Proxy node, run the following:

> proxy01

```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy heartbeat -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

> proxy02

```shell
sudo apt update && sudo apt upgrade -y && sudo apt install haproxy heartbeat -y

sudo mv /etc/haproxy/haproxy.cfg{,.bkp}
```

Add the below line to the `/etc/systctl.conf` file on each Proxy node

```shell
net.ipv4.ip_nonlocal_bind=1
```

Create a new `/etc/haproxy/haproxy.cfg` on each Proxy node:

```shell
global
    user haproxy
    group haproxy

defaults
    mode http
    log global
    retries 2
    timeout connect 3000ms
    timeout server 5000ms
    timeout client 5000ms

frontend kubernetes
    bind vip:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server kube-master-0 master01:6443 check fall 3 rise 2
    server kube-master-1 master02:6443 check fall 3 rise 2
    server kube-master-2 master03:6443 check fall 3 rise 2
```

Enable and Start HAProxy & Heartbeat on both Proxy nodes

```shell
sudo systemctl enable haproxy && sudo systemctl start haproxy
sudo systemctl enable heartbeat && sudo systemctl start heartbeat
```

REBOOT each Proxy node

Upon a successful reboot, check to see the that haproxy has started and is listening on the 192.168.1.49 address

```shell
sudo netstat -ntulp
```

Create a authkeys `/etc/ha.d/authkeys` file on both Proxy nodes, ensuring it's readable/writable by root only (`chmod 600 /etc/ha.d/authkeys`).

```shell
sudo touch /etc/ha.d/authkeys && sudo chmod 600 /etc/ha.d/authkeys
```

Firstly generate a md5sum password

```shell
echo -n somepassword | md5sum
9c42a1346e333a770904b2a2b37fa7d3
```

Then add to the `/etc/ha.d/authkeys` file

```shell
auth 1
1 md5 9c42a1346e333a770904b2a2b37fa7d3
```

Create `/etc/ha.d/ha.cf` on each Proxy node

They will differ slightly, as can be seen below

> proxy01

```shell
#       keepalive: how many seconds between heartbeats
#
keepalive 2
#
#       deadtime: seconds-to-declare-host-dead
#
deadtime 10
#
#       What UDP port to use for udp or ppp-udp communication?
#
udpport 694
bcast ens18
mcast ens18 225.255.255.0 694 1 0
ucast ens18 192.168.1.56
#
#       What interfaces to heartbeat over?
udp ens18
#
#       Facility to use for syslog()/logger (alternative to log/debugfile)
#
logfacility local0
#
#       Tell what machines are in the cluster
#       node    nodename ...    -- must match uname -n
node proxy01
node proxy02
```

> proxy02

```shell
#       keepalive: how many seconds between heartbeats
#
keepalive 2
#
#       deadtime: seconds-to-declare-host-dead
#
deadtime 10
#
#       What UDP port to use for udp or ppp-udp communication?
#
udpport 694
bcast ens18
mcast ens18 225.255.255.0 694 1 0
ucast ens18 192.168.1.57
#
#       What interfaces to heartbeat over?
udp ens18
#
#       Facility to use for syslog()/logger (alternative to log/debugfile)
#
logfacility local0
#
#       Tell what machines are in the cluster
#       node    nodename ...    -- must match uname -n
node proxy01
node proxy02
```

Create a `/etc/ha.d/haresources` file on both Proxy nodes

```shell
proxy01 192.168.1.49
```

Restart the heartbeat service on both Proxy nodes

```shell
sudo systemctl restart heartbeat
```
