# Segement Routing non-colored explicit path LSP

Time to look at non-colored LSP. No controller so I will use the manual approach where we configure anything via CLI. Here I'll use sBFP so we need to make shure sbfd local-discriminator the same on both sides.

## Configuration

The only routers where we need to configure something is headend router R1 and destination R2. Here I'll use Adj-SID to steer traffic via less preferred path. Adj-SID will not preserve between reboots, so please make sure you use the correct numbers. Also, lets use sBFD to speedup in case of failures.

Headend confgiration on R1
```
root@R1# show protocols 
source-packet-routing {
    segment-list Path-via-300-and-R7 {
        hop1 ip-address 172.16.13.3;
        hop2 label 16105;
        hop3 label 17;
        hop4 label 16107;
        hop5 label 16102;
    }
    segment-list Path-via-100-and-R8 {
        hop1 ip-address 172.16.13.3;    
        hop2 label 16105;
        hop3 label 16;
        hop4 label 16108;
        hop5 label 16102;
    }
    source-routing-path R2-Detour {
        to 2.0.0.0;
        binding-sid 1000008;
        primary {
            Path-via-300-and-R7 {
                bfd-liveness-detection {
                    sbfd {
                        remote-discriminator 2;
                    }
                    minimum-interval 1000;
                    multiplier 3;
                }
            }
        }
        secondary {
            Path-via-100-and-R8 {
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
}
```

Tailend configuration R2. Nothing special needed but sBFP parameter:
```
root@R2> show configuration protocols 
bfd {
    sbfd {
        local-discriminator 2;
    }
}
```

## Verification

First of all check is LSP is in Up state.
```
[edit]
root@R1# run show spring-traffic-engineering lsp          
To              State     LSPname
2.0.0.0         Up        R2-Detour


Total displayed LSPs: 1 (Up: 1, Down: 0)
```

Same LSP but much more detailed than above.
```
[edit]
root@R1# run show spring-traffic-engineering lsp detail 
Name: R2-Detour
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 2.0.0.0
  Te-group-id: 0
  State: Up
    Path: Path-via-300-and-R7
    Path Status: NA
    Outgoing interface: ge-0/0/1.0
    Auto-translate status: Disabled Auto-translate result: N/A
    Compute Status:Disabled , Compute Result:N/A , Compute-Profile Name:N/A
    BFD status: Up BFD name: V4-srte_bfd_session-7
    BFD remote-discriminator: 2(configured)
    Segment ID : 128 
    ERO Valid: true
      SR-ERO hop count: 5
        Hop 1 (Strict): 
          NAI: IPv4 Adjacency ID, 0.0.0.0 -> 172.16.13.3
          SID type: None
        Hop 2 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16105
        Hop 3 (Strict): 
          NAI: None                     
          SID type: 20-bit label, Value: 17
        Hop 4 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16107
        Hop 5 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16102
    Path: Path-via-100-and-R8
    Path Status: NA
    Outgoing interface: ge-0/0/1.0
    Auto-translate status: Disabled Auto-translate result: N/A
    Compute Status:Disabled , Compute Result:N/A , Compute-Profile Name:N/A
    BFD status: Up BFD name: V4-srte_bfd_session-8
    BFD remote-discriminator: 2(configured)
    Segment ID : 256 
    ERO Valid: true
      SR-ERO hop count: 5
        Hop 1 (Strict): 
          NAI: IPv4 Adjacency ID, 0.0.0.0 -> 172.16.13.3
          SID type: None
        Hop 2 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16105
        Hop 3 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16
        Hop 4 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16108
        Hop 5 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 16102


Total displayed LSPs: 1 (Up: 1, Down: 0)
```

Also sBFS should be in Up state too. In our case, we should see two sessions:
```
[edit]
root@R1# run show spring-traffic-engineering sbfd           
BFDname                  State
V4-srte_bfd_session-7    Up       
V4-srte_bfd_session-8    Up    
```

After all, don't forget to check for a new route in inet.3.
```
root@R1# run show route 2 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[IS-IS/18] 00:14:33, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 16, Push 16105(top)

inet.3: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[SPRING-TE/8] 00:00:26, metric 1, metric2 10
                    >  to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 16107, Push 17, Push 16105(top)
                       to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 16108, Push 16, Push 16105(top)
                    [L-ISIS/14] 00:14:33, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 16, Push 16105(top)
```

## Additional information
Keep in mind that sBFD protects only in case Node failures or if link with Adj-SID goes down. Only after that sBFP session goes down and steers traffic to secondary path.