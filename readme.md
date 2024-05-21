# Segement Routing IP based non-colored explicit path LSP

Further considerations about non-colored LSP. In case you are bored with using labels in LSP, we can switch segment-list to IP addresses based with a help of auto-translate feature. It will map IPs (stable after failures) with labels (Adj-SID change after failure). Moreover, this path will mimic well known RSPV explicit path. Result path will be fully Adj-SID based.

## Configuration
Most important configuration stranza is to add additional information to TED or will work nothing. At the end of the day SPRING should be aware about label to IP mapping to resolv which address is belongs to Adj-SID labels.

Inject additional data to TED and this should be done on all routers:
```
root@R1> show configuration protocols 
isis {
    traffic-engineering l3-unicast-topology;
}
```

Now we can create our IP based LSP:
```
source-packet-routing {
    segment-list Path-via-IP {
        auto-translate;
        hop1 ip-address 172.16.13.3;
        hop2 ip-address 172.16.35.5;
        hop3 ip-address 172.16.54.4;
        hop4 ip-address 172.16.24.2;
    }
    source-routing-path R2-IP-Detour {
        to 2.0.0.0;
        binding-sid 1000009;
        primary {
            Path-via-IP {
                bfd-liveness-detection {
                    sbfd {              
                        remote-discriminator 2;
                    }
                    minimum-interval 1000;
                    multiplier 3;
                }
            }
        }
    }
```

## Verification

As always check is LSP is in Up state.
```
root@R1> show spring-traffic-engineering lsp detail 
Name: R2-IP-Detour
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 2.0.0.0
  Te-group-id: 0
  State: Up
    Path: Path-via-IP
    Path Status: NA
    Outgoing interface: ge-0/0/1.0
    Auto-translate status: Enabled Auto-translate result: Success
    Compute Status:Disabled , Compute Result:N/A , Compute-Profile Name:N/A
    BFD status: Up BFD name: V4-srte_bfd_session-2
    BFD remote-discriminator: 2(configured)
    Segment ID : 128 
    ERO Valid: true
      SR-ERO hop count: 4
        Hop 1 (Strict): 
          NAI: IPv4 Adjacency ID, 0.0.0.0 -> 172.16.13.3
          SID type: 20-bit label, Value: 18
        Hop 2 (Strict): 
          NAI: IPv4 Adjacency ID, 0.0.0.0 -> 172.16.35.5
          SID type: 20-bit label, Value: 17
        Hop 3 (Strict): 
          NAI: IPv4 Adjacency ID, 0.0.0.0 -> 172.16.54.4
          SID type: 20-bit label, Value: 16
        Hop 4 (Strict): 
          NAI: IPv4 Adjacency ID, 0.0.0.0 -> 172.16.24.2
          SID type: 20-bit label, Value: 18


Total displayed LSPs: 1 (Up: 1, Down: 0)
```

After all, don't forget to check for a new route in inet.3. Note Adj-SID labels in path:
```
root@R1> show route 2 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[IS-IS/18] 00:10:10, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 17, Push 16105(top)

inet.3: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[SPRING-TE/8] 00:10:03, metric 1, metric2 10
                    >  to 172.16.13.3 via ge-0/0/1.0, Push 18, Push 16, Push 17(top)
                    [L-ISIS/14] 00:10:10, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 17, Push 16105(top)
```

## Additional information
Label stack updated with the current labels. LSPs are fault tolerant for real.
