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

![MariaDB](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/MariaDB.png)

````bash
sudo apt install mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb (check for any issues and fix)
sudo systemctl enable mariadb
````

#### Install ntop(ng)

![ntop](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/ntop.png)

````bash
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

You can find a copy of my ntopng.conf [file](https://github.com/DennisFaucher/networkflowlucid/blob/main/ntopng.conf) in this repo. My only difference is that I changed the port the ntopng web interface runs on from 3000 to 4000. Also, my MariaDB server is on a different host than localhost.

## Query the Database for the Top Flows

![SQL Logo](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/SQL.png)

It was time to dust off my rusty SQL skills and to start poking at the MariaDB ntop records to looks for the top flows. 

````SQL
use ntopng
SELECT INET_NTOA(ip_src_addr) as "SRC_IP", l4_src_port as "SRC_PORT", INET_NTOA(ip_dst_addr) \
as "DST_IP", l4_dst_port as "DST_PORT", format(SUM(in_bytes),0) as "IN", format(SUM(out_bytes),0) \
as "OUT" from flowsv4 where INET_NTOA(ip_src_addr) != "192.168.1.151" and INET_NTOA(ip_dst_addr) \
!= "192.168.1.151" and INET_NTOA(ip_dst_addr) LIKE "192.168.1.%" group by l4_dst_port  \
order by out_bytes desc limit 10;
+--------------+----------+---------------+----------+-------------+----------------+
| SRC_IP       | SRC_PORT | DST_IP        | DST_PORT | IN          | OUT            |
+--------------+----------+---------------+----------+-------------+----------------+
| 192.168.1.69 |    57668 | 192.168.1.240 |      902 | 188,567,181 | 28,614,544,156 |
| 192.168.1.66 |    55846 | 192.168.1.55  |      443 | 559,914,960 | 2,351,076,647  |
| 192.168.1.66 |    45152 | 192.168.1.51  |     9100 | 3,564,870   | 36,772,742     |
| 192.168.1.66 |     8086 | 192.168.1.51  |    38324 | 181,208     | 1,172,396      |
| 192.168.1.66 |    52326 | 192.168.1.51  |     9091 | 2,122,212   | 15,249,382     |
| 192.168.1.66 |     8086 | 192.168.1.65  |    48718 | 695,958     | 2,436,358      |
| 192.168.1.66 |     8086 | 192.168.1.65  |    48604 | 3,463,442   | 7,196,070      |
| 192.168.1.51 |    32400 | 192.168.1.71  |    49910 | 3,818,437   | 1,267,671      |
| 192.168.1.65 |    48718 | 192.168.1.66  |     8086 | 418,736,680 | 89,940,248     |
| 192.168.1.65 |    60818 | 192.168.1.71  |     8181 | 4,516,705   | 10,383,015     |
+--------------+----------+---------------+----------+-------------+----------------+
````

# Thank You
