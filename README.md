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

## Write the Flows to a Database

## Query the Database for the Top Flows

My test case is my Home Lab. My Home Lab has two VMware ESXi servers (Intel NUC, Raspberry Pi 4) and one Linux laptop. I wanted to collect network flows between VMs and physical hosts to see which apps were using the most network bandwidth. This requires 
# Thank You
