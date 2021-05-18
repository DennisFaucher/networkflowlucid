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

It was time to dust off my rusty SQL skills and to start poking at the MariaDB ntop records to looks for the top flows. After lots of trial and error, this SQL statement gave me what I wanted:

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

I filtered out any flows coming from 192.161.1.51 (My WiFi network) and kept only the flows staying within my Home Lab network (192.168.1.%). Some of the interesting busy ports you see are:

* 902: NFS destination for my backup server
* 443: vCenter
* 9100, 9091: Prometheus (Grafana)
* 8086: InfluxDB (Grafana)
* 8181: Tautulli Plex Stats

If you are ever unsure what process has a numbered port open, just run the "lsof -i" command on your host:

````bash
sudo lsof -i | grep 8086
influxd      9189        influxdb   84u  IPv6 59719059      0t0  TCP localhost:8086->localhost:42488 (ESTABLISHED)
influxd      9189        influxdb   85u  IPv6 59571748      0t0  TCP ubuntu-nuc.fios-router.home:8086->photon-arm.fios-router.home:53650 (ESTABLISHED)
influxd      9189        influxdb   89u  IPv6 59519210      0t0  TCP ubuntu-nuc.fios-router.home:8086->rpios.fios-router.home:60260 (ESTABLISHED)
influxd      9189        influxdb  135u  IPv6 59572394      0t0  TCP ubuntu-nuc.fios-router.home:8086->medialinux.fios-router.home:44318 (ESTABLISHED)
influxd      9189        influxdb  146u  IPv6 61668069      0t0  TCP ubuntu-nuc.fios-router.home:8086->192.168.1.151:54665 (ESTABLISHED)
influxd      9189        influxdb  159u  IPv6 61962153      0t0  TCP localhost:8086->localhost:39084 (ESTABLISHED)
influxd      9189        influxdb  686u  IPv6   123187      0t0  TCP *:8086 (LISTEN)
telegraf     9432        telegraf    8u  IPv4 59720637      0t0  TCP localhost:42488->localhost:8086 (ESTABLISHED)
grafana-s  353242         grafana   18u  IPv4 61962152      0t0  TCP localhost:39084->localhost:8086 (ESTABLISHED)
````

## Create Automagic LucidCharts

![LucidChart Flows](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/Lucid%20Flow.png)

LucidChart (and the free Draw.IO) allow the import of CSV files to automatically create diagrams. They even provide CVS templates. I have a license to LucidChart through work, so I went that route.

### Get the CSV Import File Ready

After playing with the different LucidChart drawing types that support CSV import (Data Linking, Entity Relationship, Org Chart, Process Diagram, Smart Containers, Sticky Notes), I found that the Process Diagram was the closest to the network flow diagram I was after. From LucidChart, create a new diagram, then File > Import Data > Process Diagram > Import Your Data. You will be presented with this dialog box where you can import your CSV or download a CSV template.

![Lucid Import Screen](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/Lucid%20Import%20Screen.png)

Once you have downloaded the template, create a shape row for each of the hosts in the output of your SQL command. Each shape needs a unique ID in LucidChart, so I used the last octet of the host IP address for the shape ID.

![Lucid Shapes](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/Lucid%20CSV%20Hosts.png)

Below the rows for shapes, create rows for the flow lines. These rows are identical except for the source, destination, and text (port number) columns. I pasted the SQL output into Excel split into columns based on the pipe and space being column separators. I could them paste columns of source, destination, and port right in to the CSV table. Once you have placed the SQL output unto the correct columns delete any extraneous information you may have pasted into the CSV file and save.

![Lucid Lines](https://github.com/DennisFaucher/networkflowlucid/blob/main/images/Lucid%20CSV%20Flows.png)

Import this CSV file as data to create a process diagram and you will see black host circles conected by black port lines. Move the hosts around in the chart as needed for readability and the lines will follow. Format to your liking and you should have a flow diagram like mine.

# Thank You

Thank you for taking the time to read this post. I learned quite a bit working through this process and am sharing to make the process easier for others. I welcome any feedback, questions, or areas for improvement.
