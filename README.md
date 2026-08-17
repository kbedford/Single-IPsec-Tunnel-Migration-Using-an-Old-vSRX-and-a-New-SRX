# Phase 1 IPsec Migration MoP — Single Tunnel from Old vSRX to New SRX

## 1. Purpose

Validate the migration of one IPsec tunnel (`CE1`) from `OLD-VSRX-GW` to `NEW-SRX-GW` without changing the CE configuration or the public IPsec gateway address.

`CE2` is the control tunnel. It remains terminated on `OLD-VSRX-GW` throughout CE1 migration and rollback.

The PoC implements the high-level method from the PX IPsec migration MoP:

1. The shared IPsec endpoint becomes reachable through the new gateway.
2. The new gateway temporarily forwards traffic from unmigrated peer addresses to the old gateway through IPBB.
3. Removing one peer address from the old-peer list causes that peer to terminate locally on the new gateway.
4. Adding the peer back provides individual rollback.

This is a functional migration PoC. It deliberately avoids production-scale routing, BGP MED, MPLS, GRE, AH, NAT-T, high availability and batch migration.

### 1.1 Read first — exact sequence that moved CE1 successfully

The tunnel is selected by three coordinated controls. They must be changed in the order shown below.

| Control | Node | Purpose |
|---|---|---|
| Shared endpoint route `203.0.113.100/32` | `AS1273-EDGE` | Attract all peer IKE/ESP traffic to `NEW-SRX-GW` |
| `OLD-PEERS` plus active/inactive CE1 IKE/VPN objects | `NEW-SRX-GW` | Decide whether CE1 is forwarded to old or terminated locally |
| Protected route `10.1.1.0/24` | `IPBB-CORE` | Select the gateway used for server-to-CE1 return traffic |

Never configure the protected route on `AS1273-EDGE`. AS1273 owns only the shared endpoint route; `IPBB-CORE` owns `10.1.1.0/24` and `10.2.2.0/24`.

Before cutover, the required staging state is:

- AS1273 already sends `203.0.113.100/32` to `192.0.2.3`, the new SRX.
- CE1 and CE2 are both active in `OLD-PEERS` and terminate on `OLD-VSRX-GW`.
- `CE1-GATEWAY` and `CE1-VPN` are preconfigured but inactive on `NEW-SRX-GW`; otherwise the new SRX can create a local self-tunnel session and steal CE1 traffic during staging.
- Both IPBB protected routes still point to the old gateway, `10.44.0.0`.
- Continuous CE1 and CE2 protected pings are running.

Execute the single-tunnel migration as follows.

1. **Record the current old CE1 indexes.** On `OLD-VSRX-GW`, identify the IKE index and IPsec ID for peer `198.51.100.0`. Mark the CE2 indexes as **do not clear**.
2. **Enable local CE1 termination and remove CE1 from the trombone in one new-SRX commit.**

   ```text
   NEW-SRX-GW> configure
   NEW-SRX-GW# activate security ike gateway CE1-GATEWAY
   NEW-SRX-GW# activate security ipsec vpn CE1-VPN
   NEW-SRX-GW# deactivate policy-options prefix-list OLD-PEERS 198.51.100.0/32
   NEW-SRX-GW# commit confirmed 10 comment "Migrate CE1 to new SRX"
   ```

3. **After that commit, clear CE1's cached decision on `NEW-SRX-GW`.** Clearing before the commit allows the old transit session to be recreated.

   ```text
   NEW-SRX-GW> clear security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32
   ```

4. **Immediately clear only the old CE1 IKE SA.**

   ```text
   OLD-VSRX-GW> clear security ike security-associations index <CURRENT-CE1-OLD-IKE-INDEX>
   ```

   If the old CE1 child SA remains without its parent IKE SA, verify its VPN/peer and clear only that IPsec ID:

   ```text
   OLD-VSRX-GW> clear security ipsec security-associations index <CE1-OLD-IPSEC-ID>
   ```

5. **Verify both Phase 1 and Phase 2 on the new SRX.** Do not move protected routing until one inbound and one outbound CE1 IPsec SA are installed. `CE1-VPN` includes `establish-tunnels immediately` so Quick Mode is triggered without a CE change.
6. **Move the CE1 protected return route on `IPBB-CORE`, not AS1273.**

   ```text
   IPBB-CORE> configure
   IPBB-CORE# delete routing-options static route 10.1.1.0/24
   IPBB-CORE# set routing-options static route 10.1.1.0/24 next-hop 10.44.0.2
   IPBB-CORE# commit confirmed 10 comment "Move CE1 protected route to new SRX"
   ```

7. **Validate and confirm.** CE1 must pass through `NEW-SRX-GW`; CE2 must continue passing through `OLD-VSRX-GW`. Confirm every pending commit before its timer expires.

The short form is:

```text
activate new CE1 IKE/VPN + remove CE1 from OLD-PEERS
        -> commit confirmed
        -> clear CE1 flow on NEW
        -> clear CE1 IKE on OLD
        -> verify new IKE and IPsec
        -> move 10.1.1.0/24 on IPBB
        -> test CE1 and CE2
        -> confirm commits
```

### 1.2 Non-negotiable change-control rules

- Keep continuous server-to-CE1 and server-to-CE2 pings running from staging onward.
- Use `commit confirmed 10` for endpoint routing, peer steering, gateway activation and protected-route changes.
- Commit the new-SRX state before clearing its CE1 flow session.
- Never clear all IKE, IPsec or flow sessions. Match CE1 by verified peer, index or endpoint pair.
- Never clear CE2. CE2 is the control tunnel and any CE2 loss is a stop-and-rollback condition.
- Never move the IPBB protected route until `NEW-SRX-GW` shows installed inbound and outbound CE1 IPsec SAs.
- A CE-side SA clear is diagnostic recovery only and is not part of the normal no-CE-change migration.

## 2. Agreed lab design

The lab uses two vSRX gateways:

The graphical lab node may still be labelled `OLD-GW_SRX`; configure its Junos hostname as `OLD-VSRX-GW` so the operational commands in this document remain consistent.

| Node | Phase 1 role |
|---|---|
| `CE1` | Tunnel being migrated; no CE configuration changes during migration |
| `CE2` | Control tunnel; remains on the old gateway |
| `AS1273-EDGE` | Represents Internet routing toward the shared IPsec endpoint |
| `OLD-VSRX-GW` | Existing functional IPsec gateway emulator for CE1 and CE2 |
| `NEW-SRX-GW` | New gateway; attracts the shared endpoint and applies temporary peer steering |
| `IPBB-CORE` | Provides the temporary path from new gateway to old gateway and selects the protected-side return gateway |
| `Server` | Protected-side Linux test host |

Important implementation decisions:

- Both gateway devices use SRX `security ike`, `security ipsec` and `st0` configuration.
- Both gateway devices use the same `203.0.113.100/32` IPsec endpoint on `lo0.0`.
- Do not create `lo0.100`. Junos permits only one loopback logical unit in the master routing instance; add the shared address to `lo0.0`.
- On each gateway, the AS1273-facing interface, IPBB-facing interface and `lo0.0` are placed in one `TRANSPORT` zone. This allows the loopback IKE endpoint to receive traffic directly from AS1273 or through the temporary IPBB trombone.
- AS1273 static routing represents the production MED change.
- An `OLD-PEERS` prefix list on `NEW-SRX-GW` represents the MoP peer list.
- A forwarding instance on `NEW-SRX-GW` sends listed peers through `IPBB-CORE` to `OLD-VSRX-GW`.
- The new CE1 IKE gateway and VPN are preconfigured but inactive during staging. The lab proved that leaving the CE1 IKE gateway active can create a local `.local..0`/`lo0.0` self-tunnel session even while CE1 is listed in `OLD-PEERS`. Both objects are activated in the same commit that removes CE1 from `OLD-PEERS`.
- The production SRX4120 supports FBF from Junos 25.2R1. The lab vSRX must pass the FBF capability gate before migration testing.

## 3. Traffic states

### 3.1 Initial baseline

```text
CE1/CE2 -> AS1273-EDGE -> OLD-VSRX-GW -> IPBB-CORE -> Server
```

AS1273 routes `203.0.113.100/32` directly to `OLD-VSRX-GW`.

### 3.2 MoP staging state

```text
CE1/CE2 -> AS1273-EDGE -> NEW-SRX-GW
                                  |
                                  | OLD-PEERS match
                                  v
                            IPBB-CORE -> OLD-VSRX-GW
```

AS1273 sends the shared endpoint to `NEW-SRX-GW`, but both CE peer addresses are initially in `OLD-PEERS`. The new CE1 IKE gateway and VPN are inactive, so the existing tunnels remain terminated on `OLD-VSRX-GW` without creating a competing local CE1 self-session.

### 3.3 CE1 migrated state

```text
CE1 -> AS1273-EDGE -> NEW-SRX-GW -> local CE1 IPsec termination

CE2 -> AS1273-EDGE -> NEW-SRX-GW -> IPBB-CORE -> OLD-VSRX-GW
```

Only CE1 is removed from `OLD-PEERS`. CE2 proves that steering and rollback are customer-specific.

## 4. Addressing plan

### 4.1 Underlay and shared endpoint

