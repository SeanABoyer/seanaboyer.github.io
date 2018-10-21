---
layout: post
title: University Pen Testing Lab
---
I created an air-gapped pen testing lab for myself and students at my univerity to learn offensive security.

## Problem
My University had hardware that was setup by students whom later graduated and left no documentation on how it was setup or any information to login to the Hypervisors. The current students had started back up a club to compete in a Network Security and Defense competition called [Cyber Panoply](http://cyberpanoply.com/). We needed a lab so that we could practice offensive security in a secure environment.

## Quick Solution OverView
We scrapped all the Hypervisors's off the current hardware and reinstalled new HyperVisors. We downloaded VM's off of [VulnHub](www.vulnhub.com) and imported them onto the HyperVisors, took an inital snapshot once they started up so that we could revert them weekly(or as we needed) so we could keep learning from these VM's. I then setup a simple DHCP/WebServer to distribute IP's and allow non admins to see which VM was running on which IP(this allowed us to google for walkthroughs if we needed them).

The full documentation was left with the faculty and student organizaiton that now runs the lab, most of this documentation is from memory + scripts I had saved.
## In-Depth Solution OverView

### Hardware
We had the following hardware:
  ```
  2 Unmanaged Trednet gigabit switches
  
  2 Poweredge T300 Servers with:
    1 Intel Core 2 Duo E6305 @1.86GHZ
    24GB Ram
    4 250GM HDD's > Raid 0
    
  1 Poweredge T605 Server with:
    32GB Ram
    4 250GB HDD's > Raid 0
  ```

### Configuration
All 3 servers were running ESX 6.0, we had a virtual Vcenter Server Appliance running on the T605. We setup a virtual Debian server running DNSMASQ and Apache2 to allow for DHCP and apache to run a very simple HTML page to display all of the VM's running in the environment.

#### ESX Hosts
In order to allow students to attack the same Vulnerable VM's on a repetitve basis we setup a script to revert the VM to the most recent snapshot.

##### Revert_Snaphots.sh
```bash
#Get all vmid's where not the vcenter or the DHCP server
VMIDS=$(vim-cmd vmsvc/getallvms | grep -v 01_Vcenter | grep -v 02_DNS_DHCP | awk '{print $1 }' | grep -v Vmid)
#loop though VMID's and check if there is a snapshot by the name of fresh install. If so, revert the vm to that snapshot and then power on the VM
for VMID in $VMIDS
do
        case $VMID in
                ''|*[!0-9]*);;
                *)
                vim-cmd vmsvc/get.snapshot $VMID | grep "Fresh Install" > /dev/null
                if [ $? -eq 0 ]; then
                        echo $(vim-cmd vmsvc/get.summary $VMID | grep name)
                        SnapShotID=$(vim-cmd vmsvc/snapshot.get $VMID | grep "Fresh Install" -A 1 | grep "Id" | awk '{print $4}')
                        vim-cmd vmsvc/snapshot.revert $VMID $SnapShotID 0 > /dev/null
                        vim-cmd vmsvc/power.on $VMID
                fi
                ;;
        esac
done
```
In order to make sure there was a Snapshot to revert back to, we setup a script to create a snapshot if there was not one for the VM.

##### Create_Snapshots.sh
```bash
#Get all vmid's where not the vcenter or the DHCP server
VMIDS=$(vim-cmd vmsvc/getallvms | grep -v 01_Vcenter | grep -v 02_DNS_DHCP | awk '{print $1 }' | grep -v Vmid)
#loop though VMID's and check if there is a snapshot by the name of fresh install. If not, take a snapshot called fresh install.
for VMID in $VMIDS
do
        case $VMID in
                ''|*[!0-9]*);;
                *)
                vim-cmd vmsvc/get.snapshot $VMID | grep "Fresh Install" > /dev/null
                if [ ! $? -eq 0 ]; then
                        vim-cmd vmsvc/snapshot.create $VMID "Fresh Install"
                fi
                ;;
        esac
done
```
In order to ___ we setup a script to get the MAC address and Name of each VM and save it to a file.

##### Script.sh
```bash
#Get all vmid's
VMIDS=$(vim-cmd vmsvc/getallvms | awk '{print $1 }' | grep -v Vmid)
echo "" > MAC_ADDRESSES
#loop though VMID's and get the mac address and name of the vm. Add them to a file.
for VMID in $VMIDS
do
        case $VMID in
                ''|*[!0-9]*);;
                *)
                MacAddress=$(vim-cmd vmsvc/device.getdevices $VMID | grep macAddress | sed 's/macAddress = "//' | sed 's/",//' | sed -r 's/\s+//g')
                VmName=$(vim-cmd vmsvc/get.summary $VMID | grep name | sed 's/name = "//' | sed 's/",//' | sed -r 's/\s+//g')
                echo $MacAddress        $VmName >> MAC_ADDRESSES
                ;;
        esac
done
```

All of these were scheduled via crontab.

##### crontab(/var/spool/cron/crontab/root)
```
#Run every 5 minutes
*/5 * * * * /root/script.sh
#Run every Saturday at 5am
* 5 * * 6/root/Create_Snapshots.sh
#Run every Sunday at 5am
* 5 * * 7/root/Revert_Snapshots.sh
```
#### DHCP/Apache2 VM
We needed to get all the files containing the names and mac address from the ESX hosts, so we created a script to pull those over and combine them.

##### Order.sh
```bash
#Gets all of the IP's and Mac addresses that have been leased
cat /var/lib/misc/dnsmasq.leases | awk '{print $2 " " $3}' > /root/DHCP
#obtains the Mac addresses from ESX4
scp root@192.168.1.10:/MAC_ADDRESSES /root/MAC_ESX4
#obtains the Mac addresses from ESX3
scp root@192.168.1.9:/MAC_ADDRESSES /root/MAC_ESX3
#obtains the Mac addresses from ESX2
scp root@192.168.1.8:/MAC_ADDRESSES /root/MAC_ESX2
#Combines all of the individual Mac Address files into one
cat /root/MAC_ESX* > /root/MAC
#Runs the IP_TO_MAC script
/root/IP_TO_MAC.sh
```

We had the script that obtained that data from the ESX hosts kick off another script that would then build the HTML file to display to us.

##### IP_TO_MAC.sh
```bash
#This script creates the HTML page to show the IP addresses and VMnames
#make sure file is empty
echo "" > /root/iptomac
#add the title html to file
echo "<title>CyberSaints Vurnable IP's</title>" >> /root/iptomac
#add the text to file
echo "<i>This webpage is updated every minute. If there are any issues, check the /root/order.log file. The scripts that create this page are in /root</i><br><br>" >> /root/iptomac
echo "If you are trying to pentest one of these IP's and are struggling, try a walkthough at www.vulnhub.com" >> /root/iptomac
#Add table start to file
echo "<table>" >> /root/iptomac
#Add table headers to file
echo "<tr><th>VMname</th><th>Ipaddress</th></tr>" >> /root/iptomac
#loop through the DHCP file, compare the macs to the files in MAC. If they match, add a new row to the html file of IP/VMname
while read line; do
    MAC=$(echo $line | awk '{print $1}')
    if grep -Rq $MAC /root/MAC
    then
        echo "<tr><td>" >> /root/iptomac
        echo $(grep -R $MAC /root/MAC | awk '{print $2}' >> /root/iptomac
        echo "</td><td>" >> /root/iptomac
        echo $line | awk '{print $2}') >> /root/iptomac
        echo "</td></tr>" >> /root/iptomac
    fi
done </root/DHCP
echo "</table>" >>/root/iptomac
#Copy the file to the index.html file to be displayed to website
cp /root/iptomac /var/www/html/index.html
```


In order to keep what VM's were online, we ran the order script regualarly.

##### crontab
```
#Runs every minute
*/1 * * * * /root/order.sh >> /root/order.log
```
