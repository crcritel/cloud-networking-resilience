# Pattern 2 — Inbound (Ingress) Steering by Measured Performance

**Author:** Cristian Critelli  
**Context:** Chapter 7 – Traffic Engineering for Resilience and Performance  

- **We influence that behavior using AWS Direct Connect BGP communities (to indicate route preference) and AS-path prepending (to make a route look less attractive).**
- **We prefer the lower-latency **Direct Connect** when healthy and automatically fall back (or de-prefer) when latency increases persistently.**

- **This pattern measures the health of the AWS return path and dynamically adjusts the advertisements we send to AWS — ensuring inbound traffic prefers the healthier DX link.**

---

## 🧩 Complete Example Configuration

```conf
! 1) Measure RTT/loss to a representative on-prem <-> AWS return-path target
ip sla 20
 icmp-echo 52.95.255.1 source-interface GigabitEthernet0/1
 frequency 5
 timeout 1000
ip sla reaction-configuration 20 react rtt threshold-type consecutive
  threshold-value 120 3
  action-type syslog
ip sla schedule 20 life forever start-time now

! 2) Prefixes we advertise from on-prem to AWS
ip prefix-list ONPREM-EXPORT seq 5 permit 192.168.0.0/16
ip prefix-list ONPREM-EXPORT seq 10 permit 172.16.0.0/12

! 3) Two outbound advertisement policies:
!    PRIMARY: high-pref community, no prepend
!    DEGRADED: low-pref community, AS-path prepends
route-map DX_PRIMARY_ADVERTISE permit 10
 match ip address prefix-list ONPREM-EXPORT
 set community 7224:7300 additive
! (7224:7300 is AWS "high" preference; verify current docs before use)

route-map DX_DEGRADED_ADVERTISE permit 10
 match ip address prefix-list ONPREM-EXPORT
 set community 7224:7100 additive
 set as-path prepend 65000 65000 65000

router bgp 65000
 neighbor 169.254.100.1 remote-as 64512
 neighbor 169.254.100.1 description DX Transit VIF – AWS
 !
 ! Start by advertising with HIGH preference
 neighbor 169.254.100.1 route-map DX_PRIMARY_ADVERTISE out

! 4) EEM flips advertisements on impairment/recovery
event manager applet INGRESS_DEGRADE
 event syslog pattern "RTTMON.*ThresholdExceeded.*rtt" maxrun 60
 action 1.0 cli command "enable"
 action 1.1 cli command "configure terminal"
 action 1.2 cli command "router bgp 65000"
 action 1.3 cli command "neighbor 169.254.100.1 route-map DX_DEGRADED_ADVERTISE out"
 action 1.4 cli command "end"
 action 1.5 syslog msg "EEM: Ingress de-preferred (low community + prepends)"

event manager applet INGRESS_RECOVER
 event syslog pattern "RTTMON.*ThresholdFalling.*rtt" maxrun 60
 action 1.0 cli command "enable"
 action 1.1 cli command "configure terminal"
 action 1.2 cli command "router bgp 65000"
 action 1.3 cli command "neighbor 169.254.100.1 route-map DX_PRIMARY_ADVERTISE out"
 action 1.4 cli command "end"
 action 1.5 syslog msg "EEM: Ingress restored (high community, no prepends)"
```

## ⚙️ Summary of Behavior

| **Condition** | **RTT State** | **Advertised Policy** | **Community / AS-Path** | **AWS Action** |
|:--|:--|:--|:--|:--|
| 🟢 **Normal** | RTT < 120 ms | `DX_PRIMARY_ADVERTISE` | 7224:7300 (High Preference) | AWS prefers this DX link for inbound traffic |
| 🟠 **Degraded** | RTT > 120 ms for 3 consecutive probes | `DX_DEGRADED_ADVERTISE` | 7224:7100 (Low Preference) + AS-Path Prepend × 3 | AWS shifts inbound traffic to alternate site/link |
| 🔵 **Recovered** | RTT < 120 ms for 3 consecutive probes | `DX_PRIMARY_ADVERTISE` | 7224:7300 (High Preference) | AWS restores inbound preference to this link |
| 🔴 **Hard Failure** | BFD/BGP detects link loss | — | — | AWS withdraws route and reroutes traffic automatically |

## ⚠️ Disclaimer

This configuration is provided **for educational and illustrative purposes only**.  
It is a simplified reference example and omits elements essential for production use —  
including **authentication, route filtering, prefix limits, and MD5 session protection**.

Before implementing:
- Replace all **IP addresses, ASNs, and communities** with your environment’s actual values.  
- Test configurations in a **lab or staging environment** first.  

> Neither the author nor Apress Media LLC assumes any responsibility for misconfiguration,  
> downtime, or data loss resulting from direct use of this example.
