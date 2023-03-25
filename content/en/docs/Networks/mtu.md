---
title: "MTU"
date: 2023-03-25
weight: 4
description: >
  A short lead description about this content page. It can be **bold** or _italic_ and can be split over multiple paragraphs.
---

{{% pageinfo %}}
This is a placeholder page. Replace it with your own content.
{{% /pageinfo %}}

# Abstract

This page explain how Linux kernel handling fragementation when transmit packet out, the TCP/UDP layer optimize to help IP layer processing fragmentation. How to discover path MTU from host to destination by ICMPv4 or ICMPv6. Also point out the problem when pass through router disable ICMPv4.

# What is MTU?

MTUs apply to communications protocols and network layers. The MTU is specified in terms of bytes or octets of the largest PDU that the layer can pass onwards. MTU parameters usually appear in association with a communications interface (NIC, serial port, etc.). Standards (Ethernet, for example) can fix the size of an MTU; or systems (such as point-to-point serial links) may decide MTU at connect time.

Underlying data link and physical layers usually add overhead to the network layer data to be transported, so for a given maximum frame size of a medium one needs to subtract the amount of overhead to calculate that medium's MTU. For example, with Ethernet, the maximum frame size is 1518 bytes, 18 bytes of which are overhead (header and FCS), resulting in an MTU of 1500 bytes.

MTU is the LAN that the network interface is connected to. If you transmit an IP packet to another host on the same LAN as the interface you use to transmit, and the size of the packet is bigger than the LAN’s MTU, the IP packet will have to be fragmented. However, if you chose a size that fits the MTU, you can ensure that no fragmentation will be required. When the destination host is not on a directly attached LAN, you cannot count on the LAN’s MTU to derive whether fragmentation will take place. Here is where path MTU discovery comes in.

# MTU Path discovery

For IPv4 packets, Path MTU Discovery works by setting the Don't Fragment (DF) flag bit in the IP headers of outgoing packets. Then, any device along the path whose MTU is smaller than the packet will drop it, and send back an Internet Control Message Protocol (ICMP) Fragmentation Needed (Type 3, Code 4) message containing its MTU, allowing the source host to reduce its Path MTU appropriately. The process is repeated until the MTU is small enough to traverse the entire path without fragmentation.

IPv6 routers do not support fragmentation and consequently don't support the Don't Fragment option. For IPv6, Path MTU Discovery works by initially assuming the path MTU is the same as the MTU on the link layer interface where the traffic originates. Then, similar to IPv4, any device along the path whose MTU is smaller than the packet will drop the packet and send back an ICMPv6 Packet Too Big (Type 2) message containing its MTU, allowing the source host to reduce its Path MTU appropriately. The process is repeated until the MTU is small enough to traverse the entire path without fragmentation.

If the Path MTU changes after the connection is set up and is lower than the previously determined Path MTU, the first large packet will cause an ICMP error and the new, lower Path MTU will be found. Conversely, if PMTUD finds that the path allows a larger MTU than is possible on the lower link, the OS will periodically reprobe to see if the path has changed and now allows larger packets. On both Linux and Windows this timer is set by default to ten minutes.

Maximum Path MTUIPv4 Maximum PMTU At least 68, max of 64 KB (RFC 791. p. 24)
IPv6 Maximum PMTU At least 1280(link MTU, RFC 8200), max of 64 KiB (RFC 2460), but up to 4GB with optional jumbogram (RFC 2460)

# Linux Kernel implementation

## PMTU

Path MTU discovery is used to discover the biggest size a packet transmitted to a given destination address can have without being fragmented. That parameter is called the Path MTU (PMTU) . Basically, the PMTU is the smallest MTU encountered along all the connections along the route from one host to the other.

Since the path between two endpoints can be asymmetric, it follows that there can be two different PMTUs for any given pair of hosts. Each host computes and uses the one appropriate for sending packets to the other. Furthermore, a change of route can lead to a change of PMTU.

Since each destination IP address can use a different PMTU, it is cached in the associated routing table cache entry. We will see the routes in the routing table can aggregate several IP addresses. For instance, you can have a route that says that network 10.0.1.0/24 is reachable via gateway 10.0.2.1. The routing table cache, on the other hand, has one single entry for each destination IP address the host has been talking to in the recent past, even though they are reached through the same gateway. Each of those entries includes a unique PMTU. You may therefore have an entry for host 10.0.1.2 and another one for 10.0.1.3. You may object that, if those two addresses belong to two hosts within the same LAN, a third host would probably use the same route to reach both hosts and therefore share the same PMTU. It would make sense to keep just one PMTU in the routing table. This is unfortunately not possible. Just because one route is used to reach a bunch of addresses does not necessarily mean that they belong to the same LAN.