| Link/function | Node/interface | Address |
|---|---|---|
| CE1—AS1273 | CE1 `ge-0/0/0.0` | `198.51.100.0/31` |
| CE1—AS1273 | AS1273 `ge-0/0/0.0` | `198.51.100.1/31` |
| CE2—AS1273 | CE2 `ge-0/0/1.0` | `198.51.100.2/31` |
| CE2—AS1273 | AS1273 `ge-0/0/1.0` | `198.51.100.3/31` |
| AS1273—old vSRX | AS1273 `ge-0/0/2.0` | `192.0.2.0/31` |
| AS1273—old vSRX | OLD-VSRX-GW `ge-0/0/2.0` | `192.0.2.1/31` |
| AS1273—new SRX | AS1273 `ge-0/0/3.0` | `192.0.2.2/31` |
| AS1273—new SRX | NEW-SRX-GW `ge-0/0/3.0` | `192.0.2.3/31` |
| Old vSRX—IPBB | OLD-VSRX-GW `ge-0/0/5.0` | `10.44.0.0/31` |
| Old vSRX—IPBB | IPBB-CORE `ge-0/0/5.0` | `10.44.0.1/31` |
| New SRX—IPBB | NEW-SRX-GW `ge-0/0/4.0` | `10.44.0.2/31` |
| New SRX—IPBB | IPBB-CORE `ge-0/0/4.0` | `10.44.0.3/31` |
| IPBB—Server | IPBB-CORE `ge-0/0/1.0` | `10.100.0.1/24` |
| IPBB—Server | Server `eth1` | `10.100.0.10/24` |
| Shared IKE/IPsec endpoint | OLD-VSRX-GW `lo0.0` | `203.0.113.100/32` |
| Shared IKE/IPsec endpoint | NEW-SRX-GW `lo0.0` | `203.0.113.100/32` |

### 4.2 Protected networks

| Site | Prefix/test address |
|---|---|
| CE1 protected network | `10.1.1.0/24`; CE test address `10.1.1.1` |
| CE2 protected network | `10.2.2.0/24`; CE test address `10.2.2.1` |
| IPBB protected network | `10.100.0.0/24`; server `10.100.0.10` |

### 4.3 Cryptographic profile

| Parameter | Value |
|---|---|
| IKE | IKEv1 main mode |
| Authentication | Pre-shared key |
| IKE encryption/integrity | AES-256-CBC / SHA-256 |
| Diffie-Hellman | Group 14 |
| IKE lifetime | 28,800 seconds |
| IPsec protocol | Native ESP tunnel mode |
| ESP encryption/integrity | AES-256-CBC / HMAC-SHA-256-128 |
| PFS | Group 14 |
| IPsec lifetime | 3,600 seconds |
| NAT-T | Disabled on every gateway and CE |
| Lab PSK | `Phase1-POC-PSK-2026`; replace outside this isolated PoC |

## 5. CE1 configuration — migrated tunnel

Paste from configuration mode.

```text
set system host-name CE1

set interfaces ge-0/0/0 description TO-AS1273-EDGE
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.0/31
set interfaces lo0 unit 0 description CE1-PROTECTED-TEST
set interfaces lo0 unit 0 family inet address 10.1.1.1/24
set interfaces st0 unit 0 family inet

set routing-options static route 0.0.0.0/0 next-hop 198.51.100.1
set routing-options static route 10.100.0.0/24 next-hop st0.0

set security ike proposal POC-IKE-PROPOSAL authentication-method pre-shared-keys
set security ike proposal POC-IKE-PROPOSAL dh-group group14
set security ike proposal POC-IKE-PROPOSAL authentication-algorithm sha-256
set security ike proposal POC-IKE-PROPOSAL encryption-algorithm aes-256-cbc
set security ike proposal POC-IKE-PROPOSAL lifetime-seconds 28800

set security ike policy POC-IKE-POLICY mode main
set security ike policy POC-IKE-POLICY proposals POC-IKE-PROPOSAL
set security ike policy POC-IKE-POLICY pre-shared-key ascii-text "Phase1-POC-PSK-2026"

set security ike gateway SHARED-IPSEC-GW ike-policy POC-IKE-POLICY
set security ike gateway SHARED-IPSEC-GW address 203.0.113.100
set security ike gateway SHARED-IPSEC-GW external-interface ge-0/0/0.0
set security ike gateway SHARED-IPSEC-GW local-address 198.51.100.0
set security ike gateway SHARED-IPSEC-GW local-identity inet 198.51.100.0
set security ike gateway SHARED-IPSEC-GW version v1-only
set security ike gateway SHARED-IPSEC-GW no-nat-traversal
set security ike gateway SHARED-IPSEC-GW dead-peer-detection interval 10
set security ike gateway SHARED-IPSEC-GW dead-peer-detection threshold 3

set security ipsec proposal POC-IPSEC-PROPOSAL protocol esp
set security ipsec proposal POC-IPSEC-PROPOSAL authentication-algorithm hmac-sha-256-128
set security ipsec proposal POC-IPSEC-PROPOSAL encryption-algorithm aes-256-cbc
set security ipsec proposal POC-IPSEC-PROPOSAL lifetime-seconds 3600

set security ipsec policy POC-IPSEC-POLICY perfect-forward-secrecy keys group14
set security ipsec policy POC-IPSEC-POLICY proposals POC-IPSEC-PROPOSAL

set security ipsec vpn CE1-VPN bind-interface st0.0
set security ipsec vpn CE1-VPN ike gateway SHARED-IPSEC-GW
set security ipsec vpn CE1-VPN ike ipsec-policy POC-IPSEC-POLICY
set security ipsec vpn CE1-VPN traffic-selector CE1-TS local-ip 10.1.1.0/24
set security ipsec vpn CE1-VPN traffic-selector CE1-TS remote-ip 10.100.0.0/24
set security ipsec vpn CE1-VPN establish-tunnels immediately

set security zones security-zone INTERNET interfaces ge-0/0/0.0 host-inbound-traffic system-services ike
set security zones security-zone INTERNET interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone LAN interfaces lo0.0 host-inbound-traffic system-services ping
set security zones security-zone VPN interfaces st0.0

set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN match source-address any
set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN match destination-address any
set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN match application any
set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN then permit

set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN match source-address any
set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN match destination-address any
set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN match application any
set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN then permit

set security flow tcp-mss ipsec-vpn mss 1350
```

## 6. CE2 configuration — control tunnel

```text
set system host-name CE2

set interfaces ge-0/0/1 description TO-AS1273-EDGE
set interfaces ge-0/0/1 unit 0 family inet address 198.51.100.2/31
set interfaces lo0 unit 0 description CE2-PROTECTED-TEST
set interfaces lo0 unit 0 family inet address 10.2.2.1/24
set interfaces st0 unit 0 family inet

set routing-options static route 0.0.0.0/0 next-hop 198.51.100.3
set routing-options static route 10.100.0.0/24 next-hop st0.0

set security ike proposal POC-IKE-PROPOSAL authentication-method pre-shared-keys
set security ike proposal POC-IKE-PROPOSAL dh-group group14
set security ike proposal POC-IKE-PROPOSAL authentication-algorithm sha-256
set security ike proposal POC-IKE-PROPOSAL encryption-algorithm aes-256-cbc
set security ike proposal POC-IKE-PROPOSAL lifetime-seconds 28800

set security ike policy POC-IKE-POLICY mode main
set security ike policy POC-IKE-POLICY proposals POC-IKE-PROPOSAL
set security ike policy POC-IKE-POLICY pre-shared-key ascii-text "Phase1-POC-PSK-2026"

set security ike gateway SHARED-IPSEC-GW ike-policy POC-IKE-POLICY
set security ike gateway SHARED-IPSEC-GW address 203.0.113.100
set security ike gateway SHARED-IPSEC-GW external-interface ge-0/0/1.0
set security ike gateway SHARED-IPSEC-GW local-address 198.51.100.2
set security ike gateway SHARED-IPSEC-GW local-identity inet 198.51.100.2
set security ike gateway SHARED-IPSEC-GW version v1-only
set security ike gateway SHARED-IPSEC-GW no-nat-traversal
set security ike gateway SHARED-IPSEC-GW dead-peer-detection interval 10
set security ike gateway SHARED-IPSEC-GW dead-peer-detection threshold 3

set security ipsec proposal POC-IPSEC-PROPOSAL protocol esp
set security ipsec proposal POC-IPSEC-PROPOSAL authentication-algorithm hmac-sha-256-128
set security ipsec proposal POC-IPSEC-PROPOSAL encryption-algorithm aes-256-cbc
set security ipsec proposal POC-IPSEC-PROPOSAL lifetime-seconds 3600

set security ipsec policy POC-IPSEC-POLICY perfect-forward-secrecy keys group14
set security ipsec policy POC-IPSEC-POLICY proposals POC-IPSEC-PROPOSAL

set security ipsec vpn CE2-VPN bind-interface st0.0
set security ipsec vpn CE2-VPN ike gateway SHARED-IPSEC-GW
set security ipsec vpn CE2-VPN ike ipsec-policy POC-IPSEC-POLICY
set security ipsec vpn CE2-VPN traffic-selector CE2-TS local-ip 10.2.2.0/24
set security ipsec vpn CE2-VPN traffic-selector CE2-TS remote-ip 10.100.0.0/24
set security ipsec vpn CE2-VPN establish-tunnels immediately

set security zones security-zone INTERNET interfaces ge-0/0/1.0 host-inbound-traffic system-services ike
set security zones security-zone INTERNET interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone LAN interfaces lo0.0 host-inbound-traffic system-services ping
set security zones security-zone VPN interfaces st0.0

set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN match source-address any
set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN match destination-address any
set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN match application any
set security policies from-zone LAN to-zone VPN policy LAN-TO-VPN then permit

set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN match source-address any
set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN match destination-address any
set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN match application any
set security policies from-zone VPN to-zone LAN policy VPN-TO-LAN then permit

set security flow tcp-mss ipsec-vpn mss 1350
```

