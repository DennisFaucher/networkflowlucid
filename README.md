# Discovering and graphing network flows with ntop, MariaDB, LucidChart

## Tool Comparison and LucidChart Creation Video Tutorial
[![Discovery Tool Comparison](http://img.youtube.com/vi/rQYM_lNA2Ak/0.jpg)](http://www.youtube.com/watch?v=rQYM_lNA2Ak)


# Why

![Why](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/why.png)

Mapping the network flows between servers, VMs, and applications can be very useful for application dependancy mapping, data center migration, disaster recovery planning, and network congestion. My learning journey took me to for-fee and for-free products including VMware [VRNI](https://www.vmware.com/products/vrealize-network-insight.html), [Dynatrace](https://www.dynatrace.com/), [ntop](https://www.ntop.org/), [MariaDB](https://mariadb.org/), and [LucidChart](https://lucid.app/). If you have the IT and financial resources, VRNI and Dynatrace are excellent. If you do not, ntop, MariaDB, and LucidChart are a good start.

# How

![How](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/How25.jpeg)

## Capture the Network Flows

There are many tools to capture network packets. I found ntop the easiest to use and ntop has database-writing built in. If you want ntop to see all packets and not just the packets destined fo the host ntop is running on, you need to first place your network switch in Promiscous Mode. I placed the Standard vSwitch in my two ESXI hosts in Promiscous Mode with these steps:

* Select a host in the vSphere Web Client
* Select the Configure tab
* Select Virtual Switches
* Select Edit for a standard vSwitch
* Select the Security section
* Change all settings to Accept
* Select OK

![vSwitch](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/vSwitch.png)

![Security Settings](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/vSwitch%20Security%20Settings.png)

Now that any host can see all packets, it's time to install ntop(ng) on one of the hosts and have ntopng write to MariaDB. I chose one of my Ubuntu VMs. The steps are pretty simple:

#### Install MariaDB

````sh
sudo apt install mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb (check for any issues and fix)
sudo systemctl enable mariadb
````

#### Install ntop(ng)

````sh
sudo apt install ntopng
sudo vi /etc/ntopng.conf
    (Change -i to match your Ethernet device.)
    -i=ens192
sudo systemctl start ntopng
sudo systemctl status ntopng (check for any issues and fix)
sudo systemctl enable ntopng
````

#### Write the Flows to MariaDB

````bash
sudo vi /etc/ntopng.conf
    (Add the two MySQL(MariaDB) lines changing the IP address to your MariaDB IP address)
    # Dump flows to MySQL
    --dump-flows=mysql;192.168.1.65;ntopng;flows;ntopng;ntopng
sudo systemctl restart ntopng
sudo systemctl status ntopng (check for any issues and fix)
````
ntopng creates the flows database and the flowsv4 & flowsv6 tables in MariaDB on startup. You do not need to do anything. For the life of me, I cannot remember if I needed to create the ntopng user in MariaDB with a password of ntopng. If I did, these would be the commands:

````SQL
sudo mysql -u root -p
CREATE USER 'ntopng'@localhost IDENTIFIED BY 'ntopng';
exit;
````

## Query the Database for the Top Flows

My test case is my Home Lab. My Home Lab has two VMware ESXi servers (Intel NUC, Raspberry Pi 4) and one Linux laptop. I wanted to collect network flows between VMs and physical hosts to see which apps were using the most network bandwidth. This requires 
# Thank You
