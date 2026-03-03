# gns3-mpls-inter-as-lab

<img width="1256" height="1021" alt="image" src="https://github.com/user-attachments/assets/b5aac46f-d5d6-4b88-aba9-1364def308cb" />

In this lab we will create a topology where 2 different ASes are connected to each other as described in https://datatracker.ietf.org/doc/html/rfc4364#section-10

# Technical Study: MPLS VPN Inter-AS Option B Label Swapping
**Topic:** How ASBR2 processes and forwards VPN traffic using a two-level label stack.
**Scenario:** A packet arrives at PE-ASBR2 from an external AS (via a GRE tunnel) with **Local Label 23** and must be delivered to an internal PE.

---

## 1. High-Level Logic Flow
When an ASBR acts as the gateway in an Option B setup, it must perform a "Look-and-Swap" operation. It translates the external VPN label into an internal VPN label and then "stacks" a transport label on top to reach the final destination.



### The Step-by-Step Packet Transformation
1. **Arrival**: ASBR2 receives a packet with **Label 23**.
2. **Identification**: ASBR2 identifies that Label 23 belongs to **VRF 2** (RD 100:2) for destination **10.20.1.0/24**.
3. **BGP Lookup**: ASBR2 determines that the BGP Next-Hop is **3.3.3.3** and the remote VPN label is **20**.
4. **Transport Lookup**: ASBR2 determines that the transport label to reach **3.3.3.3** is **17**.
5. **Execution**: ASBR2 swaps **23 → 20** and then pushes **17** on top.
6. **Forwarding**: The packet leaves with stack **[17, 20]**.

---

## 2. Technical Confirmation (The Proof)

To verify this flow in the router console, follow these three commands.

### Step A: Identify the "Incoming" Instruction
We check the Label Forwarding Information Base (LFIB) to see what the router does with Label 23.
* **Command:** `show mpls forwarding-table | include 23`
* **Output:**
```text
Local      Outgoing   Prefix              Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id        Switched      interface
23         20         100:2:10.20.1.0/24  610           Gi2/0      10.0.23.2
```

Observation: This proves that Label 23 is mapped to Outgoing Label 20 and is heading toward the internal next-hop 10.0.23.2.

### Step B: Confirm the VPN "BGP" Instruction

We verify why the router chose Label 20. This is the Control Plane instruction.
* `Command: show ip bgp vpnv4 all 10.20.1.0/24`
*  **Output**
```
BGP routing table entry for 100:2:10.20.1.0/24, version 5
  Paths: (1 available, best #1)
    3.3.3.3 (metric 3) from 3.3.3.3 (3.3.3.3)
      mpls labels in/out 23/20
```
Observation: This confirms that the internal PE (3.3.3.3) is the one that requested Label 20 for this specific VRF.

To get the packet to the BGP Next-Hop (3.3.3.3), the router needs a transport label from the internal IGP/LDP.

* Command: show mpls forwarding-table | include 3.3.3.3

* Output:
```
19         17         3.3.3.3/32          0             Gi2/0      10.0.23.2
```

### Final forwarding stack
```
[17][20][IP]
```

# How ASBR2 Tells ASBR1 Which VPN Label to Use (Inter-AS Option B)

This explains how ASBR2 informs ASBR1 which MPLS VPN label must be used when sending traffic toward ASBR2.

This is purely a **BGP control-plane process**.  
No LDP runs across the AS boundary in Option B.

---

## 1️⃣ Route Origination Inside AS200

PE2 originates a VPN route:

Example:

- RD: 100:2
- Prefix: 10.20.1.0/24
- VPN label: 20
- Next-hop: 3.3.3.3
- RT: 100:2

On PE2:
```
PE2# show bgp vpnv4 unicast all 10.20.1.0/24

BGP routing table entry for 100:2:10.20.1.0/24
Paths: (1 available, best #1)
Local
3.3.3.3 (metric 0) from 3.3.3.3 (3.3.3.3)
Origin IGP, localpref 100, valid, best
Extended Community: RT:100:2
MPLS label: 20
```

This route is advertised via iBGP VPNv4 to ASBR2.

---

## 2️⃣ ASBR2 Installs the Route

ASBR2 receives the VPNv4 route from PE2.
```
ASBR2# show bgp vpnv4 unicast all 10.20.1.0/24

BGP routing table entry for 100:2:10.20.1.0/24
Paths: (1 available, best #1)
200
3.3.3.3 (metric 0) from 3.3.3.3 (3.3.3.3)
Origin IGP, valid, internal, best
Extended Community: RT:100:2
MPLS label: 20
```
ASBR2:
- Imports route into VRF (based on RT)
- Resolves next-hop 3.3.3.3 via IGP
- Resolves transport label via LDP
- Prepares to advertise to ASBR1