## 7. AS1273-EDGE configuration

AS1273 initially routes the shared endpoint to the old vSRX. During staging, one static route change represents the production MED preference toward the new gateway.

```text
set system host-name AS1273-EDGE

set interfaces ge-0/0/0 description TO-CE1
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/31
set interfaces ge-0/0/1 description TO-CE2
set interfaces ge-0/0/1 unit 0 family inet address 198.51.100.3/31
set interfaces ge-0/0/2 description TO-OLD-VSRX-GW
set interfaces ge-0/0/2 unit 0 family inet address 192.0.2.0/31
set interfaces ge-0/0/3 description TO-NEW-SRX-GW
set interfaces ge-0/0/3 unit 0 family inet address 192.0.2.2/31

set routing-options static route 203.0.113.100/32 next-hop 192.0.2.1
```

Baseline verification:

```text
show route 203.0.113.100/32 exact
ping 192.0.2.1
ping 192.0.2.3
```

## 8. OLD-VSRX-GW configuration

The old vSRX initially terminates CE1 and CE2. It keeps both customer VPN configurations during migration so that CE1 can be rolled back without rebuilding configuration.

```text
set system host-name OLD-VSRX-GW

set interfaces ge-0/0/2 description TO-AS1273-EDGE
set interfaces ge-0/0/2 unit 0 family inet address 192.0.2.1/31
set interfaces ge-0/0/5 description TO-IPBB-CORE
set interfaces ge-0/0/5 unit 0 family inet address 10.44.0.0/31
set interfaces lo0 unit 0 description SHARED-IPSEC-ENDPOINT
set interfaces lo0 unit 0 family inet address 203.0.113.100/32
set interfaces st0 unit 1 family inet
set interfaces st0 unit 2 family inet

set routing-options static route 0.0.0.0/0 next-hop 192.0.2.0
set routing-options static route 10.100.0.0/24 next-hop 10.44.0.1
set routing-options static route 10.1.1.0/24 next-hop st0.1
set routing-options static route 10.2.2.0/24 next-hop st0.2

set security ike proposal POC-IKE-PROPOSAL authentication-method pre-shared-keys
set security ike proposal POC-IKE-PROPOSAL dh-group group14
set security ike proposal POC-IKE-PROPOSAL authentication-algorithm sha-256
set security ike proposal POC-IKE-PROPOSAL encryption-algorithm aes-256-cbc
set security ike proposal POC-IKE-PROPOSAL lifetime-seconds 28800

set security ike policy POC-IKE-POLICY mode main
set security ike policy POC-IKE-POLICY proposals POC-IKE-PROPOSAL
set security ike policy POC-IKE-POLICY pre-shared-key ascii-text "Phase1-POC-PSK-2026"

set security ike gateway CE1-GATEWAY ike-policy POC-IKE-POLICY
set security ike gateway CE1-GATEWAY address 198.51.100.0
set security ike gateway CE1-GATEWAY external-interface lo0.0
set security ike gateway CE1-GATEWAY local-address 203.0.113.100
set security ike gateway CE1-GATEWAY local-identity inet 203.0.113.100
set security ike gateway CE1-GATEWAY version v1-only
set security ike gateway CE1-GATEWAY no-nat-traversal
set security ike gateway CE1-GATEWAY dead-peer-detection interval 10
set security ike gateway CE1-GATEWAY dead-peer-detection threshold 3

set security ike gateway CE2-GATEWAY ike-policy POC-IKE-POLICY
set security ike gateway CE2-GATEWAY address 198.51.100.2
set security ike gateway CE2-GATEWAY external-interface lo0.0
set security ike gateway CE2-GATEWAY local-address 203.0.113.100
set security ike gateway CE2-GATEWAY local-identity inet 203.0.113.100
set security ike gateway CE2-GATEWAY version v1-only
set security ike gateway CE2-GATEWAY no-nat-traversal
set security ike gateway CE2-GATEWAY dead-peer-detection interval 10
set security ike gateway CE2-GATEWAY dead-peer-detection threshold 3

set security ipsec proposal POC-IPSEC-PROPOSAL protocol esp
set security ipsec proposal POC-IPSEC-PROPOSAL authentication-algorithm hmac-sha-256-128
set security ipsec proposal POC-IPSEC-PROPOSAL encryption-algorithm aes-256-cbc
set security ipsec proposal POC-IPSEC-PROPOSAL lifetime-seconds 3600

set security ipsec policy POC-IPSEC-POLICY perfect-forward-secrecy keys group14
set security ipsec policy POC-IPSEC-POLICY proposals POC-IPSEC-PROPOSAL

set security ipsec vpn CE1-VPN bind-interface st0.1
set security ipsec vpn CE1-VPN ike gateway CE1-GATEWAY
set security ipsec vpn CE1-VPN ike ipsec-policy POC-IPSEC-POLICY
set security ipsec vpn CE1-VPN traffic-selector CE1-TS local-ip 10.100.0.0/24
set security ipsec vpn CE1-VPN traffic-selector CE1-TS remote-ip 10.1.1.0/24

set security ipsec vpn CE2-VPN bind-interface st0.2
set security ipsec vpn CE2-VPN ike gateway CE2-GATEWAY
set security ipsec vpn CE2-VPN ike ipsec-policy POC-IPSEC-POLICY
set security ipsec vpn CE2-VPN traffic-selector CE2-TS local-ip 10.100.0.0/24
set security ipsec vpn CE2-VPN traffic-selector CE2-TS remote-ip 10.2.2.0/24

set security zones security-zone TRANSPORT interfaces ge-0/0/2.0 host-inbound-traffic system-services ike
set security zones security-zone TRANSPORT interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security zones security-zone TRANSPORT interfaces ge-0/0/5.0 host-inbound-traffic system-services ike
set security zones security-zone TRANSPORT interfaces ge-0/0/5.0 host-inbound-traffic system-services ping
set security zones security-zone TRANSPORT interfaces lo0.0 host-inbound-traffic system-services ike
set security zones security-zone TRANSPORT interfaces lo0.0 host-inbound-traffic system-services ping
set security zones security-zone VPN interfaces st0.1
set security zones security-zone VPN interfaces st0.2

set security policies from-zone TRANSPORT to-zone TRANSPORT policy TRANSPORT-INTRAZONE match source-address any
set security policies from-zone TRANSPORT to-zone TRANSPORT policy TRANSPORT-INTRAZONE match destination-address any
set security policies from-zone TRANSPORT to-zone TRANSPORT policy TRANSPORT-INTRAZONE match application any
set security policies from-zone TRANSPORT to-zone TRANSPORT policy TRANSPORT-INTRAZONE then permit

set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN match source-address any
set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN match destination-address any
set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN match application any
set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN then permit

set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT match source-address any
set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT match destination-address any
set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT match application any
set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT then permit

set security flow tcp-mss ipsec-vpn mss 1350
```

Do not configure `establish-tunnels immediately` on the old gateway. CE1 and CE2 are the initiators, preventing the old gateway from repeatedly initiating CE1 after migration.

`TRANSPORT-INTRAZONE` is mandatory in this lab because outer IKE/ESP traffic can enter on one transport interface and leave through another transport interface during tromboning. The broad `any` match is acceptable only for this isolated POC; production policy must be restricted to the shared endpoint, approved peer prefixes and the required IKE/ESP applications.

## 9. NEW-SRX-GW configuration

The new SRX owns the same loopback endpoint and is preconfigured for CE1. During baseline and staging, the CE1 IKE gateway and VPN remain inactive; its input filter diverts peer addresses present in `OLD-PEERS` to the old gateway.

### 9.1 Interfaces, routing and CE1 IPsec

