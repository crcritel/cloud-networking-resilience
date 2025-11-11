# Pattern 1 — Outbound (Egress) Steering by Measured Performance

**Author:** Cristian Critelli  
**Context:** Chapter 7 – Traffic Engineering for Resilience and Performance  

- **Outbound is ours to control.**
- **We prefer the lower-latency **Direct Connect** when healthy and automatically fall back (or de-prefer) when latency increases persistently.**

- **This pattern demonstrates how to **measure**, **react**, and **adapt BGP attributes** to maintain optimal egress routing performance — using IP SLA, EEM, and dynamic route-map switching.**

---

### ⚙️ Configuration Disclaimer

> **Note:** The configuration below is **illustrative only (non-production)**.  
> Exact commands, syslog patterns, timers, and neighbor policy hooks may vary depending on your router platform and software version.  
> Always validate in a **lab environment** and follow your organization’s **change control procedures** before deployment.

---

## 🧩 Complete Example Configuration

```conf
! 1) Measure RTT to an AWS-side IP that reflects the DX path
ip sla 10
 icmp-echo 52.95.255.1 source-interface GigabitEthernet0/0
 frequency 5
 timeout 1000
! Define an "excess RTT" threshold at 100 ms and generate events
ip sla reaction-configuration 10 react rtt threshold-type consecutive
  threshold-value 100 3  ! >100ms for 3 consecutive probes
  action-type syslog

ip sla schedule 10 life forever start-time now

! 2) Track hard reachability (up/down)
track 10 ip sla 10 reachability

! 3) BGP policy: two route-maps we will toggle between
ip prefix-list AWS-PFX seq 5 permit 10.0.0.0/8 le 24

route-map DX_PRIMARY_EGRESS permit 10
 match ip address prefix-list AWS-PFX
 set local-preference 200

route-map DX_DEGRADED_EGRESS permit 10
 match ip address prefix-list AWS-PFX
 set local-preference 100
! (Optionally) discourage equal-cost forwarding
! set weight 0

router bgp 65000
 neighbor 169.254.100.1 remote-as 64512
 neighbor 169.254.100.1 description DX Transit VIF – AWS
 !
 ! Start in "primary" mode
 neighbor 169.254.100.1 route-map DX_PRIMARY_EGRESS in
!
! 4) React to RTT threshold crossings using EEM (syslog-triggered)
event manager applet RTT_HIGH
 event syslog pattern "RTTMON.*ThresholdExceeded.*rtt" maxrun 60
 action 1.0 cli command "enable"
 action 1.1 cli command "configure terminal"
 action 1.2 cli command "router bgp 65000"
 action 1.3 cli command "neighbor 169.254.100.1 route-map DX_DEGRADED_EGRESS in"
 action 1.4 cli command "end"
 action 1.5 syslog msg "EEM: Switched to DEGRADED egress due to high RTT"

event manager applet RTT_RECOVERED
 event syslog pattern "RTTMON.*ThresholdFalling.*rtt" maxrun 60
 action 1.0 cli command "enable"
 action 1.1 cli command "configure terminal"
 action 1.2 cli command "router bgp 65000"
 action 1.3 cli command "neighbor 169.254.100.1 route-map DX_PRIMARY_EGRESS in"
 action 1.4 cli command "end"
 action 1.5 syslog msg "EEM: Restored PRIMARY egress after RTT recovery"

! 5) Hard failure? Let BFD/BGP handle fast withdrawal; track can still warn.
router bgp 65000
 neighbor 169.254.100.1 bfd
!
end
```

## ⚙️ Summary of Behavior

| Condition | RTT State | Active Route Map | Preferred Path | Action |
|:--|:--|:--|:--|:--|
| **Normal** | < 100 ms | `DX_PRIMARY_EGRESS` | Direct Connect (Primary) | Normal operation |
| **Degraded** | > 100 ms (3×) | `DX_DEGRADED_EGRESS` | Backup (Secondary DX or VPN) | Local-pref reduced |
| **Recovered** | < 100 ms (3×) | `DX_PRIMARY_EGRESS` | Direct Connect restored | Route-map reverted |
| **Hard Failure** | Link Down | — | Alternate path | BFD triggers fast failover |
