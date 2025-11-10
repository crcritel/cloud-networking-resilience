# From Concept to Configuration
### Dual Direct Connect and VPN BGP Example  
**Author:** Cristian Critelli  
**Context:** Chapter 7 – Traffic Engineering for Resilience and Performance  

- **This configuration illustrates how to design a hybrid network using two AWS Direct Connect links (Frankfurt and Paris) with a VPN backup.**
- **Frankfurt is preferred for both inbound and outbound traffic, Paris serves as secondary, and the VPN provides tertiary failover.**

---

## 🧠 Logical Overview

| Location | Role | Preference | BGP Community | AS Path | Notes |
|:--|:--|:--|:--|:--|:--|
| **Frankfurt (DX-1)** | Primary | High (200) | 7224:7300 | Normal | Preferred path for all traffic |
| **Paris (DX-2)** | Secondary | Medium (100) | 7224:7100 | 3× Prepend | Backup path when FRA unavailable |
| **VPN** | Tertiary | N/A | N/A | 3× Prepend | Last-resort failover path |

---

## 🧩 Complete Configuration

```conf
# From Concept to Configuration
# Dual Direct Connect and VPN BGP Example
# Author: Cristian Critelli
# Context: Chapter 7 - Traffic Engineering for Resilience and Performance

!
! BGP Base Configuration
!

router bgp 65000
  bgp log-neighbor-changes

  !
  ! Frankfurt (Primary DX)
  !
  neighbor 192.0.2.1 remote-as 7224
  neighbor 192.0.2.1 description AWS-DX-Frankfurt
  neighbor 192.0.2.1 route-map RM-AWS-FRA out

  !
  ! Paris (Secondary DX)
  !
  neighbor 198.51.100.1 remote-as 7224
  neighbor 198.51.100.1 description AWS-DX-Paris
  neighbor 198.51.100.1 route-map RM-AWS-PAR out

  !
  ! VPN Backup
  !
  neighbor 203.0.113.1 remote-as 7224
  neighbor 203.0.113.1 description AWS-VPN
  neighbor 203.0.113.1 route-map RM-VPN-OUT out

  !
  ! Advertise On-Prem Networks
  !
  address-family ipv4
    network 10.0.0.0 mask 255.255.0.0
    network 10.0.10.0 mask 255.255.255.0
    neighbor 192.0.2.1 activate
    neighbor 198.51.100.1 activate
    neighbor 203.0.113.1 activate
  exit-address-family
!

!
! Route Maps
!

route-map RM-AWS-FRA permit 10
  match ip prefix-list AWS-ALL
  set local-preference 200
  set community 7224:7300
!

route-map RM-AWS-PAR permit 10
  match ip prefix-list AWS-ALL
  set local-preference 100
  set community 7224:7100
  set as-path prepend 65000 65000 65000
!

route-map RM-VPN-OUT permit 10
  match ip prefix-list AWS-ALL
  set as-path prepend 65000 65000 65000
!

!
! Prefix Lists
!

ip prefix-list AWS-ALL seq 5 permit 10.0.0.0/16
ip prefix-list AWS-ALL seq 10 permit 10.0.10.0/24
!

!
! Notes:
! - Frankfurt is preferred outbound (local-pref 200)
! - AWS prefers Frankfurt inbound (community 7224:7300)
! - Paris advertises same prefixes but with lower preference and AS-path prepend
! - VPN used only when both DX connections fail
! - Prefix lists control which networks are advertised to AWS
!
end
```
## ⚠️ Disclaimer

This configuration is provided **for educational and illustrative purposes only**.  
It is a simplified reference example and omits elements essential for production use —  
including **authentication, route filtering, prefix limits, and MD5 session protection**.

Before implementing:
- Replace all **IP addresses, ASNs, and communities** with your environment’s actual values.  
- Test configurations in a **lab or staging environment** first.  

> Neither the author nor Apress Media LLC assumes any responsibility for misconfiguration,  
> downtime, or data loss resulting from direct use of this example.