```text
set system host-name NEW-SRX-GW

set interfaces ge-0/0/3 description TO-AS1273-EDGE
set interfaces ge-0/0/3 unit 0 family inet address 192.0.2.3/31
set interfaces ge-0/0/4 description TO-IPBB-CORE
set interfaces ge-0/0/4 unit 0 family inet address 10.44.0.2/31
set interfaces lo0 unit 0 description SHARED-IPSEC-ENDPOINT
set interfaces lo0 unit 0 family inet address 203.0.113.100/32
set interfaces st0 unit 1 family inet

set routing-options static route 0.0.0.0/0 next-hop 192.0.2.2
set routing-options static route 10.100.0.0/24 next-hop 10.44.0.3
set routing-options static route 10.1.1.0/24 next-hop st0.1

set security ike proposal POC-IKE-PROPOSAL authentication-method pre-shared-keys
set security ike proposal POC-IKE-PROPOSAL dh-group group14
set security ike proposal POC-IKE-PROPOSAL authentication-algorithm sha-256
set security ike proposal POC-IKE-PROPOSAL encryption-algorithm aes-256-cbc
set security ike proposal POC-IKE-PROPOSAL lifetime-seconds 28800

set security ike policy POC-IKE-POLICY mode main
set security ike policy POC-IKE-POLICY proposals POC-IKE-PROPOSAL
set security ike policy POC-IKE-POLICY pre-shared-key ascii-text "Phase1-POC-PSK-2026"

set security ike gateway CE1-GATEWAY ike-policy POC-IKE-POLICY
set security ike gateway CE1-GATEWAY address 198.51.100.0
set security ike gateway CE1-GATEWAY external-interface lo0.0
set security ike gateway CE1-GATEWAY local-address 203.0.113.100
set security ike gateway CE1-GATEWAY local-identity inet 203.0.113.100
set security ike gateway CE1-GATEWAY version v1-only
set security ike gateway CE1-GATEWAY no-nat-traversal
set security ike gateway CE1-GATEWAY dead-peer-detection interval 10
set security ike gateway CE1-GATEWAY dead-peer-detection threshold 3

set security ipsec proposal POC-IPSEC-PROPOSAL protocol esp
set security ipsec proposal POC-IPSEC-PROPOSAL authentication-algorithm hmac-sha-256-128
set security ipsec proposal POC-IPSEC-PROPOSAL encryption-algorithm aes-256-cbc
set security ipsec proposal POC-IPSEC-PROPOSAL lifetime-seconds 3600

set security ipsec policy POC-IPSEC-POLICY perfect-forward-secrecy keys group14
set security ipsec policy POC-IPSEC-POLICY proposals POC-IPSEC-PROPOSAL

set security ipsec vpn CE1-VPN bind-interface st0.1
set security ipsec vpn CE1-VPN ike gateway CE1-GATEWAY
set security ipsec vpn CE1-VPN ike ipsec-policy POC-IPSEC-POLICY
set security ipsec vpn CE1-VPN traffic-selector CE1-TS local-ip 10.100.0.0/24
set security ipsec vpn CE1-VPN traffic-selector CE1-TS remote-ip 10.1.1.0/24
set security ipsec vpn CE1-VPN establish-tunnels immediately

deactivate security ipsec vpn CE1-VPN
deactivate security ike gateway CE1-GATEWAY

set security zones security-zone TRANSPORT interfaces ge-0/0/3.0 host-inbound-traffic system-services ike
set security zones security-zone TRANSPORT interfaces ge-0/0/3.0 host-inbound-traffic system-services ping
set security zones security-zone TRANSPORT interfaces ge-0/0/4.0 host-inbound-traffic system-services ike
set security zones security-zone TRANSPORT interfaces ge-0/0/4.0 host-inbound-traffic system-services ping
set security zones security-zone TRANSPORT interfaces lo0.0 host-inbound-traffic system-services ike
set security zones security-zone TRANSPORT interfaces lo0.0 host-inbound-traffic system-services ping
set security zones security-zone VPN interfaces st0.1

set security policies from-zone TRANSPORT to-zone TRANSPORT policy TROMBONE-OUTER-IPSEC match source-address any
set security policies from-zone TRANSPORT to-zone TRANSPORT policy TROMBONE-OUTER-IPSEC match destination-address any
set security policies from-zone TRANSPORT to-zone TRANSPORT policy TROMBONE-OUTER-IPSEC match application any
set security policies from-zone TRANSPORT to-zone TRANSPORT policy TROMBONE-OUTER-IPSEC then permit

set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN match source-address any
set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN match destination-address any
set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN match application any
set security policies from-zone TRANSPORT to-zone VPN policy TRANSPORT-TO-VPN then permit

set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT match source-address any
set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT match destination-address any
set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT match application any
set security policies from-zone VPN to-zone TRANSPORT policy VPN-TO-TRANSPORT then permit

set security flow tcp-mss ipsec-vpn mss 1350
```

The two `deactivate` commands are part of the baseline state, not deletions. They preserve the complete CE1 configuration for an atomic activation during cutover. `establish-tunnels immediately` is configured in advance so the new SRX can drive Phase 2 as soon as CE1 IKE moves.

### 9.2 Old-peer forwarding instance

Only the directly connected new-SRX-to-IPBB subnet is imported into the forwarding table. This prevents the local `203.0.113.100/32` loopback route from overriding the static trombone route.

`inet.0` remains the primary table in the RIB group. The import policy limits the interface route copied into the secondary `OLD-TROMBONE.inet.0` table; the pre-flight gate below also verifies that normal `inet.0` routes remain present.

```text
set policy-options prefix-list OLD-PEERS 198.51.100.0/32
set policy-options prefix-list OLD-PEERS 198.51.100.2/32

set policy-options policy-statement TROMBONE-IMPORT term IPBB-CONNECTED from protocol direct
set policy-options policy-statement TROMBONE-IMPORT term IPBB-CONNECTED from route-filter 10.44.0.2/31 exact
set policy-options policy-statement TROMBONE-IMPORT term IPBB-CONNECTED then accept
set policy-options policy-statement TROMBONE-IMPORT term REJECT then reject

set routing-instances OLD-TROMBONE instance-type forwarding
set routing-instances OLD-TROMBONE routing-options static route 203.0.113.100/32 next-hop 10.44.0.3

set routing-options rib-groups TROMBONE-RIB import-rib inet.0
set routing-options rib-groups TROMBONE-RIB import-rib OLD-TROMBONE.inet.0
set routing-options rib-groups TROMBONE-RIB import-policy TROMBONE-IMPORT
set routing-options interface-routes rib-group inet TROMBONE-RIB
```

### 9.3 Old-peer input filter

NAT-T is excluded, so the filter matches only IKE UDP/500, native ESP and diagnostic ICMP. Unmatched traffic is accepted and uses the normal routing table, allowing a removed peer to reach the local new-SRX IPsec service.

Apply this filter **only to `NEW-SRX-GW ge-0/0/3.0` in the input direction**, which is the interface receiving peer traffic from `AS1273-EDGE`. Do not apply it to the old vSRX, AS1273, IPBB, `lo0.0`, `ge-0/0/4.0`, or any output direction.

```text
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-IKE from source-prefix-list OLD-PEERS
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-IKE from destination-address 203.0.113.100/32
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-IKE from protocol udp
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-IKE from destination-port 500
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-IKE then count OLD-IKE-TO-OLD
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-IKE then next term

set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-IKE from source-prefix-list OLD-PEERS
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-IKE from destination-address 203.0.113.100/32
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-IKE from protocol udp
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-IKE from destination-port 500
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-IKE then routing-instance OLD-TROMBONE

set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ESP from source-prefix-list OLD-PEERS
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ESP from destination-address 203.0.113.100/32
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ESP from protocol esp
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ESP then count OLD-ESP-TO-OLD
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ESP then next term

set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ESP from source-prefix-list OLD-PEERS
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ESP from destination-address 203.0.113.100/32
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ESP from protocol esp
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ESP then routing-instance OLD-TROMBONE

set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ICMP from source-prefix-list OLD-PEERS
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ICMP from destination-address 203.0.113.100/32
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ICMP from protocol icmp
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ICMP then count OLD-ICMP-TO-OLD
set firewall family inet filter IPSEC-TROMBONE term COUNT-OLD-ICMP then next term

set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ICMP from source-prefix-list OLD-PEERS
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ICMP from destination-address 203.0.113.100/32
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ICMP from protocol icmp
set firewall family inet filter IPSEC-TROMBONE term FORWARD-OLD-ICMP then routing-instance OLD-TROMBONE

set firewall family inet filter IPSEC-TROMBONE term DEFAULT then accept

set interfaces ge-0/0/3 unit 0 family inet filter input IPSEC-TROMBONE
```

The counter and forwarding actions are deliberately in separate terms. The counter term uses `next term`; the following term performs the terminating `routing-instance` action. Do not use `then next-ip` on the SRX.

## 10. IPBB-CORE configuration

The core route for the shared endpoint provides the new-to-old trombone. Protected-prefix routes initially point to the old vSRX.

```text
set system host-name IPBB-CORE

set interfaces ge-0/0/5 description TO-OLD-VSRX-GW
set interfaces ge-0/0/5 unit 0 family inet address 10.44.0.1/31
set interfaces ge-0/0/4 description TO-NEW-SRX-GW
set interfaces ge-0/0/4 unit 0 family inet address 10.44.0.3/31
set interfaces ge-0/0/1 description TO-SERVER
set interfaces ge-0/0/1 unit 0 family inet address 10.100.0.1/24

set routing-options static route 203.0.113.100/32 next-hop 10.44.0.0
set routing-options static route 10.1.1.0/24 next-hop 10.44.0.0
set routing-options static route 10.2.2.0/24 next-hop 10.44.0.0
```

Verification:

```text
show route 203.0.113.100/32 exact
show route 10.1.1.0/24 exact
show route 10.2.2.0/24 exact
ping 10.44.0.0
ping 10.44.0.2
ping 10.100.0.10
```

## 11. Linux Server configuration

The lab output identifies the test-facing interface as `eth1`. Keep the management default route through `eth0`; add only the two protected-prefix routes through IPBB.

