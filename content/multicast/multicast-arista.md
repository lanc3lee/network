
```
lance@LANC3 ~ % mkdir -p /Users/lance/Documents/multicast-arista01
lance@LANC3 ~ % cd /Users/lance/Documents/multicast-arista01
lance@LANC3 multicast-arista01 % nano topology.clab.yml
```

topology is :`h1 — r1 — r3 — r2 — h2` 

```
containerlab graph --mermaid -t topology.clab.yml
```
![[multicast-arista-mermaid.png]]

```
name: multicast-lab-allceos

topology:
  nodes:
    r1:
      kind: ceos
      image: ceos:4.33.9M
    r2:
      kind: ceos
      image: ceos:4.33.9M
    r3:
      kind: ceos
      image: ceos:4.33.9M

    h1:
      kind: linux
      image: praqma/network-multitool:latest
      exec:
        - ip route add 224.0.0.0/4 dev eth1
        - apk add --no-cache iperf tcpdump

    h2:
      kind: linux
      image: praqma/network-multitool:latest
      exec:
        - ip route add 224.0.0.0/4 dev eth1
        - apk add --no-cache iperf tcpdump

  links:
    - endpoints: ["h1:eth1", "r1:eth1"]
    - endpoints: ["r1:eth2", "r3:eth1"]
    - endpoints: ["r3:eth2", "r2:eth1"]
    - endpoints: ["r2:eth2", "h2:eth1"]
```




![[multicast-aristalabs01-allcEOS.png]]

```
containerlab inspect -t topology.clab.yml

```

![[multicast-aristalabs01-inspect-topology.png]]






![[multicast-r1-r2-r3-sh-ip-pim-nei.png]]




```
r1#show ip pim rp

Group: 224.0.0.0/4

  RP: 10.100.0.1

    Uptime: 0:03:37, Expires: never, Priority: 0, Override: False

r1#sh ip pim nei

PIM Neighbor Table for default VRF

Neighbor Address  Interface  Uptime    Expires   Mode    Transport  

10.100.13.3       Ethernet2  00:07:54  00:01:22  sparse  datagram

r1#

r2#show ip pim rp

Group: 224.0.0.0/4

  RP: 10.100.0.1

    Uptime: 0:02:58, Expires: never, Priority: 0, Override: False

r2#sh ip pim nei

PIM Neighbor Table for default VRF

Neighbor Address  Interface  Uptime    Expires   Mode    Transport  

10.100.23.3       Ethernet1  00:08:02  00:01:44  sparse  datagram

r2#

r3#

sh ip ospfr3#sh ip ospf nei

Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface

10.100.0.2      1        default  1   FULL/DR                00:00:37    10.100.23.2     Ethernet2

10.100.0.1      1        default  1   FULL/DR                00:00:33    10.100.13.1     Ethernet1

r3#show ip pim rp

Group: 224.0.0.0/4

  RP: 10.100.0.1

    Uptime: 0:03:34, Expires: never, Priority: 0, Override: False

r3#sh ip pim nei

PIM Neighbor Table for default VRF

Neighbor Address  Interface  Uptime    Expires   Mode    Transport  

10.100.13.1       Ethernet1  00:07:44  00:01:30  sparse  datagram   

10.100.23.2       Ethernet2  00:07:42  00:01:34  sparse  datagram

r3#

lance@clab:/Users/lance$ docker exec -it clab-multicast-lab-allceos-h2 sh

/ # ip addr add 10.100.20.20/24 dev eth1

/ # 

/ # iperf -s -u -B 239.192.1.1 -i 1

------------------------------------------------------------

Server listening on UDP port 5001

Joining multicast group  239.192.1.1

Server set to single client traffic mode (per multicast receive)

UDP buffer size:  224 KByte (default)

------------------------------------------------------------

lance@clab:/Users/Lance/Documents/multicast-arista01$ docker exec -it clab-multicast-lab-allceos-h1 sh

/ # ip addr add 10.100.12.10/24 dev eth1

/ # 

/ # iperf -c 239.192.1.1 -u -t 20 -i 1

------------------------------------------------------------

Client connecting to 239.192.1.1, UDP port 5001

Sending 1470 byte datagrams, IPG target: 11215.21 us (kalman adjust)

UDP buffer size:  224 KByte (default)

------------------------------------------------------------

[  1] local 10.100.12.10 port 60895 connected with 239.192.1.1 port 5001

[ ID] Interval       Transfer     Bandwidth

[  1] 0.00-1.00 sec   131 KBytes  1.07 Mbits/sec

[  1] 1.00-2.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 2.00-3.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 3.00-4.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 4.00-5.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 5.00-6.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 6.00-7.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 7.00-8.00 sec   129 KBytes  1.06 Mbits/sec

[  1] 8.00-9.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 9.00-10.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 10.00-11.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 11.00-12.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 12.00-13.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 13.00-14.00 sec   129 KBytes  1.06 Mbits/sec

[  1] 14.00-15.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 15.00-16.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 16.00-17.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 17.00-18.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 18.00-19.00 sec   129 KBytes  1.06 Mbits/sec

[  1] 19.00-20.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 0.00-20.02 sec  2.51 MBytes  1.05 Mbits/sec

[  1] Sent 1788 datagrams

/ #
```


