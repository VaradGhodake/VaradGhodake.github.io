### alias docker='podman'

<br />

☸️🐋

---

Stumbled across this tweet today:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">I completely forgot that ~2 months ago I set up &quot;alias docker=&#39;podman&#39;&quot; and it has been a dream. <a href="https://twitter.com/hashtag/nobigfatdaemons?src=hash&amp;ref_src=twsrc%5Etfw">#nobigfatdaemons</a> <a href="https://twitter.com/projectatomic?ref_src=twsrc%5Etfw">@projectatomic</a></p>&mdash; Alan Moran ☕️💻 (@ialanmoran) <a href="https://twitter.com/ialanmoran/status/1001671953571303425?ref_src=twsrc%5Etfw">May 30, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Seems quite simple and effective, doesn't it? Not exactly. Let's go through some important details, specifically regarding network namespaces that one might miss out.

###### Background

### Container Networking

Useful resources: 
- Docker/Podman official documentations
- <a href="https://nigelpoulton.com/"> Nigel Poulton </a>
- <a href="https://www.redhat.com/sysadmin/container-networking-podman"> Redhat: Podman rootfull vs rootless networking </a>
- <a href="https://blogs.igalia.com/dpino/2016/04/10/network-namespaces/"> Bride-node networking tutorial </a> | <a href="https://gist.github.com/dpino/6c0dca1742093346461e11aa8f608a99"> Gist </a>

###### Basics
Components:
1. CNM: Container Network Model
> Network model tenets:
> - Sandbox: network namespace
> - Endpoint: IP address
> - Network: binds the ip with host
> It's the same idea as that of Bridge Networking (discussed below)
2. Libnetwork
> Implementation of the design spec i.e. CNM. This library takes care of setting the net namespace.

>  <br />
RE: Redhat blog listed above, **TL;DR:** <br />
> When using Podman as a rootless user, the network setup is automatic. Technically, the container itself does not have an IP address, because without root privileges, network device association cannot be achieved. Moreover, pinging from a rootless container does not work because it lacks the CAP_NET_RAW security capability that the ping command requires.

#### Bridge networking

```sh
Tools required: ip command
```
1. Create a network namespace with ip command
2. Create a veth pair: `veth1` and `vpeer1`
3. Push `vpeer1` inside the newly created namespace; create loopback `lo` as well
4. Assign IP addresses (make sure you use masking) to both of them and set them UP
5. Create iptable rule inside the namespace to set up routes to loopback and connects `vpeer1` and `veth1`
6. Enable forwarding through `/proc/sys/net/ipv4/ip_forward` and iptables rule b/w `eth0` and `veth1`

`ip` command cheatsheet:
```sh
$ ip {netns, link, route} show # show resources
# Execute commands inside the namespace with:
$ ip netns exec <namespace_name> <command>
```
---

A container is a collection of namespaces. Libnetwork takes care of setting up the net namespace. Plugins set up a topology.

Container Network:
<img src="assets/container-network.png" />

```sh
# list existing network namespaces
$ docker network ls
# create a network namespace
$ docker create network -d bridge brooklyn
# launch and bind this network to an existing container
$ docker run -dit --name nyc --network brooklyn alpine sh
# disconnect this network from the container
$ docker disconnect brooklyn nyc
# connect back with connect instead of disconnect  
```
Inter-container communication for docker can be achieved as above, it doesn't work for the default bridge, only for newly networks the containers will be able to talk to each other. <br />

#### Podman

There are some limitations with podman. Because of the dnsname plugin's dependency on dnsmasq: <br />
<a href="https://translate.google.com/translate?hl=en&sl=ja&u=https://ttsknight.com/2020/08/podman-network/&prev=search&pto=aue"> This blog </a> (translated from Japanese) explains it. The fix is to set `--disable-dns` flag.

Rootless containers need `-P` to talk through host's networking. Rootfull containers can attach to a user defined bridge and would be able to talk to each other just fine through IP or the container name.