```bash
sudo ip link set eth1 up
sudo ip address replace 10.100.0.10/24 dev eth1
sudo ip route replace 10.1.1.0/24 via 10.100.0.1 dev eth1
sudo ip route replace 10.2.2.0/24 via 10.100.0.1 dev eth1

ip -br address show eth1
ip route
ip route get 10.1.1.1
ip route get 10.2.2.1
ping -I eth1 -c 5 10.100.0.1
```

Expected route lookups:

```text
10.1.1.1 via 10.100.0.1 dev eth1 src 10.100.0.10
10.2.2.1 via 10.100.0.1 dev eth1 src 10.100.0.10
```

These commands are runtime-only. Add equivalent Netplan configuration only if persistence across server reboots is needed.

## 12. Build and pre-flight gates

### 12.1 Recommended commit order

1. `IPBB-CORE`.
2. Linux Server.
3. `AS1273-EDGE` with the endpoint still routed to the old vSRX.
4. `OLD-VSRX-GW`.
5. `NEW-SRX-GW`, including its inactive-in-practice FBF path.
6. `CE1`.
7. `CE2`.

Use `commit check`, followed by `commit confirmed 10` for routing, steering or gateway-state changes. The lab showed that five minutes was insufficient for indexed clears, Phase 2 verification and protected-route testing.

For each Junos configuration block, use this safe sequence from configuration mode:

```text
show | compare
commit check
commit confirmed 10
run show system commit
```

Run the relevant gates below before issuing a plain `commit` to confirm the change. If verification fails, use `rollback 1` and `commit`, or allow the confirmed commit to time out.

### 12.2 Configuration-hygiene and node-identity gate

This gate prevents configuration copied from another node from changing the test outcome. Run it on every SRX before tunnel establishment:

```text
show configuration | display set | except groups
show configuration security ike | display set
show configuration security ipsec vpn | display set
show configuration security zones | display set
show route 203.0.113.100
```

Required node identities:

| Node | Must contain | Must not contain |
|---|---|---|
| `CE1` | `ge-0/0/0.0 198.51.100.0/31`, `lo0.0 10.1.1.1/24`, `st0.0`, `SHARED-IPSEC-GW`, `CE1-VPN`, default via `198.51.100.1`, `10.100.0.0/24` via `st0.0` | Local `203.0.113.100/32`, `CE2-VPN`, `CE1-GATEWAY`, `CE2-GATEWAY`, `st0.1`, `st0.2`, `ge-0/0/2`, `ge-0/0/5`, `TRANSPORT` zone |
| `CE2` | `ge-0/0/1.0 198.51.100.2/31`, `lo0.0 10.2.2.1/24`, `st0.0`, `SHARED-IPSEC-GW`, `CE2-VPN`, default via `198.51.100.3`, `10.100.0.0/24` via `st0.0` | Local `203.0.113.100/32`, `CE1-VPN`, old/new gateway objects, `st0.1`, `st0.2`, old/new gateway transport links |
| `OLD-VSRX-GW` | Both CE gateways/VPNs, `lo0.0 203.0.113.100/32`, `st0.1`, `st0.2`, `TRANSPORT-INTRAZONE` | `OLD-PEERS`, `IPSEC-TROMBONE`, `OLD-TROMBONE` |
| `NEW-SRX-GW` | Inactive CE1 gateway/VPN with `establish-tunnels immediately`, shared endpoint, `OLD-PEERS`, `IPSEC-TROMBONE`, `OLD-TROMBONE` | CE2 VPN or CE2 termination configuration in Phase 1 |

On CE1 and CE2, `203.0.113.100` must resolve through the default route. A result of `Direct/0 via lo0.0` is a hard stop: remove the accidental shared-endpoint address from that CE before proceeding.

Re-enter the same POC pre-shared key on CE1, CE2, `OLD-VSRX-GW` and `NEW-SRX-GW` during the controlled build. A displayed `$9$...` value cannot be compared between devices because Junos stores an encrypted representation; successful IKE authentication is the operational proof.

### 12.3 SRX configuration and routing gate

Run these checks immediately after `commit check` and before allowing either CE to establish.

On `OLD-VSRX-GW`:

```text
show interfaces terse | match "ge-0/0/2|ge-0/0/5|lo0.0|st0.1|st0.2"
show security zones detail
show configuration security ike | display set
show configuration security ipsec | display set
show route 198.51.100.0
show route 198.51.100.2
show route 10.100.0.10
show route 10.1.1.1
show route 10.2.2.1
show security ipsec inactive-tunnels detail
```

Required results:

- `ge-0/0/2.0`, `ge-0/0/5.0` and `lo0.0` are in `TRANSPORT`.
- `st0.1` and `st0.2` are in `VPN`.
- `TRANSPORT-INTRAZONE` is present and permits the outer transport path used by the POC.
- Peer routes resolve through the default route to `192.0.2.0`.
- `10.100.0.0/24` resolves through `10.44.0.1`.
- `10.1.1.0/24` resolves through `st0.1`; `10.2.2.0/24` resolves through `st0.2`.
- CE1 and CE2 may appear as inactive tunnels waiting for a trigger; they must not show a configuration or proposal error.

On `NEW-SRX-GW`:

```text
show interfaces terse | match "ge-0/0/3|ge-0/0/4|lo0.0|st0.1"
show security zones detail
show configuration security ike | display set
show configuration security ipsec | display set
show route 198.51.100.0
show route 10.100.0.10
show route 10.1.1.1
show security ipsec inactive-tunnels detail
```

Required results:

- `ge-0/0/3.0`, `ge-0/0/4.0` and `lo0.0` are in `TRANSPORT`.
- `st0.1` is in `VPN`.
- CE1 peer `198.51.100.0` resolves through the default route to `192.0.2.2`.
- `10.100.0.0/24` resolves through `10.44.0.3`.
- `10.1.1.0/24` resolves through `st0.1`.
- `CE1-GATEWAY` and `CE1-VPN` are inactive during staging, so the new SRX has no CE1 IKE/IPsec SA and no CE1 local self-tunnel session.

### 12.4 Underlay gate

```text
CE1> ping 198.51.100.1
CE2> ping 198.51.100.3

OLD-VSRX-GW> ping 192.0.2.0
OLD-VSRX-GW> ping 10.44.0.1

NEW-SRX-GW> ping 192.0.2.2
NEW-SRX-GW> ping 10.44.0.3

IPBB-CORE> ping 10.44.0.0
IPBB-CORE> ping 10.44.0.2
IPBB-CORE> ping 10.100.0.10
```

### 12.5 New-SRX FBF capability gate

This gate must pass before changing AS1273 routing.

```text
NEW-SRX-GW> show interfaces filters
NEW-SRX-GW> show configuration firewall family inet filter IPSEC-TROMBONE | display set
NEW-SRX-GW> show configuration routing-instances OLD-TROMBONE | display set
NEW-SRX-GW> show configuration routing-options rib-groups TROMBONE-RIB | display set
NEW-SRX-GW> show route 192.0.2.2/31 exact
NEW-SRX-GW> show route 10.44.0.2/31 exact
NEW-SRX-GW> show route 203.0.113.100/32 exact
NEW-SRX-GW> show route table OLD-TROMBONE.inet.0 10.44.0.2/31 exact
NEW-SRX-GW> show route table OLD-TROMBONE.inet.0 10.44.0.3
NEW-SRX-GW> show route table OLD-TROMBONE.inet.0 203.0.113.100/32 exact
NEW-SRX-GW> show route table OLD-TROMBONE.inet.0 hidden
NEW-SRX-GW> show firewall filter IPSEC-TROMBONE
```

Required results:

- `IPSEC-TROMBONE` is installed on `ge-0/0/3.0` input.
- Normal new-SRX routes remain active in `inet.0`, including the AS1273 link, IPBB link and local shared endpoint.
- `10.44.0.2/31` is present in `OLD-TROMBONE.inet.0` as the connected resolution route.
- `203.0.113.100/32` is a static route through `10.44.0.3`.
- `203.0.113.100/32` must not appear as a direct or local route in `OLD-TROMBONE.inet.0`.
- The count terms are followed by their corresponding forwarding terms in the required order.
- `commit check` and the vSRX forwarding plane show no FBF errors.

If the lab vSRX image cannot pass this gate, stop. Do not continue with a partially installed forwarding instance or clear any existing tunnel.

### 12.6 Operator terminals and evidence record

Keep these four views open throughout staging, migration and rollback:

1. Linux server: continuous ping to CE1 `10.1.1.1`.
2. Linux server: continuous ping to CE2 `10.2.2.1`; CE2 is the control and any sustained loss is an immediate rollback trigger.
3. `NEW-SRX-GW`: FBF counters plus IKE/IPsec SA state.
4. `OLD-VSRX-GW`: CE1 and CE2 IKE/IPsec SA state.

Before every state change, record the timestamp, active IKE index, IPsec VPN/ID, route next hop and relevant packet counters. Never reuse a recorded SA index without checking the peer address and gateway name immediately before the clear operation.

## 13. Baseline — both tunnels on OLD-VSRX-GW

AS1273 must still show the shared endpoint through the old gateway:

```text
AS1273-EDGE> show route 203.0.113.100/32 exact
```

Generate protected traffic from the CEs:

```text
CE1> ping 10.100.0.10 source 10.1.1.1 rapid count 100
CE2> ping 10.100.0.10 source 10.2.2.1 rapid count 100
```

