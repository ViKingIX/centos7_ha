# CentOS 7 HA (corosync + pacemaker)

### Architecture
 - Router: IBM X3550 M3 * 2 (r1, r2)
 - Host: HP DL360p gen8 * 1


### Prerequisite
#### Add host resolution
all# `echo -e "$IP_r1 router-01\n$IP_r2 router-02" >> /etc/hosts`

#### Setup SSH passwordless login
##### Generate SSH keys
all# `ssh-keygen -N ''`

##### Install sshpass
all# `yum --enablerepo=epel -y install sshpass`

#### set passwordless login
r1# `sshpass -p $PASSWD ssh-copy-id router-02`\
r2# `sshpass -p $PASSWD ssh-copy-id router-01`

#### Allow incoming connection from peer
all# `iptables -I INPUT 1 -i $NIC_UP -s $IP_network -j ACCEPT`


### Install pacakges
all# `yum -y install pcs pacemaker resource-agents fence-agents`


### Configure the cluster
#### enable services
all# `systemctl enable pcsd.service corosync.service pacemaker.service`
#### set password for pcs user
all# `echo $PASSWD | passwd --stdin hacluster`
#### authenticate pcs user
any# `pcs cluster auth -u hacluster -p $PASSWD router-01 router-02`
#### setup cluster
any# `pcs cluster setup --name router router-01 router-02`
#### disable stonith
any# `pcs property set stonith-enabled=false`
#### disable quorum
any# `pcs property set no-quorum-policy=ignore`


### Verify cluster
#### start cluster
any# `pcs cluster start --all`
#### show cluster status
any# `pcs cluster status`


### Add resources
#### Add vip for uplink
any# `pcs resource create vip_up ocf:heartbeat:IPaddr2 ip=$IP_up cidr_netmask=24 nic=$NIC_up`

#### Add vip for downlink
any# `pcs resource create vip_down ocf:heartbeat:IPaddr2 ip=$IP_down cidr_netmask=24 nic=$NIC_down`

#### Group the vips (to make the resources run on the same machine)
any# `pcs resource group add grp_vip vip_up vip_down`

#### Verify the resources status
any# `pcs status`