```
Sat Aug 15 07:55:36 UTC 2026

/ # 

/ # iperf -c 239.192.1.1 -u -t 10 -i 1

------------------------------------------------------------

Client connecting to 239.192.1.1, UDP port 5001

Sending 1470 byte datagrams, IPG target: 11215.21 us (kalman adjust)

UDP buffer size:  224 KByte (default)

------------------------------------------------------------

[  1] local 10.100.12.10 port 37919 connected with 239.192.1.1 port 5001

[ ID] Interval       Transfer     Bandwidth

[  1] 0.00-1.00 sec   131 KBytes  1.07 Mbits/sec

[  1] 1.00-2.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 2.00-3.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 3.00-4.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 4.00-5.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 5.00-6.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 6.00-7.00 sec   129 KBytes  1.06 Mbits/sec

[  1] 7.00-8.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 8.00-9.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 9.00-10.00 sec   128 KBytes  1.05 Mbits/sec

[  1] 0.00-10.02 sec  1.25 MBytes  1.05 Mbits/sec

[  1] Sent 896 datagrams

/ # 

r1#show interfaces Ethernet2 | include packets

  5 minutes input rate 177 bps (0.0% with framing overhead), 0 packets/sec

  5 minutes output rate 0 bps (0.0% with framing overhead), 0 packets/sec

     391 packets input, 43792 bytes

     0 packets output, 0 bytes

r1#show clock

Sat Aug 15 07:55:16 2026

Timezone: UTC

Clock source: local

r1#show interfaces Ethernet2 | include packets

  5 minutes input rate 173 bps (0.0% with framing overhead), 0 packets/sec

  5 minutes output rate 0 bps (0.0% with framing overhead), 0 packets/sec

     411 packets input, 45923 bytes

     0 packets output, 0 bytes

r1#

r2#

r2#show interfaces Ethernet2 | include packets

  5 minutes input rate 4 bps (0.0% with framing overhead), 0 packets/sec

  5 minutes output rate 0 bps (0.0% with framing overhead), 0 packets/sec

     12 packets input, 744 bytes

     0 packets output, 0 bytes

r2#

r3#show interfaces Ethernet2 | include packets

  5 minutes input rate 176 bps (0.0% with framing overhead), 0 packets/sec

  5 minutes output rate 0 bps (0.0% with framing overhead), 0 packets/sec

     392 packets input, 43010 bytes

     0 packets output, 0 bytes

r3#show clock

Sat Aug 15 07:55:27 2026

Timezone: UTC

Clock source: local

r3#

r3#show interfaces Ethernet2 | include packets

  5 minutes input rate 170 bps (0.0% with framing overhead), 0 packets/sec

  5 minutes output rate 0 bps (0.0% with framing overhead), 0 packets/sec

     411 packets input, 45039 bytes

     0 packets output, 0 bytes

r3#
```