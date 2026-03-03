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