Each routing table entry is associated with an egress device, the device to use to transmit traffic to the next hop along the route. If the device is directly connected to its correspondent and PMTU discovery is enabled, the PMTU is set by default to the MTU of the egress device.

Directly connected devices include the two endpoints of a telecom cable or devices on an Ethernet LAN. It’s particularly important for all devices on the LAN (with no router between them) to share the same MTU for proper operation.

If devices are not directly connected—that is, if at least one router lies between them—or if PMTU discovery is disabled, the PMTU by default is set to 576. This is not a random value, but is defined in the original IP RFC 791.[‡] Regardless of the default, an administrator can set the initial PMTU through a user-space configuration program such as ifconfig.

Let’s see how PMTU discovery works. The algorithm simply takes advantage of the IP header’s fields used to handle fragmentation/defragmentation and the associated ICMP messages.

If you transmit an IP packet with the DF flag set in the header and no one complains, it means that no fragmentation has taken place along the path to the destination, and that the PMTU you used is fine. This does not mean you are using the optimal size. You might well be able to increase the PMTU and still not have fragmentation. A simple example is where two Ethernet LANs are connected by a router. On both sides of the network, the MTU is 1,500, but hosts of each LAN use the MTU of 576 to talk to the hosts of the other LAN because they are not directly connected. This is not optimal.

If you increase the size of the packets in a probe to their optimal size, you will be notified with an ICMP message when you cross the real PMTU. The ICMP message will include the MTU of the device that complained so that the kernel can update the local PMTU accordingly.

Linux can be configured to handle path MTU discovery in one of the following ways:
    IP_PMTUDISC_DONT
        Never send IP packets with the DF flag set in the header; therefore, do not use path MTU discovery.
    IP_PMTUDISC_DO
        Always set the DF flag in the header of packets generated on the local node (not forwarded ones), in an attempt to find the best PMTU for every transmission.
    IP_PMTUDISC_WANT
         Decide whether to use path MTU discovery on a per-route basis. This is the default.
When path MTU discovery is enabled, the PMTU associated with a route can change at any time to include routers with a smaller maximum size, resulting in the source receiving an ICMP FRAGMENTATION NEEDED message. In this case, the PMTU is updated for all the entries in the routing cache with the same destination. It should be noted that the algorithm always shrinks the PMTU, it never increases it. However, the entries of the routing cache whose PMTU is derived from an ingress ICMP FRAGMENTATION NEEDED message expire after some time(600 seconds be default), which is equivalent to going back to the (bigger) default PMTU. Manually delete route entry from table don't clean the table cache, you could see the pmtu is stored back when add same route entry back.

The PMTU of a route can also be set manually when adding the route through the ip route command.

Even if path MTU discovery was enabled, it is still possible to lock the current PMTU so that it will not be changed. This happens in two main cases:

When using ip route to set the PMTU, it is possible to lock it with the lock keyword. The following example adds a route to the 10.10.1.0/24 network via the next hop gateway 100.100.100.1 and locks the PMTU to 750 bytes:

```shell
ip route add 10.10.1.0/24 via 100.100.100.1 mtu lock 750
```

In Linux, the ip_dont_fragment function uses the considerations described here to decide whether a packet should be fragmented when it exceeds the PMTU.

The value of the PMTU on a given transmission can also be influenced by the following factors:
 Whether the device’s MTU is explicitly configured from user space
 Whether the application has changed the maximum segment size (mss) to use on a given TCP socket
If the PMTU you are supposed to use as a consequence of a received ICMP FRAGMENTATION NEEDED message is smaller than the minimum allowed value, the PMTU is set to that minimum value, and locked. The minimum value can be configured with the /proc/sys/net/ipv4/route/min_pmtu file. In any case, the PMTU cannot be set to a value lower than 68, as requested by RFC 1191, section 3.0 (and indirectly by RFC 791, section “Fragmentation and reassembly”).

### IP Fragmentation

In particular, ip_fragment must be able to handle both of the following cases:

Big chunks of data that need to be split into smaller parts.
    Splitting the big buffer requires the allocation of new buffers and memory copies from the big buffer to the small ones. This, of course, impacts performance.