Generate protected traffic from the Linux server:

```bash
ping -I 10.100.0.10 -c 100 10.1.1.1
ping -I 10.100.0.10 -c 100 10.2.2.1
```

Verify each CE. Use the same commands on CE2, substituting `CE2-VPN` and the CE2 addresses where applicable:

```text
CE1> show security ike security-associations detail
CE1> show security ike active-peer detail
CE1> show security ipsec security-associations detail
CE1> show security ipsec statistics
CE1> show route 10.100.0.10 exact
CE1> show interfaces st0.0 extensive
```

Verify the old gateway:

```text
OLD-VSRX-GW> show security ike security-associations detail
OLD-VSRX-GW> show security ike active-peer detail
OLD-VSRX-GW> show security ipsec security-associations detail
OLD-VSRX-GW> show security ipsec statistics
OLD-VSRX-GW> show interfaces terse | match "st0.1|st0.2"
OLD-VSRX-GW> show route 10.100.0.10
OLD-VSRX-GW> show security policies hit-count
OLD-VSRX-GW> show security flow session
```

Verify the new gateway has no SAs:

```text
NEW-SRX-GW> show security ike security-associations
NEW-SRX-GW> show security ipsec security-associations
NEW-SRX-GW> show security ipsec inactive-tunnels detail
```

Baseline passes only when:

- CE1 and CE2 have installed IKE and IPsec SAs on `OLD-VSRX-GW`.
- Both protected flows pass bidirectionally.
- Encrypt and decrypt counters increase on `OLD-VSRX-GW`.
- `NEW-SRX-GW` has no CE1 or CE2 SA.
- Outer traffic uses UDP/500 and ESP protocol 50; UDP/4500 is absent.

## 14. MoP staging — new gateway attracts, old gateway terminates

This stage proves the temporary forwarding design before moving CE1. No customer is migrated in this section.

### Step 1 — Verify the required new-SRX staging state

On `NEW-SRX-GW`:

```text
show configuration security ike gateway CE1-GATEWAY | display set
show configuration security ipsec vpn CE1-VPN | display set
show configuration policy-options prefix-list OLD-PEERS | display set
```

Required state:

- `CE1-GATEWAY` is inactive.
- `CE1-VPN` is inactive and contains `establish-tunnels immediately`.
- `198.51.100.0/32` and `198.51.100.2/32` are active in `OLD-PEERS`.

Do not begin endpoint attraction if the CE1 IKE gateway is active. The lab proved that an active gateway can claim UDP/500 locally and create a `lo0.0`/`.local..0` self-tunnel session even while CE1 is in `OLD-PEERS`.

### Step 2 — Start continuous protected traffic

Use two Linux terminals:

```bash
ping -I 10.100.0.10 -i 0.2 10.1.1.1
ping -I 10.100.0.10 -i 0.2 10.2.2.1
```

Optionally run CE-to-server traffic in parallel:

```text
CE1> ping 10.100.0.10 source 10.1.1.1 rapid count 1000
CE2> ping 10.100.0.10 source 10.2.2.1 rapid count 1000
```

### Step 3 — Move the shared endpoint route to the new gateway

On `AS1273-EDGE`:

```text
configure
delete routing-options static route 203.0.113.100/32
set routing-options static route 203.0.113.100/32 next-hop 192.0.2.3
commit confirmed 10 comment "Attract shared IPsec endpoint to new SRX"
```

Verify:

```text
run show route 203.0.113.100/32 exact
```

The active next hop must be `192.0.2.3` via `ge-0/0/3.0`. Do not configure `10.1.1.0/24` on AS1273; that protected route belongs only on `IPBB-CORE`.

### Step 4 — Prove both peers are forwarded to the old vSRX

On `NEW-SRX-GW`:

```text
show firewall filter IPSEC-TROMBONE
show route table OLD-TROMBONE.inet.0 203.0.113.100/32 exact
show security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32 extensive
show security flow session source-prefix 198.51.100.2/32 destination-prefix 203.0.113.100/32 extensive
show security ike security-associations
show security ipsec security-associations
```

Required evidence:

- `OLD-ESP-TO-OLD` increments for established CE1 and CE2 traffic.
- `OLD-IKE-TO-OLD` increments during IKE, DPD or rekey activity.
- A CE outer-flow session, when present, shows policy `TROMBONE-OUTER-IPSEC`, input `ge-0/0/3.0` and output `ge-0/0/4.0`.
- The session does **not** show input `lo0.0`, output `.local..0`, `Self tunnel type: IPsec` or a self-tunnel ID.
- `NEW-SRX-GW` has no local CE1 or CE2 IKE/IPsec SA.

Filter counters alone prove a match, not successful forwarding. Use the flow session and old-gateway SA/counter state together as the staging gate.

On `OLD-VSRX-GW`:

```text
show security ike security-associations detail
show security ipsec security-associations detail
show security ipsec statistics
```

Both SAs must remain installed, both directions of the protected traffic must pass, and the old-gateway encrypt/decrypt counters must increase.

Optional packet observation:

```text
NEW-SRX-GW> monitor traffic interface ge-0/0/3.0 no-resolve matching "host 203.0.113.100 and (udp port 500 or udp port 4500 or proto 50)"
NEW-SRX-GW> monitor traffic interface ge-0/0/4.0 no-resolve matching "host 203.0.113.100 and (udp port 500 or udp port 4500 or proto 50)"
OLD-VSRX-GW> monitor traffic interface ge-0/0/5.0 no-resolve matching "host 203.0.113.100 and (udp port 500 or udp port 4500 or proto 50)"
```

Native IPsec uses UDP/500 and ESP protocol 50; UDP/4500 must be absent. A zero-packet `monitor traffic` result on a new-SRX data interface is not a failure by itself because transit/PFE traffic may not be copied to the RE. Flow sessions, firewall counters and IPsec statistics are the authoritative evidence.

Confirm the AS1273 commit only after all staging gates pass:

```text
AS1273-EDGE# commit
```

## 15. Phase 1 migration — CE1 only

Run the steps in this order. Do not combine the IPBB route change with the new-SRX peer-state change.

### Step 1 — Record the pre-change state and current indexes

```text
NEW-SRX-GW> show configuration security ike gateway CE1-GATEWAY | display set
NEW-SRX-GW> show configuration security ipsec vpn CE1-VPN | display set
NEW-SRX-GW> show configuration policy-options prefix-list OLD-PEERS | display set
NEW-SRX-GW> show security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32 extensive
OLD-VSRX-GW> show security ike security-associations detail
OLD-VSRX-GW> show security ipsec security-associations detail
IPBB-CORE> show route 10.1.1.0/24 exact
IPBB-CORE> show route 10.2.2.0/24 exact
```

Record the old IKE index and IPsec ID whose peer is `198.51.100.0` and whose VPN is `CE1-VPN`. Mark every CE2 index as **do not clear**. Confirm that the new CE1 gateway/VPN are inactive and that both peer /32s are active in `OLD-PEERS`.

### Step 2 — Atomically enable new termination and remove CE1 from forwarding

On `NEW-SRX-GW`:

```text
configure
activate security ike gateway CE1-GATEWAY
activate security ipsec vpn CE1-VPN
deactivate policy-options prefix-list OLD-PEERS 198.51.100.0/32
show | compare
commit check
commit confirmed 10 comment "Migrate CE1 to new SRX"
```

This must be one commit. Deactivating only the VPN is insufficient: an active IKE gateway can still terminate UDP/500 locally during staging.

### Step 3 — After the commit, clear only CE1's cached new-SRX flow

Run this on `NEW-SRX-GW` only after Step 2 has committed:

```text
clear security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32
show security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32 extensive
```

Clearing the flow before the commit can allow the transit session to be recreated under the old running configuration. After the cutover commit, any recreated CE1 outer session must be local IPsec processing, not policy `TROMBONE-OUTER-IPSEC`.

### Step 4 — Immediately clear only the old CE1 IKE SA

```text
OLD-VSRX-GW> clear security ike security-associations index <CURRENT-CE1-OLD-IKE-INDEX>
```

Recheck the old gateway. If the CE1 child SA remains without its parent IKE SA, verify the peer and VPN name, then clear only its IPsec ID:

```text
OLD-VSRX-GW> show security ipsec security-associations detail
OLD-VSRX-GW> clear security ipsec security-associations index <CE1-OLD-IPSEC-ID>
```

Never issue an unqualified clear and never clear the CE2 index. A temporary `DOWN` old CE1 IKE entry may appear while the old object tries to initiate; it is not proof that the tunnel migrated back.

### Step 5 — Gate on installed CE1 IKE and IPsec SAs on the new SRX

```text
NEW-SRX-GW> show security ike security-associations detail
NEW-SRX-GW> show security ike active-peer detail
NEW-SRX-GW> show security ipsec security-associations detail
NEW-SRX-GW> show security ipsec statistics
NEW-SRX-GW> show interfaces st0.1 extensive
NEW-SRX-GW> show route 10.100.0.10
NEW-SRX-GW> show route 10.1.1.1
OLD-VSRX-GW> show security ike security-associations
OLD-VSRX-GW> show security ipsec security-associations
```

Proceed only when all conditions are true:

