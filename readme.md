# Segement Routing TI-LFA

This baranch is dedicated to **TI-LFA**. To enable we need to be shure our device more than provided 3 label stack by default. So first of all we need to enable deep label stack with maximum-labels configuration stranza. Only after that we can protect our links with TI-LFA.

## Configuration
Configure all mpls enabled interfaces with maximum-labels of 16.
```
show configuration interfaces             
ge-0/0/0 {
    unit 0 {
        family mpls {
            maximum-labels 16;
        }}}
ge-0/0/1 {
    unit 0 {
        family mpls {
            maximum-labels 16;
        }}}
```
Then confgure backup-spf-option and post-convergence-lfa inside ISIS:
```
isis {
    interface ge-0/0/0.0 {
        level 2 {
            post-convergence-lfa;
        }
        point-to-point;
    backup-spf-options {
        use-post-convergence-lfa;
        use-source-packet-routing;
    }
}
```

## Verification
Check maximum labels on interface:
```
root@R1> show mpls interface detail 
Interface: ge-0/0/0.0
  State: Up
  Administrative group: <none>
  Maximum labels: 16
  Static protection revert time: 5 seconds
  Always mark connection protection tlv: Disabled
  Switch away lsps : Disabled
Interface: ge-0/0/1.0
  State: Up
  Administrative group: <none>
  Maximum labels: 16
  Static protection revert time: 5 seconds
  Always mark connection protection tlv: Disabled
  Switch away lsps : Disabled
```
Verify pre-computed a secondary path across the protected resource
```
root@R1> show route 2 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[IS-IS/18] 00:12:02, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 17, Push 16105(top)

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2.0.0.0/32         *[L-ISIS/14] 00:12:02, metric 10
                    >  to 172.16.12.2 via ge-0/0/0.0
                       to 172.16.13.3 via ge-0/0/1.0, Push 16102, Push 17, Push 16105(top)

root@R1> show route label 17 

mpls.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[L-ISIS/14] 00:13:33, metric 0
                    >  to 172.16.12.2 via ge-0/0/0.0, Pop      
                       to 172.16.13.3 via ge-0/0/1.0, Swap 16102, Push 17, Push 16105(top)
17(S=0)            *[L-ISIS/14] 00:13:33, metric 0
                    >  to 172.16.12.2 via ge-0/0/0.0, Pop      
                       to 172.16.13.3 via ge-0/0/1.0, Swap 16102, Push 17, Push 16105(top)
```

## Additional information
TI-LFA is possible because SR changes the paradigm behind forwarding; no longer destination based but source based but as always look at router configuration. Try to build your own lab and practice.