A list or array of data fragments that do not need to be fragmented further.
    If the buffers were allocated such that they have room to allow the addition of lower-layer L3 and L2 headers, ip_fragment can handle them without a memory copy. All the IP layer needs to do is add an IP header to each fragment and handle the checksum.

### L4 protocols help

Newer kernels is to make the L4 protocols aid in the fragmentation task in advance: instead of passing to the IP layer a single buffer that will have to be fragmented, they can pass a set of buffers appropriate to the PMTU. This way, the IP fragmentation handled at the IP layer consists simply of creating an IP header for each data fragment already formed. This does not mean that the L4 protocols implement IP fragmentation; it simply means that since L4 protocols are aware of IP fragmentation, they try to cooperate and make life easier for the IP layer. The L4 protocols do not touch the IP headers.

### TCP MSS (Maximum segment size)

The maximum segment size (MSS) is a parameter of the options field of the TCP header that specifies the largest amount of data, specified in bytes, that a computer or communications device can receive in a single TCP segment. It does not count the TCP header or the IP header (unlike, for example, the MTU for IP datagrams). The IP datagram containing a TCP segment may be self-contained within a single packet, or it may be reconstructed from several fragmented pieces; either way, the MSS limit applies to the total amount of data contained in the final, reconstructed TCP segment.

The selection of the MSS is based on the need to balance various competing performance and implementation issues in the transmission of data on TCP/IP networks. The main TCP standard, RFC 793, doesn't discuss MSS very much, which opened the potential for confusion on how the parameter should be used. RFC 879 was published a couple of years after the TCP standard to clarify this parameter and the issues surrounding it. Some issues with MSS are fairly mundane; for example, certain devices are limited in the amount of space they have for buffers to hold TCP segments, and therefore may wish to limit segment size to a relatively small value. In general, though, the MSS must be chosen by balancing two competing performance issues:

Overhead Management: The TCP header takes up 20 bytes of data (or more if options are used); the IP header also uses 20 or more bytes. This means that between them a minimum of 40 bytes are needed for headers, all of which is non-data “overhead”. If we set the MSS too low, this results in very inefficient use of bandwidth. For example, suppose we set it to 40; if we did, a maximum of 50% of each segment could actually be data; the rest would just be headers. Many segment datagrams would be even worse in terms of efficiency.

IP Fragmentation: TCP segments will be packaged into IP datagrams. As we saw in the section on IP, datagrams have their own size limit issues: the matter of the maximum transmission unit (MTU) of an underlying network. If a TCP segment is too large, it will lead to an IP datagram is too large to be sent without fragmentation. Fragmentation reduces efficiency and increases the chances of part of a TCP segment being lost, resulting in the entire segment needing to be retransmitted.

### TCP Default Maximum Segment Size

The solution to these two competing issues was to establish a default MSS for TCP that was as large as possible while avoiding fragmentation for most transmitted segments.

This was computed by starting with the minimum MTU for IPv4 networks of 576 (RFC 879, page 1, Section 1). All networks are required to be able to handle an IP datagram of this size without fragmenting. From this number, we subtract 20 bytes for the TCP header and 20 for the IPv4 header, leaving 536 (= 576 - 20 - 20) bytes. This is the standard MSS for TCP.

This was computed by starting with the minimum MTU for IPv6 networks of 1280 (RFC 2460, page 24, Section 5). All networks are required to be able to handle an IP datagram of this size without fragmenting. From this number, we subtract 20 bytes for the TCP header and 40 for the IPv6 header, leaving 1220 (=1280 - 40 - 20 ) bytes. This is the standard MSS for TCP.

The selection of this value was a compromise of sorts. When this number is used, it means that most TCP segments will be sent unfragmented across an IP internetwork. However, if any TCP or IP options are used, this will cause the minimum MTU of 576 to be exceeded and fragmentation to happen. Still, it makes more sense to allow some segments to be fragmented rather than use a much smaller MSS to ensure that none are ever fragmented. If we chose, say, an MSS of 400, we would probably never have fragmentation, but we'd lower the data/header ratio from 536:40 (93% data) to 400:40 (91% data) for all segments.

### Set MSS

ip route command: ip route add 192.168.1.0/24 dev eth0 advmss 1500
socket option:
       TCP_MAXSEG
              The maximum segment size for outgoing TCP packets.  In Linux
              2.2 and earlier, and in Linux 2.6.28 and later, if this option
              is set before connection establishment, it also changes the
              MSS value announced to the other end in the initial packet.
              Values greater than the (eventual) interface MTU have no
              effect.  TCP will also impose its minimum and maximum bounds
              over the value provided.