---

## 3️⃣ ASBR2 Allocates a New VPN Label for Inter-AS Advertisement

ASBR2 does NOT send PE2’s internal label (20) to ASBR1.

Instead, it allocates a locally significant label for ASBR1 to use.

Example: label 23


This means:

"If ASBR1 sends traffic with label 23, I (ASBR2) will know it belongs to 100:2:10.20.1.0/24."

---

## 4️⃣ ASBR2 Advertises the Route to ASBR1 (eBGP VPNv4)

ASBR2 sends a BGP UPDATE with:

- MP_REACH_NLRI
- RD
- Prefix
- VPN label (23)
- Next-hop
- Route Target

On ASBR1:
```
ASBR1# show bgp vpnv4 unicast all 10.20.1.0/24

BGP routing table entry for 100:2:10.20.1.0/24
Paths: (1 available, best #1)
200
10.255.255.2 (metric 0) from 10.255.255.2 (2.2.2.2)
Origin IGP, valid, external, best
Extended Community: RT:100:2
MPLS label: 23
```


This proves:

ASBR2 told ASBR1 to use VPN label 23.

The label is carried inside the MP_REACH_NLRI attribute.

---

## 5️⃣ ASBR1 Programs Its LFIB

ASBR1 now installs forwarding entry:
```
PE-ASBR1#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         Pop Label  10.255.255.2/32  0             Tu0        point2point
17         Pop Label  2.2.2.2/32       0             Gi1/0      10.0.13.1
18         17         1.1.1.1/32       0             Gi1/0      10.0.13.1
19         Pop Label  10.0.12.0/30     0             Gi1/0      10.0.13.1
20         19         100:1:10.10.1.0/24   \
                                       0             Gi1/0      10.0.13.1
21         20         100:2:10.10.1.0/24   \
                                       1830          Gi1/0      10.0.13.1
22         21         100:1:10.20.1.0/24   \
                                       0             Tu0        point2point
23         23         100:2:10.20.1.0/24   \
                                       1920          Tu0        point2point
```


And if we inspect the BGP detail:

```
PE-ASBR1#show bgp vpnv4 unicast all 10.20.1.0/24
BGP routing table entry for 100:1:10.20.1.0/24, version 4
Paths: (1 available, best #1, no table)
  Advertised to update-groups:
     2
  Refresh Epoch 1
  200 65002
    10.255.255.2 from 10.255.255.2 (22.22.22.22)
      Origin IGP, localpref 100, valid, external, best
      Extended Community: RT:100:1
      mpls labels in/out 22/21
      rx pathid: 0, tx pathid: 0x0
BGP routing table entry for 100:2:10.20.1.0/24, version 5
Paths: (1 available, best #1, no table)
  Advertised to update-groups:
     2
  Refresh Epoch 1
  200 65004
    10.255.255.2 from 10.255.255.2 (22.22.22.22)
      Origin IGP, localpref 100, valid, external, best
      Extended Community: RT:100:2
      mpls labels in/out 23/23
      rx pathid: 0, tx pathid: 0x0
```


This means:

When sending traffic toward ASBR2 for that prefix,
ASBR1 must push label 23.

---

## 6️⃣ Data Plane Behavior

When CE traffic reaches ASBR1:

ASBR1 sends:
[23][IP]

toward ASBR2.

ASBR2 receives:
`Incoming label: 23`


On ASBR2:

```
PE-ASBR2#show mpls forwarding-table | include 23
17         Pop Label  4.4.4.4/32       0             Gi2/0      10.0.23.2
18         Pop Label  10.0.24.0/30     0             Gi2/0      10.0.23.2
19         17         3.3.3.3/32       0             Gi2/0      10.0.23.2
                                       0             Gi2/0      10.0.23.2
23         20         100:2:10.20.1.0/24   \
                                       1830          Gi2/0      10.0.23.2
```


Meaning:

Incoming 23 → swap to 20 → forward inside AS200.

ASBR2 then pushes its internal transport label (via LDP) and forwards.

---

## 🔷 Key Proof Points

✔ The VPN label (23) is learned via BGP.  
✔ It is visible in `show bgp vpnv4 unicast all`.  
✔ It appears in LFIB via `show mpls forwarding-table`.  
✔ No LDP is used across AS boundary.  
✔ Route Target is used only for import policy — not forwarding.  

---

## 🔷 One-Line Summary

ASBR2 tells ASBR1 which VPN label to use by advertising it inside the BGP VPNv4 MP_REACH_NLRI attribute.  
ASBR1 installs that label and uses it when sending data traffic toward ASBR2.
