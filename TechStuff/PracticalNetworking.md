### Network Troubleshooting methodology

<br />

🌐📶

---

Useful resources: <a href="https://benpiper.com/"> Ben Piper </a> and <a href="https://www.dummies.com/programming/networking/cisco/network-basics-remote-host-arp-requests/"> Short intro to ARP </a>


For basic networking troubleshooting:
> One should learn common commands for diagnosing and troubleshooting network related issues. Some of the utilities SREs/Software Developers use include things like ping, traceroute, ipconfig/ip/ifconfig, nslookup, and netsh. One should also learn how and when to use them and how to interpret their outputs. 

##### Sections

###### Background
1. Introduction to internetworking
    * Internetworking and internet

2. Connectivity
    * LANs
    * Routers/Layer-3 Switches (physical devices) and Gateways (interface)
    * ARP protocol: IP (level-3) -> MAC (level-2), default gateways <-> MAC mapping for destination packet building
    > Every machine has a default gateway i.e. a physical devide/IP that can route packets to the destination. This gateway can be a Router or a Layer-3 switch. While creating a destination packet, machine uses ARP protocol to get level-3 device's MAC address. It encapsulates the destination IP and gateway MAC in that packet.

    > Same thing happens in reverse-order on the receiving-side
    * Firewalls: for security and NAT-ing
    * ISP: Customer Premises Equipment (CKE) router and ISP-peering
    > These CKEs (given by ISPs) provide an interface to on-premise firewalls to connect to ISPs. ISPs peer with each other to facilitate the connection.
    * Root servers and ADNS
    > Some huge orgs maintain root DNS servers; while querying a site over the internet, intranet DNS server connects to one of these root servers and queries for the destination address IP. These root servers know what the Authoratative DNS for that external site is. This ADNS then provides the IP address of pertaining service. 

3. Troubleshooting methodology
    * Defining the problem
    * Isolate the layer
    * Isolate the scope
    * Isolate the cause

---

#### Isolating the layer

We go from the lowest layers (Physical, DLL) to the highest (Network, Transport) since each layer depends on all lower layers. <br />
Networking related problems are generally upto transport layer. <br />
The last 3, Session, Presentation, Application layers are generally application specific and one needs to have some sort of application implimentation knowledge.

##### 1. Physical Layer
* Is the network cable plugged in?
* Does the NIC have a link light?

##### 2. Data Link Layer
* Initial check: `ipconfig/ifconfig` to check if ip and netmask are configured properly, ARP won't work otherwise
* Is the problem confined to intranet or spans internet as well?

* Other devices on LAN are reachable?
```sh
$ ping <broadcast_ip>
$ arp -a
```
* Is the default gateway reachable? Same step as above with the default/ARP gateway

If the IP address is wrong, we should get one assigned through DHCP. Some switches are well equipped enough to act as a DHCP server: `ifconfig/ipconfig` lists the DHCP server.

##### 3. Network
* Is the destination reachable? `traceroute, pathping`


##### IPv4
Possible causes:
* Wrong IPv4 default gateway
* Static APR entry; should be dynamic

```sh
# delete ARP cache
$ arp -d
```
Can also use `netsh` (Network Shell)

##### IPv6

IPv6 does not use ARP. It uses Neighbor Discovery Protocol; operates in similar pricipal to ARP but we administer it differently.

`arp -a` equivalent:
```sh
$ netsh
> interface ipv6
> show neighbors
# arp -d
> delete neighbors
# disable router discovery (ipv6 feature) if required
> set interface "Local Area Network" routerdiscovery=disabled
```

##### 4. Transport
* Ports come into play in this layer. Check if they are accepting connections on standard/requested ports.
`telnet, nslookup`

Ports dictate what application will be receiving packets. We can reach upto layer-7 if we know the port and most of the times the troubleshooting takes place on the application layer. <br />
Use telnet to see if the application uses just HTTP instead of HTTPS.

#### SSL/TLS toubleshooting
Use `openssl` command to glean more information about the certificate <br />
* Check if the certificate is correct (Canonicalized DNS name for Kerberos)
* Check if the certificate is valid; not expired
* Check NGINX/Apache (revese proxy) logs and configs

#### Isolating the scope

* How many users are affected?
* How long has this problem been going on?
* Have you ever had this problem before?

### Network slowness

* Different methods while troubleshooting IPv4 (and ICMPv4 pings) and IPv6 (ICMPv6 pings). Traceroute sends multiple packets, don't be alarmed if some of them time-out.

* Run traceroute multiple times, sometimes packets take different routes. This can explain intermittant access. One of the routes can be damaged.
