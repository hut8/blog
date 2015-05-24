+++
date = "2015-05-24T15:58:33-04:00"
draft = false
title = "Vultr Private Network Configuration"
tags = [ "vultr", "networking", "debian", "ubuntu", "performance", "vps" ]
icon = "plug42.svg"
+++

### for Debian/Ubuntu

[Vultr.com](http://www.vultr.com/?ref=6831514) has a cool feature where you can set up a private network between all of your instances. This provides three advantages:

* **It's more secure**: services need not listen on the public interface if its only consumers are other vultr instances
* **It's really fast**: gigabit ethernet
* **It's unmetered**: it doesn't count toward your bandwidth quota

This article is about how to configure it on Debian or Ubuntu.

<!--more-->

As root, we'll create a new file in `/etc/network/interfaces.d`. You can name it whatever you want; I named it `private`. It's a good idea to put a new file in `interfaces.d` rather than edit `/etc/network/interfaces` directly. This way, during upgrades, you don't have to merge files if the `interfaces` file changes. Make this file on each server, being careful to assign a unique IP address (one is suggested on the dashboard).

```none
liam@lorenz ~ % cat /etc/network/interfaces.d/private
auto eth1
iface eth1 inet static
     address 10.99.0.10
     netmask 255.255.0.0
     mtu 1450
```

As they note, you can actually use any private IP address that you want. They will only be visible to other instances in your account. Just make sure they're all unique and use something in the [private address space](https://www.arin.net/knowledge/address_filters.html).

Then apply the change:

```none
liam@lorenz ~ % sudo /etc/init.d/networking restart
```

That will not close your current connections, so you can do that over SSH.

#### Speed test

```none
liam@lorenz ~ % iperf -n 1024M -c crib                                                                                  ------------------------------------------------------------
Client connecting to crib, TCP port 5001
TCP window size: 45.0 KByte (default)
------------------------------------------------------------
[  3] local 10.99.0.10 port 36032 connected with 10.99.0.11 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-12.5 sec  1.00 GBytes   685 Mbits/sec
```

*Crazy fast!*

<hr>

#### MTU setting

I was pretty disappointed at first when the MTU was set to (the default) 1500:

```none
liam@lorenz ~ % iperf -n 1024M -c
------------------------------------------------------------
Client connecting to crib, TCP port 5001
TCP window size: 45.0 KByte (default)
------------------------------------------------------------
[  3] local 10.99.0.10 port 30123 connected with 10.99.0.11 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3] 0.0-779.6 sec  1.00 GBytes   11.0 Mbits/sec
```

[These vultr.com docs](https://www.vultr.com/docs/configuring-private-network) don't mention the MTU issue, although on the dashboard there's a mention of it if you click a button for more information.

As you can see, a difference of __6200%__!

What accounts for this massive difference?
[IP Fragmentation](http://en.wikipedia.org/wiki/IP_fragmentation). The VPN behind the private networking adds a little bit of data as a header on every packet. The MTU of the network is probably 1500. Then, when the OS sends a packet that's 1500 bytes, the overhead from the VPN pushes the packet size just over the MTU. That causes the packet to be fragmented (outside of the server), then all kinds of overhead ensues. So keep your MTU for that interface at 1450.