Segmentation Offloads (Network Interface card hardware accelaration)

### TCP Segmentation Offload

TCP segmentation allows a device to segment a single frame into multiple frames with a data payload size specified in skb_shinfo()->gso_size. When TCP segmentation requested the bit for either SKB_GSO_TCPV4 or SKB_GSO_TCPV6 should be set in skb_shinfo()->gso_type and skb_shinfo()->gso_size should be set to a non-zero value.

TCP segmentation is dependent on support for the use of partial checksum offload. For this reason TSO is normally disabled if the Tx checksum offload for a given device is disabled.

In order to support TCP segmentation offload it is necessary to populate the network and transport header offsets of the skbuff so that the device drivers will be able determine the offsets of the IP or IPv6 header and the TCP header. In addition as CHECKSUM_PARTIAL is required csum_start should also point to the TCP header of the packet.

For IPv4 segmentation we support one of two types in terms of the IP ID. The default behavior is to increment the IP ID with every segment. If the GSO type SKB_GSO_TCP_FIXEDID is specified then we will not increment the IP ID and all segments will use the same IP ID. If a device has NETIF_F_TSO_MANGLEID set then the IP ID can be ignored when performing TSO and we will either increment the IP ID for all frames, or leave it at a static value based on driver preference.

TCP Segmentation Offload is supported in Linux by the network device layer. A driver that wants to offer TSO needs to set the NETIF_F_TSO bit in the network device structure. In order for a device to support TSO, it needs to also support Net:TCP checksum offloading and Net:Scatter Gather.

The driver will then receive super-sized skb's. These are indicated to the driver by skb_shinfo(skb)→gso_size being non-zero. The gso_size is the size the hardware should fragment the TCP data. TSO may change how and when TCP decides to send data.

If the driver has setup hooks in ETHTOOL_OPS() than TSO can be disabled
ethtool -K enccw0.0.f500 tx on sg on tso on

### UDP Fragmentation Offload

UDP fragmentation offload allows a device to fragment an oversized UDP datagram into multiple IPv4 fragments. Many of the requirements for UDP fragmentation offload are the same as TSO. However the IPv4 ID for fragments should not increment as a single IPv4 datagram is fragmented.

UFO is deprecated: modern kernels will no longer generate UFO skbs, but can still receive them from tuntap and similar devices. Offload of UDP-based tunnel protocols is still supported.

### Generic Segmentation Offload

Generic segmentation offload is a pure software offload that is meant to deal with cases where device drivers cannot perform the offloads described above. What occurs in GSO is that a given skbuff will have its data broken out over multiple skbuffs that have been resized to match the MSS provided via skb_shinfo()->gso_size.

Before enabling any hardware segmentation offload a corresponding software offload is required in GSO. Otherwise it becomes possible for a frame to be re-routed between devices and end up being unable to be transmitted.

For now GSO is off by default but can be enabled through ethtool. It is conceivable that with enough optimisation GSO could be a win in most cases and we could enable it by default.

However, even without enabling GSO explicitly it can still function on bridged and forwarded packets. As it is, passing TSO packets through a bridge only works if all constiuents support TSO. With GSO, it provides a fallback so that we may enable TSO for a bridge even if some of its constituents do not support TSO.

This provides massive savings for Xen as it uses a bridge-based architecture and TSO/GSO produces a much larger effective MTU for internal traffic between domains.


# Testing

## PMTU enabled

Linux default PMTU discovery is enabled on both VM and PC

$ cat /proc/sys/net/ipv4/ip_no_pmtu_disc
0
PC (tcp client) send 50KB data to VM (tcp server)
When PMTU discovery enabled, the IP flag DF always set
Router forward the packet, don't send out the ICMP
Linux don't learn PMTU

```shell
#ip route get to 3.3.0.181
3.3.0.181 via 3.9.0.254 dev enp4s0 src 3.9.0.1 uid 0
```
 cache
Captured packets: N/A
VM (tcp client) send 50KB data to PC (tcp server)
When PMTU discovery enabled, the IP flag DF always set
Router will reply ICMP to indicate the MTU when received packet size larger than the MTU
Linux will save the MTU to route cache, the timeout is 600 seconds.

```shell
$ ip route get to 3.9.0.1
3.9.0.1 via 3.3.0.254 dev ens9 src 3.3.0.181
cache expires 600sec mtu 1000

$ cat /proc/sys/net/ipv4/route/mtu_expires
600
Captured packets
```

## PMTU disabled

PC (tcp client) send 50KB data to VM (tcp server)

