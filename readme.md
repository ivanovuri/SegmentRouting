# Segement Routing confguration examples

Segment Routing is one of the latest technologies that really represents something new in core computer networks. This repository is represented by various branches that relate to different topics. Here you see branch where only basic SR routing is configured with several VRFs for demonstrations.

This repository is represented baranch is dedicated to **TI-LFA**. To enable we need to be shure our device more than provided 3 label stack by default. So first of all we need to enable deep label stack with maximum-labels configuration stranza. Only after that we can protect our links with TI-LFA.

## Configuration
Configure basic ISIS with SR enables. SRG range is default directly from Segment Routing book. Here you can see only L2 ISIS domain with wide metrics enabled which is a prerequisite for correct SR operation.
```
isis {
    interface all {
        point-to-point;
    }
    interface lo0.0 {
        passive;
    }
    source-packet-routing {             
        srgb start-label 16000 index-range 8000;
    }
    level 1 disable;
    level 2 wide-metrics-only;
    export Node-SID;
}
/* vMX only stranza to get SR working */
chassis {
    network-services enhanced-ip;
}
``` 
In this lab I'll use policy to confgire Node-SID:
```
policy-options {
    policy-statement Node-SID {
        from {
            interface lo0.0;
            route-filter 1.0.0.0/32 exact;
        }
        then {
            prefix-segment {
                index 101;
                node-segment;
            }
            accept;
        }}}
```
In case you don't want to mess with policies Index-SID can be configured inside protocol isis confgiration section:
```
set protocols isis source-packet-routing node-segment ipv4-index 101
```

## Verification
Check SR is enabled in ISIS. Find SPRING section:
```
root@R1> show isis overview 
Instance: master
  Router ID: 1.0.0.0
  Hostname: R1
  Sysid: 0000.0000.0001
  Areaid: 49.0007
  Adjacency holddown: enabled
  Maximum Areas: 3
  LSP life time: 1200
  Attached bit evaluation: enabled
  SPF delay: 200 msec, SPF holddown: 5000 msec, SPF rapid runs: 3
  IPv4 is enabled, IPv6 is enabled, SPRING based MPLS is enabled
  Traffic engineering: enabled
  Traffic engineering v6: disabled
  Restart: Disabled
    Helper mode: Enabled
  Layer2-map: Disabled
  Source Packet Routing (SPRING): Enabled
    SRGB Config Range :
      SRGB Start-Label : 16000, SRGB Index-Range : 8000
    SRGB Block Allocation: Success
      SRGB Start Index : 16000, SRGB Size : 8000, Label-Range: [ 16000, 23999 ]
    Node Segments: Disabled
    SRv6: Disabled
  Post Convergence Backup: Enabled      
    Max labels: 3, Max spf: 100, Max Ecmp Backup: 1
  Level 1
    Internal route preference: 15
    External route preference: 160
    Prefix export count: 0
    Wide metrics are enabled, Narrow metrics are enabled
    Source Packet Routing is enabled
  Level 2
    Internal route preference: 18
    External route preference: 165
    Prefix export count: 0
    Wide metrics are enabled
    Source Packet Routing is enabled
```
New information should be presend inside ISIS database. Find IPV4 Index, SPRING and Adj-SID sections:
```
root@R1> show isis database extensive R7 
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

R7.00-00 Sequence: 0x7, Checksum: 0xa1c6, Lifetime: 686 secs
  IPV4 Index: 107
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 8000, Label-Range: [ 16000, 23999 ]
   IS neighbor: R2.00                         Metric:       10
     Two-way fragment: R2.00-00, Two-way first fragment: R2.00-00
     P2P IPv4 Adj-SID:      16, Weight:   0, Flags: --VL--
   IS neighbor: R4.00                         Metric:       10
     Two-way fragment: R4.00-00, Two-way first fragment: R4.00-00
     P2P IPv4 Adj-SID:      17, Weight:   0, Flags: --VL--
   IP prefix: 7.0.0.0/32                      Metric:        0 Internal Up
   IP prefix: 172.16.27.0/27                  Metric:       10 Internal Up
   IP prefix: 172.16.47.0/27                  Metric:       10 Internal Up

  Header: LSP ID: R7.00-00, Length: 213 bytes
    Allocated length: 284 bytes, Router ID: 7.0.0.0
    Remaining lifetime: 686 secs, Level: 2, Interface: 336
    Estimated free bytes: 115, Actual free bytes: 71
    Aging timer expires in: 686 secs
    Protocols: IP, IPv6                 

  Packet: LSP ID: R7.00-00, Length: 213 bytes, Lifetime : 1196 secs
    Checksum: 0xa1c6, Sequence: 0x7, Attributes: 0x3 <L1 L2>
    NLPID: 0x83, Fixed length: 27 bytes, Version: 1, Sysid length: 0 bytes
    Packet type: 20, Packet version: 1, Max area: 0

  TLVs:
    Area address: 49.0007 (3)
    LSP Buffer Size: 1492
    Speaks: IP
    Speaks: IPV6
    IP router id: 7.0.0.0
    IP address: 7.0.0.0
    Hostname: R7
    IP extended prefix: 7.0.0.0/32 metric 0 up
      8 bytes of subtlvs
      Node SID, Flags: 0x40(R:0,N:1,P:0,E:0,V:0,L:0), Algo: SPF(0), Value: 107
    IP extended prefix: 172.16.27.0/27 metric 10 up
    IP extended prefix: 172.16.47.0/27 metric 10 up
    Router Capability:  Router ID 7.0.0.0, Flags: 0x00
      SPRING Capability - Flags: 0xc0(I:1,V:1), Range: 8000, SID-Label: 16000
      SPRING Algorithm - Algo: 0
      SPRING Algorithm - Algo: 1        
      Node MSD Advertisement Sub-TLV:Type:  23, Length: 4
        Base MPLS Imposition MSD:Type:  1, Value: 15
        Entropy Readable Label Depth MSD:Type:  2, Value: 16
    Extended IS Reachability TLV, Type: 22, Length: 88
    IS extended neighbor: R2.00, Metric: default 10 SubTLV len: 33
      IP address: 172.16.27.7
      Neighbor's IP address: 172.16.27.2
      Local interface index: 334, Remote interface index: 336
      Link MSD Advertisement Sub-TLV:Type: 15, Length: 2
        Base MPLS Imposition MSD:Type: 1, Value: 15
      P2P IPV4 Adj-SID - Flags:0x30(F:0,B:0,V:1,L:1,S:0,P:0), Weight:0, Label: 16
      P2P IPv4 Adj-SID:      16, Weight:   0, Flags: --VL--
    IS extended neighbor: R4.00, Metric: default 10 SubTLV len: 33
      IP address: 172.16.47.7
      Neighbor's IP address: 172.16.47.4
      Local interface index: 337, Remote interface index: 337
      Link MSD Advertisement Sub-TLV:Type: 15, Length: 2
        Base MPLS Imposition MSD:Type: 1, Value: 15
      P2P IPV4 Adj-SID - Flags:0x30(F:0,B:0,V:1,L:1,S:0,P:0), Weight:0, Label: 17
      P2P IPv4 Adj-SID:      17, Weight:   0, Flags: --VL--
  No queued transmissions 
```

## Additional information
The idea behind SR is to compose and stack those segments to create a wider path. Here no paths exists all othe topics will be covered in different branches inside this repo.