- New IKE peer `198.51.100.0` is `UP` with local address/identity `203.0.113.100`.
- `CE1-VPN` has one installed inbound and one installed outbound ESP SA.
- Selectors are local `10.100.0.0/24` and remote `10.1.1.0/24`.
- `OLD-VSRX-GW` no longer has an installed CE1 SA and still has CE2.
- `NEW-SRX-GW` has no CE2 SA.

The preconfigured `establish-tunnels immediately` causes Phase 2 to be negotiated without clearing or changing CE1. Allow up to 60 seconds. If IKE comes up but no Phase 2 SA installs, stop before changing the protected route.

During rekey or migration, CE1 can temporarily display two child-SA pairs under one tunnel. Match the CE-side SPIs to the new-SRX SPIs and verify counters on the new pair. Do not clear working CE SAs merely because the old pair is still ageing out.

### Step 6 — Move CE1 protected return routing on IPBB-CORE

Run this on `IPBB-CORE`—never on `AS1273-EDGE`:

```text
configure
delete routing-options static route 10.1.1.0/24
set routing-options static route 10.1.1.0/24 next-hop 10.44.0.2
commit confirmed 10 comment "Move CE1 protected route to new SRX"
```

Verify:

```text
run show route 10.1.1.0/24 exact
run show route 10.2.2.0/24 exact
```

Required result:

- `10.1.1.0/24` uses new-gateway next hop `10.44.0.2` via `ge-0/0/4.0`.
- `10.2.2.0/24` remains on old-gateway next hop `10.44.0.0` via `ge-0/0/5.0`.

### Step 7 — Validate migrated and control traffic

From the CEs:

```text
CE1> ping 10.100.0.10 source 10.1.1.1 rapid count 1000
CE2> ping 10.100.0.10 source 10.2.2.1 rapid count 1000
```

From the Linux server:

```bash
ping -I 10.100.0.10 -c 1000 10.1.1.1
ping -I 10.100.0.10 -c 1000 10.2.2.1
```

Confirm separation and counter movement:

```text
NEW-SRX-GW> show security ipsec statistics
OLD-VSRX-GW> show security ipsec statistics
NEW-SRX-GW> show firewall filter IPSEC-TROMBONE
NEW-SRX-GW> show security policies hit-count
OLD-VSRX-GW> show security policies hit-count
```

CE1 counters must rise on new; CE2 counters must rise on old. CE2 continues matching `OLD-ESP-TO-OLD`, while CE1 no longer uses the forwarding policy. UDP/4500 must remain absent.

### Step 8 — Optional MTU and re-establishment tests

```bash
ping -I 10.100.0.10 -M do -s 1300 -c 20 10.1.1.1
```

If the approved test plan requires a re-establishment test, first record the new CE1 IKE index and clear only that index:

```text
NEW-SRX-GW> show security ike security-associations
NEW-SRX-GW> clear security ike security-associations index <CE1-NEW-IKE-INDEX>
```

CE1 must reinstall on the new gateway; CE2 must remain stable.

### Step 9 — Confirm every pending commit

After all gates pass:

```text
NEW-SRX-GW# commit
IPBB-CORE# commit
```

Also verify that the earlier AS1273 staging commit was confirmed. A live SA can remain temporarily after an unconfirmed configuration automatically rolls back; SA presence is not proof that the intended configuration is committed.

```text
show system commit
```

Record packet loss, recovery time, final routes, IKE indexes, IPsec IDs and SPIs.

#### Recovery — old CE1 IKE is gone but its child SA remains

Use this only if the new gateway has not installed CE1 and the old child SA is orphaned.

1. On new, atomically restore CE1 forwarding and deactivate local termination:

   ```text
   configure
   activate policy-options prefix-list OLD-PEERS 198.51.100.0/32
   deactivate security ipsec vpn CE1-VPN
   deactivate security ike gateway CE1-GATEWAY
   commit confirmed 10 comment "Recover CE1 to old vSRX"
   ```

2. After that commit, clear the CE1 endpoint flow on new.
3. Verify the orphaned old IPsec ID still belongs to CE1, then clear only that ID.
4. Verify CE1 and CE2 on old, validate protected traffic, and confirm the recovery commit.

Do not clear SAs on CE1 during the normal MoP. A CE-side targeted clear is a lab-recovery action only because the production requirement is no CE change.

## 16. Individual CE1 rollback

Rollback reverses the same three controls without changing CE1.

### Step 1 — Record state and prepare old termination

```text
NEW-SRX-GW> show security ike security-associations detail
NEW-SRX-GW> show security ipsec security-associations detail
OLD-VSRX-GW> show configuration security ike gateway CE1-GATEWAY | display set
OLD-VSRX-GW> show configuration security ipsec vpn CE1-VPN | display set
```

Record the new CE1 IKE index and IPsec ID. Mark CE2 **do not clear**. If the old CE1 gateway or VPN was administratively deactivated after cutover, activate it and commit before continuing.

### Step 2 — Atomically restore forwarding and disable new local termination

On `NEW-SRX-GW`:

```text
configure
activate policy-options prefix-list OLD-PEERS 198.51.100.0/32
deactivate security ipsec vpn CE1-VPN
deactivate security ike gateway CE1-GATEWAY
show | compare
commit check
commit confirmed 10 comment "Rollback CE1 to old vSRX"
```

### Step 3 — After the commit, clear only CE1 state on new

```text
clear security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32
clear security ike security-associations index <CE1-NEW-IKE-INDEX>
```

If the parent IKE SA is already absent but the verified new CE1 child ID remains, clear only that IPsec ID:

```text
show security ipsec security-associations detail
clear security ipsec security-associations index <CE1-NEW-IPSEC-ID>
```

The order matters: commit the forwarding/local-termination state first, clear the cached flow second, then clear the indexed SA.

### Step 4 — Gate on CE1 returning to OLD-VSRX-GW

```text
OLD-VSRX-GW> show security ike security-associations detail
OLD-VSRX-GW> show security ipsec security-associations detail
OLD-VSRX-GW> show security ipsec statistics
NEW-SRX-GW> show security ike security-associations
NEW-SRX-GW> show security ipsec security-associations
```

Require installed CE1 IKE and inbound/outbound IPsec SAs on old before moving the protected route. CE2 must remain installed throughout.

### Step 5 — Return CE1 protected routing on IPBB-CORE

```text
configure
delete routing-options static route 10.1.1.0/24
set routing-options static route 10.1.1.0/24 next-hop 10.44.0.0
commit confirmed 10 comment "Rollback CE1 protected route to old vSRX"
```

### Step 6 — Validate and confirm rollback

```text
CE1> ping 10.100.0.10 source 10.1.1.1 rapid count 1000
CE2> ping 10.100.0.10 source 10.2.2.1 rapid count 1000
```

```bash
ping -I 10.100.0.10 -c 1000 10.1.1.1
ping -I 10.100.0.10 -c 1000 10.2.2.1
```

Expected final rollback state:

- CE1 and CE2 terminate on `OLD-VSRX-GW`.
- `NEW-SRX-GW` has no CE1/CE2 SA.
- Both peers are active in `OLD-PEERS`.
- Both protected routes use old next hop `10.44.0.0`.

Confirm pending commits and record the evidence:

```text
NEW-SRX-GW# commit
IPBB-CORE# commit
```

### Rollback and hard-stop criteria

Rollback immediately, or allow the applicable `commit confirmed` to expire, if:

- CE2 loses traffic or its IKE/IPsec SA.
- CE1 does not install IKE and IPsec on the intended gateway within 60 seconds.
- A CE1 session has the wrong transit/local disposition.
- An IKE authentication, proposal or selector error appears.
- The protected route cannot be validated with more than two minutes remaining.
- An SA cannot be identified safely by peer and index.

Never use a broad clear, change CE configuration, or move the protected route before the destination gateway has an installed CE1 Phase 2 SA.

## 17. Required Phase 1 test set

| ID | Test | Pass condition |
|---|---|---|
| P1-01 | Underlay reachability | Every directly connected lab link is reachable |
| P1-02 | Old-gateway baseline | CE1 and CE2 pass bidirectional protected traffic on `OLD-VSRX-GW` |
| P1-03 | Native IPsec | IKE uses UDP/500, data uses ESP/50 and UDP/4500 is absent |
| P1-04 | Shared endpoint attraction | AS1273 sends only `203.0.113.100/32` to `NEW-SRX-GW` |
| P1-05 | Staging object state | New CE1 gateway/VPN are inactive and both peers are active in `OLD-PEERS` |
| P1-06 | Trombone disposition | CE sessions show `TROMBONE-OUTER-IPSEC`, ingress new Internet link and egress new IPBB link |
| P1-07 | No local self-session | No listed peer uses `lo0.0`, `.local..0` or a self-tunnel on new |
| P1-08 | No CE change | CE1 peer address, selectors, PSK and local routing remain unchanged |
| P1-09 | Atomic cutover | CE1 gateway/VPN activation and peer-list removal occur in one commit |
| P1-10 | New Phase 2 | New CE1 has installed inbound/outbound IPsec SAs before routing changes |
| P1-11 | Protected-route ownership | Only IPBB moves `10.1.1.0/24`; AS1273 owns only the shared endpoint route |
| P1-12 | Control continuity | CE2 remains established and passing on old throughout |
| P1-13 | Bidirectional data | CE1 passes through new and CE2 through old after cutover |
| P1-14 | Re-establishment | A targeted CE1 clear returns it to the selected gateway |
| P1-15 | MTU | A 1300-byte DF ping succeeds through migrated CE1 |
| P1-16 | Individual rollback | CE1 returns to old without moving CE2 |
| P1-17 | Commit safety | Every confirmed commit is explicitly confirmed or allowed to roll back deliberately |
| P1-18 | Measured disruption | Packet loss and recovery time are recorded for migration and rollback |