Set PMTU discovery disabled: 
echo 1 > /proc/sys/net/ipv4/ip_no_pmtu_disc


The IP flag DF always not set
Router forward the packet, don't send out the ICMP
Linux can't learn PMTU
VM (tcp client) send 50KB data to PC (tcp server)
Set PMTU discovery disabled: 
echo 1 > /proc/sys/net/ipv4/ip_no_pmtu_disc


The IP flag DF always not set
Router don't reply ICMP
Router fragment packets per its mtu and forward
Linux can't learn PMTU
Captured packets




Static Set PMTU
VM (tcp client) send 50KB data to PC (tcp server)

set pmtu on VM
ip route add 3.9.0.0/16 via 3.3.0.254 dev ens9 mtu lock 800


$ ip route get to 3.9.0.1
3.9.0.1 via 3.3.0.254 dev ens9 src 3.3.0.181
 cache mtu lock 800
VM send packets with the length 800 + 18 (Ethernet head + type + FCS) + 4 (if vlan exist)
Router forward the packet, don't send out the ICMP

If the set locked mtu larger than router mtu, go to above PMTU enabled/disable case
Captured packet



TCP MSS
Default TCP MSS set to interface MTU
TCP packet size should be the minimum size of interface MTU, peer MSS and PMTU(learned or configured)
TCP notify peer the MSS in the TCP option during three-way handshaking, actual in "SYNC" packet, you could find in above captured packets


Segment offloads
Hardware assemble or fragment packets according upper layers parameters.
It don't impact the egress packets length
Notice: While using tcpdump(libpcap) or wireshark(libpcap in unix likes, npcap in windows) to capture packets, it would impacted by the segment offload feature, the captured packets are not the last packets send out, NIC will do offload them.
Where tcpdump/wireshard work:

NIC Segment offloads on on receiver, TSO off on sender

captured packet: the received packets assembled from the wireshark captured, but I don't see fragmentation happened when sending out packet. segmentation_ingress.pcapng is the mirrored packets from mediate device.


NIC Segment offloads off
captured packet: No packets assembled or fragmented 

TSO both enabled on sender and receiver, I can't mirror the packets pass through mediate devices, the MTU is 1500 from end to end.
You could find larger packets send to NIC driver in sender, larger packets lift to IP stack from NIC driver in receiver.
 
Problem
Many network security devices block all ICMP messages for perceived security benefits (for example, to prevent denial-of-service attacks), including the errors that are necessary for the proper operation of PMTUD. This can result in connections that complete the TCP three-way handshake correctly, but then hang when data is transferred. This state is referred to as a black hole connection.
Some implementations of PMTUD attempt to prevent this problem by inferring that large payload packets have been dropped due to MTU rather than because of link congestion. However, in order for the Transmission Control Protocol (TCP) to operate most efficiently, ICMP Unreachable messages (type 3) should be permitted. A robust method for PMTUD that relies on TCP or another protocol to probe the path with progressively larger packets has been standardized in RFC 4821, packetization layer path mtu.
A workaround used by some routers is to change the maximum segment size (MSS) of all TCP connections passing through links with MTU lower than the Ethernet default of 1500. This is known as MSS clamping.
Solution
Manually configure the ptmu (Prefferred)

The interface is different in legacy routed mode(vccap-loopback interface) and cloud mode(the interface which controller connect to fabric )
set interface MTU to 2100
set each outgoing route entry pmtu to 1500
Disable pmtu on sender, "DF" won't be set when pmtu disabled, then router could fragement and forward it.
net.ipv4.ip_no_pmtu_disc = 1 or echo 1 > /proc/sys/net/ipv4/ip_no_pmtu_disc
This only works for ipv4, don't help for ipv6

# References

1. RFC 1191, Path MTU Discovery, J. Mogul, S. Deering (November 1990)
2. RFC 1981, Path MTU Discovery for IP version 6, J. McCann, S. Deering, J. Mogul (August 1996) (obsoleted by RFC 8201)
3. RFC 2923, TCP Problems with Path MTU Discovery, K. Lahey (September 2000)
4. RFC 4821, Packetization Layer Path MTU Discovery, M. Mathis, J. Heffner (March 2007)
5. RFC 7915, IP/ICMP Translation Algorithm, C. Bao, X. Li, F. Baker, T. Anderson, F. Gont (June 2016)
6. RFC 8201, Path MTU Discovery for IP version 6, J. McCann, S. Deering, J. Mogul, R. Hinden, Ed.(July 2017)