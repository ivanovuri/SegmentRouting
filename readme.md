# Segement Routing Admin Groups calculated LSP

IGP is extended with additional info (link colors, link bw, etc…) that is added to the TED. Then, user defines some constraints like "don't use green colored links". CSPF uses those constraints as input and combines them with TED data to build the lsp. This is simpler and in some cases more extendable that to use explicit path.


## Configuration
As always most important configuration stranza is to add additional information to TED or will work nothing. At the end of the day SPRING should be aware about label to IP mapping to resolv which address is belongs to Adj-SID labels.

Inject additional data to TED and this should be done on all routers. Advertisement always stranza will inject information about link coloring:
```
root@R1> show configuration protocols 
isis {
    traffic-engineering {
        l3-unicast-topology;
        advertisement always;
    }
}
```
Don't forget to add link coloring. At the end of the day, only number bound to name will be propagated into TED so you can use any textual identification.
```
mpls {
    admin-groups {
        Green 2;
    }
    interface all;
    interface ge-0/0/0.0 {
        admin-group Green;
    }
}  
```

Now we can create our IP based LSP. I'll use sBFD and ignore Green links:
```
source-packet-routing {
    compute-profile Avoid-Green {
        admin-group exclude Green;
        maximum-segment-list-depth 16;
        maximum-computed-segment-lists 1;
    }
    source-routing-path R2-AdminGroups {
        to 2.0.0.0;
        binding-sid 1000009;
        primary {
            Avoid-Green {
                bfd-liveness-detection {
                    sbfd {
                        remote-discriminator 2;
                    }
                    minimum-interval 1000;
                    multiplier 3;
                }
                compute {
                    Avoid-Green;
                }
            }
        }                               
    }
}
```

## Verification

As always check is LSP is in Up state.
```
root@R1> show spring-traffic-engineering lsp detail 
Name: R2-AdminGroups
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 2.0.0.0
  Te-group-id: 0
  State: Up
    Path: Avoid-Green
    Path Status: NA
    Outgoing interface: NA
    Auto-translate status: Disabled Auto-translate result: N/A
    Compute Status:Enabled , Compute Result:success , Compute-Profile Name:Avoid-Green
    Total number of computed paths: 1
    Segment ID : 128 
    Computed-path-index: 1
      BFD status: Up BFD name: V4-srte_bfd_session-2
      BFD remote-discriminator: 2(configured)
      TE metric: 150, IGP metric: 150
      Delay metrics: Min: 100663290, Max: 100663290, Avg: 100663290
      Metric optimized by type: TE
      computed segments count: 4
        computed segment : 1 (computed-node-segment): 
          node segment label: 16105
          router-id: 5.0.0.0            
        computed segment : 2 (computed-adjacency-segment): 
          label: 16
          source router-id: 5.0.0.0, destination router-id: 6.0.0.0
          source interface-address: 172.16.56.5, destination interface-address: 172.16.56.6
        computed segment : 3 (computed-node-segment): 
          node segment label: 16107
          router-id: 7.0.0.0
        computed segment : 4 (computed-node-segment): 
          node segment label: 16102
          router-id: 2.0.0.0


Total displayed LSPs: 1 (Up: 1, Down: 0)
```

After all, don't forget to check for a new route in inet.3. Note Adj-SID labels in path:
```
root@R1> show route 2 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[IS-IS/18] 00:18:28, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 16, Push 16105(top)

inet.3: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[SPRING-TE/8] 00:18:14, metric 1, metric2 10
                    >  to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 16107, Push 16, Push 16105(top)
                    [L-ISIS/14] 00:18:28, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 16, Push 16105(top)
```

## Additional information
Now you can see that here I don't use any explicit IP addresses or labels. Such patha are free from Adj-SID, Node-SID or IP address changing and/or configuration.