Phase 1 is successful only when all required tests pass. Batch migration is a separate Phase 2 activity.

## 18. Troubleshooting checkpoints

### 18.1 OLD-TROMBONE route is inactive or resolves locally

```text
show route table OLD-TROMBONE.inet.0 10.44.0.2/31 exact
show route table OLD-TROMBONE.inet.0 203.0.113.100/32 exact
show configuration routing-options rib-groups | display set
show configuration policy-options policy-statement TROMBONE-IMPORT | display set
```

The forwarding table must contain the IPBB connected subnet and a static endpoint route through `10.44.0.3`. If the endpoint appears direct/local, correct the import policy before continuing.

### 18.2 A peer in OLD-PEERS terminates locally on new

This is the most important staging fault found in the lab. Inspect the peer-specific session:

```text
show security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32 extensive
```

Incorrect local processing is indicated by `Interface: lo0.0`, output `.local..0`, `Self tunnel type: IPsec` or a self-tunnel ID. The cause is normally an active local CE1 IKE gateway/VPN during staging, or a flow created before the correct configuration committed.

Fix:

1. Ensure CE1 is active in `OLD-PEERS`.
2. Deactivate both `CE1-VPN` and `CE1-GATEWAY` on new in one commit.
3. After the commit, clear only the CE1 endpoint flow.
4. Require a `TROMBONE-OUTER-IPSEC` flow from `ge-0/0/3.0` to `ge-0/0/4.0` before continuing.

### 18.3 FBF counters do not increment

```text
show interfaces filters
show firewall filter IPSEC-TROMBONE
show configuration policy-options prefix-list OLD-PEERS | display inheritance no-comments
```

Check the peer source, shared destination, input interface and protocol. With NAT-T disabled, expect UDP/500 and ESP rather than UDP/4500. Counters are not sufficient on their own; verify the flow disposition.

### 18.4 A listed peer does not reach OLD-VSRX-GW

Check:

1. AS1273 endpoint route uses `192.0.2.3`.
2. New FBF matches and `OLD-TROMBONE.inet.0` resolves through `10.44.0.3`.
3. IPBB route `203.0.113.100/32` points through `10.44.0.0`.
4. Old `ge-0/0/5.0` permits IKE and the transport intrazone policy permits forwarding.
5. Old has a route back toward the CE peer through AS1273.

### 18.5 CE1 does not establish on NEW-SRX-GW

Confirm:

- `CE1-GATEWAY` and `CE1-VPN` are active.
- CE1 `198.51.100.0/32` is inactive in `OLD-PEERS`; CE2 remains active.
- The new CE1 transit flow was cleared after the cutover commit.
- The old CE1 IKE SA was cleared by its current index.
- AS1273 still sends `203.0.113.100/32` to `192.0.2.3`.
- IKE host-inbound, local address/identity and native-IPsec settings are correct.

```text
show security ike security-associations detail
show security ike active-peer detail
show security ipsec security-associations detail
show security ipsec inactive-tunnels detail
show security flow session source-prefix 198.51.100.0/32 destination-prefix 203.0.113.100/32 extensive
show log kmd
```

### 18.6 IKE is up but Phase 2 is absent or fails

Verify `CE1-VPN` is active and contains:

```text
set security ipsec vpn CE1-VPN establish-tunnels immediately
```

Then compare selectors exactly:

| Node | Local selector | Remote selector |
|---|---|---|
| CE1 | `10.1.1.0/24` | `10.100.0.0/24` |
| OLD-VSRX-GW CE1 | `10.100.0.0/24` | `10.1.1.0/24` |
| NEW-SRX-GW CE1 | `10.100.0.0/24` | `10.1.1.0/24` |
| CE2 | `10.2.2.0/24` | `10.100.0.0/24` |
| OLD-VSRX-GW CE2 | `10.100.0.0/24` | `10.2.2.0/24` |

Also compare PFS, ESP authentication, encryption and lifetime. Do not move the protected route while Phase 2 is absent.

### 18.7 SAs are installed but protected traffic fails

Verify:

1. CE route `10.100.0.0/24` through `st0.0`.
2. Selected gateway route to the CE prefix through the correct `st0` unit.
3. Gateway route to `10.100.0.0/24` through IPBB.
4. `IPBB-CORE`—not AS1273—has `10.1.1.0/24` through the selected gateway.
5. Linux routes use `eth1` through `10.100.0.1`.
6. Relevant SRX security-policy counters increment.

```text
show security policies hit-count
show security flow session
show security ipsec statistics
show route 10.100.0.0/24 exact
show route 10.1.1.0/24 exact
show route 10.2.2.0/24 exact
```

On the server:

```bash
ip route get 10.1.1.1
ip route get 10.2.2.1
tcpdump -ni eth1 icmp
```

### 18.8 A commit rolled back but SAs still appear up

Use `show system commit` and inspect running configuration. An IKE/IPsec SA can outlive an automatic configuration rollback until rekey, timeout or a targeted clear. Do not infer committed control-plane state from an existing SA. Reapply the intended configuration with `commit confirmed 10`, validate it, and explicitly `commit` before expiry.

## 19. Validated Phase 1 lab result

The attached lab evidence demonstrates a successful single-tunnel migration with no CE configuration change.

| Check | Observed result |
|---|---|
| CE1 termination | `NEW-SRX-GW`, IKE peer `198.51.100.0`, IKE index `1247201`, state `UP` |
| CE1 Phase 2 | `CE1-VPN`, IPsec ID `67108867`, inbound SPI `4eec51cd`, outbound SPI `fa058235` |
| CE1 selectors | Local `10.100.0.0/24`, remote `10.1.1.0/24` |
| CE1 traffic | 10/10 CE1-to-server pings passed; new encrypt/decrypt counters increased with no errors |
| CE2 control | 100/100 CE2-to-server pings passed; CE2 remained on `OLD-VSRX-GW` |
| Old gateway final SA state | Only CE2 remained: peer `198.51.100.2`, IPsec ID `67108866` |
| IPBB split routing | CE1 `10.1.1.0/24` via new `10.44.0.2`; CE2 `10.2.2.0/24` via old `10.44.0.0` |

CE1 temporarily displayed two child-SA pairs under one IKE tunnel. The new-SRX SPIs matched the newer CE-side pair in the opposite directions, and traffic/counters proved the new path. The older pair was allowed to age out; it was not broadly cleared.

The result validates the Phase 1 objective: shared endpoint unchanged, CE configuration unchanged, CE1 migrated individually, CE2 preserved on old, and independent protected-route control maintained.

## 20. Production interpretation

The PoC proves the migration mechanism, not the final production design.

| PoC mechanism | Production equivalent |
|---|---|
| AS1273 static endpoint route | BGP advertisement and preferred MED from the new PX/SRX |
| `OLD-PEERS` plus local gateway/VPN state | Per-peer migration inventory and local-termination state |
| New-SRX forwarding instance | Temporary GIPN/IPBB path from new gateway to old gateway |
| IPBB static protected routes | L3VPN route/interface activation on the selected gateway |
| Atomic CE1 activation/list removal | Individual customer migration action |
| Atomic list restoration/local deactivation | Individual customer rollback action |

Before production use, separately validate the physical SRX4120 software release, FBF and IPsec scale, redundancy, routing convergence, restricted security policy, automation, indexed state clearing and commit-confirmation workflow.

## 21. Reference basis

- PX IPsec migration Method of Procedure, version 0.3.
- [Juniper filter-based forwarding overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html)
- [Juniper filter-based forwarding configuration example](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html)
- [Juniper FBF feature support](https://apps.juniper.net/feature-explorer/feature/5208)
- [Juniper IKE gateway configuration and local-address requirements](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/security-edit-gateway-ike.html)
- [Juniper scale-out IPsec example using `lo0.0` as the IKE gateway](https://www.juniper.net/documentation/us/en/software/jvd/jvd-scale-out-ipsec-solution-for-enterprises/solution_architecture.html)
- [Juniper traffic selectors in route-based VPNs](https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-traffic-selectors-in-route-based-vpns.html)
- [Juniper route-based IPsec VPN verification](https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-route-based-ipsec-vpns.html)
- [Juniper NAT-T behaviour](https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-route-based-and-policy-based-vpns-with-nat-t.html)
- [Juniper SRX flow-based processing and sessions](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-flow-based-session-for-srx-series-devices.html)
- [Juniper `show security ike security-associations`](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-security-ike-security-associations.html)
- [Juniper `show security ipsec security-associations`](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-security-ipsec-security-associations.html)
- [Juniper targeted flow-session clear command](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-security-flow-session-all.html)
- [Juniper targeted IKE SA clear command](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-security-ike-security-associations.html)
- [Juniper targeted IPsec SA clear command](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-security-ipsec-security-associations.html)
