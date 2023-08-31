# Automated OS provision using Foreman
###  Birds eye view 
We will install and setup forman on a Almalinux virtual machine which will install puppetserver, DHCP server, TFTP server and we will setup intenal private network on virtual box that will help us to do automated PXE provison.

If you are interested in manual PXE booting you can go [here](https://github.com/vaibhav0320/PXE-booting-guide) where I mention what is DHCP and TFTP server. ( or you can google it for better results  )


### Step 1: Setup Virtual Machine

I've used Almalinux 8.8 with 2 cpu cores and 4 GB of RAM on virtualbox. I use [Vagrant](https://www.vagrantup.com/) for quick VM deployment. For networking I've 3 ports connected to my VM. first is NAT from virtual box, second internal private network (FOR DHCP) and third bridged network that is connected to my home router. Used 20GB of disk for filesystem.

You can manually configure or I've provided vagrant file if you're lazy doing all this stuff. [Vagrantfile]()

### Step 2: Install foreman

After the VM is up and running we need to install foreman which is a piece of software by which we can provision new machines, configures it and monitor is from single dashboard. More about foreman [here](https://www.theforeman.org/). 

You can follow the steps here to install basic [foreman](https://www.theforeman.org/manuals/3.1/quickstart_guide.html). Once installed try to login from browser to the ip address that assigned to VM by your router. If you see login page then you can follow next steps else need to troubleshoot it further. 

### Step 3: Setup DHCP and TFTP using foreman

Copy and edit below snippet and run it in your shell to setup the DHCP and TFTP server.
```bash showLineNumbers
foreman-installer \
--enable-foreman-proxy \
--puppet-puppetmaster=puppetmaster.vm.local \
--foreman-proxy-tftp=true \
--foreman-proxy-tftp-servername=192.168.128.2 \
--foreman-proxy-dhcp=true \
--foreman-proxy-dhcp-interface=enp0s8 \
--foreman-proxy-dhcp-gateway=192.168.128.2 \
--foreman-proxy-dhcp-range="192.168.128.31 192.168.128.40" \
--foreman-proxy-dhcp-nameservers="192.168.128.2" \
--foreman-proxy-dns=true \
--foreman-proxy-dns-interface=enp0s8 \
--foreman-proxy-dns-zone=vm.local \
--foreman-proxy-dns-reverse=128.168.192.in-addr.arpa \
--foreman-proxy-dns-forwarders=192.168.128.2 \
--foreman-proxy-foreman-base-url=https://puppetmaster.vm.local \
```

Explanation of each line

1. ```foreman-installer ``` : starts foreman installer
2. ```--enable-foreman-proxy ``` : enables foreman proxy
3. ```--puppet-puppetmaster=puppetmaster.vm.local ``` : puppet master server hostname (NEED TO EDITED AS PER YOUR HOSTNAME)
4. ```--foreman-proxy-tftp=true ```: configures TFTP
5. ```--foreman-proxy-tftp-servername=192.168.128.2```: IP of tftp server ( I've used IP of internal network everywhere. )
6. ```--foreman-proxy-dhcp=true```: Setups the DHCP server
7. ```--foreman-proxy-dhcp-interface=enp0s8```: Interface name (Use the interface which is for private internal network)
8. ```--foreman-proxy-dhcp-gateway=192.168.128.2 ```: Internal network gateway IP
9. ```--foreman-proxy-dhcp-range="192.168.128.31 192.168.128.40" ```: IP range which will be assigned to DHCP clients
10. ```--foreman-proxy-dhcp-nameservers="192.168.128.2" ```: Nameserver IP
11. ```--foreman-proxy-dns=true ```: Setup the DNS server
12. ```--foreman-proxy-dns-interface=enp0s8 ```: Interface for DNS server
13. ```--foreman-proxy-dns-zone=vm.local```: domain name for internal network (my case its vm.local)
14. ```--foreman-proxy-dns-reverse=128.168.192.in-addr.arpa ```: reverse Domain name record (reverse the first 3 numbers from IP address you used for internal network)
15. ```--foreman-proxy-dns-forwarders=192.168.128.2 ```: Gateway IP
16. ```--foreman-proxy-foreman-base-url=https://puppetmaster.vm.local```: Use you hostname

Once done login to foreman from browser and go to smart proxy there you should see puppet, DHCP, TFTP and DNS. If not troubleshoot the issue.

### Step 4: Setup NAT on foreman server

After completing the above step we are good to provision the server but issue is when the newly provisioned server try to fetch the OS image from internet it will fail because it is connected to our private network which is not connected to internet. One solution is to host the OS image on private network and other is to setup net on foreman server.

So what happens is when we provision a new machine it will get the IP from DHCP server where we mention the gateway IP which is the IP of our foreman server. so all the non local traffic will reach to our Foreman server where we will tell the iptables to forward the traffic using NAT from the interface which is connected to internet.

Use below commands to setup NAT using IPTABLES. Run this on your FOREMAN server. Adjust rule position accordingly if you already have iptables rules in place
```bash
# Enable IP forwading in kernel ( it will reset on restart)
echo 1 > /proc/sys/net/ipv4/ip_forward

# Add rule to filter table to allow forwading the packets coming through private interface in my case it was enp0s8
iptables -A FORWARD -i enp0s8 -j ACCEPT

# Add rule to nat tables postrouting chain to process the packet for nat. Here the interface is the one that connected to internet
iptables -A POSTROUTING -o enp0s9 -j MASQUERADE

```
this should do the work. You can test it by connecting a running VM to this private network and doing ping.

### Step 5: Provision the new VM

Create a new vm from virtualbox with the configuration you like but for network connect it to the our custom private network and keep pxe booting(network booting) ON.




