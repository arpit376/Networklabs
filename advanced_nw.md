

# Advanced Linux Network Lab: Bridges, NAT, Policy Routing, VXLAN

This lab extends a basic Linux network namespace setup by adding a virtual switch, a NAT firewall router, policy-based routing, and a VXLAN overlay. It is designed to run on a single Linux host with `iproute2`, `iptables` (or `nftables`), `ping`, and optionally `tcpdump`.

---

## Part 1: L2 switch with bridge and multiple hosts

**Objective:** Use a Linux bridge to emulate a switch and connect three namespaces on the same Layer 2 segment.

### Topology

- Namespaces: `host1`, `host2`, `host3`
- Root namespace: Linux bridge `br0`
- Each host connected to `br0` via a veth pair
- Subnet: `192.168.10.0/24`

### Steps

1. Create host namespaces:

```
sudo ip netns add host1
sudo ip netns add host2
sudo ip netns add host3

sudo ip netns exec host1 ip link set lo up
sudo ip netns exec host2 ip link set lo up
sudo ip netns exec host3 ip link set lo up
```

2. Create and bring up bridge:


```

sudo ip link add br0 type bridge
sudo ip link set br0 up

```

3. For each host, create a veth pair and attach to bridge:

```


# host1

sudo ip link add veth1 type veth peer name veth1h
sudo ip link set veth1h netns host1
sudo ip link set veth1 master br0
sudo ip link set veth1 up
sudo ip netns exec host1 ip link set veth1h up

# host2

sudo ip link add veth2 type veth peer name veth2h
sudo ip link set veth2h netns host2
sudo ip link set veth2 master br0
sudo ip link set veth2 up
sudo ip netns exec host2 ip link set veth2h up

# host3

sudo ip link add veth3 type veth peer name veth3h
sudo ip link set veth3h netns host3
sudo ip link set veth3 master br0
sudo ip link set veth3 up
sudo ip netns exec host3 ip link set veth3h up

```

4. Assign IP addresses:

```

sudo ip netns exec host1 ip addr add 192.168.10.11/24 dev veth1h
sudo ip netns exec host2 ip addr add 192.168.10.12/24 dev veth2h
sudo ip netns exec host3 ip addr add 192.168.10.13/24 dev veth3h

```

5. Test connectivity:

```

sudo ip netns exec host1 ping -c 3 192.168.10.12
sudo ip netns exec host2 ping -c 3 192.168.10.13

```

### Tasks

- Inspect ARP and neighbors:

```

sudo ip netns exec host1 ip neigh
sudo tcpdump -i br0 -n

```

- Bring an interface down and observe:

```

sudo ip link set veth2 down
sudo ip netns exec host1 ping -c 3 192.168.10.12

```

---

## Part 2: Router + NAT firewall namespace

**Objective:** Insert a router namespace between the L2 “LAN” and an “internet” namespace and configure NAT.

### Topology

- LAN: `host1`, `host2`, `host3` on `192.168.10.0/24`
- Router namespace: `router`
- WAN namespace: `wan`
- Router connects to `br0` (LAN side) and `wan` (WAN side)

**IP plan:**

- Router LAN: `192.168.10.1/24`
- Router WAN: `203.0.113.1/24`
- WAN host: `203.0.113.2/24`

### Steps

1. Create `router` and `wan` namespaces:

```

sudo ip netns add router
sudo ip netns add wan

sudo ip netns exec router ip link set lo up
sudo ip netns exec wan ip link set lo up

```

2. Connect router to bridge:

```

sudo ip link add vethr-br type veth peer name vethr
sudo ip link set vethr netns router
sudo ip link set vethr-br master br0
sudo ip link set vethr-br up
sudo ip netns exec router ip link set vethr up
sudo ip netns exec router ip addr add 192.168.10.1/24 dev vethr

```

3. Connect router to WAN:

```

sudo ip link add veth-wan type veth peer name veth-wanh
sudo ip link set veth-wan netns router
sudo ip link set veth-wanh netns wan

sudo ip netns exec router ip link set veth-wan up
sudo ip netns exec wan ip link set veth-wanh up

sudo ip netns exec router ip addr add 203.0.113.1/24 dev veth-wan
sudo ip netns exec wan ip addr add 203.0.113.2/24 dev veth-wanh

```

4. Set default routes:

On LAN hosts:

```

sudo ip netns exec host1 ip route add default via 192.168.10.1
sudo ip netns exec host2 ip route add default via 192.168.10.1
sudo ip netns exec host3 ip route add default via 192.168.10.1

```

On `wan`:

```

sudo ip netns exec wan ip route add default via 203.0.113.1

```

5. Enable forwarding on router:

```

sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1

```

6. Configure NAT on router (iptables example):

```

sudo ip netns exec router iptables -t nat -A POSTROUTING -o veth-wan -j MASQUERADE

```

7. Test connectivity:

```

sudo ip netns exec host1 ping -c 3 203.0.113.2

```

### Additional NAT task

Expose a TCP service on `host2` (e.g., port 8080) to the WAN via DNAT:

```


# Assume host2 runs a service on 192.168.10.12:8080

sudo ip netns exec router iptables -t nat -A PREROUTING -i veth-wan -p tcp --dport 8080 \
-j DNAT --to-destination 192.168.10.12:8080

```

---

## Part 3: Policy-based routing with multiple uplinks

**Objective:** Give the router two WAN links and route traffic based on source address using `ip rule` and multiple routing tables.

### Topology

- Existing LAN and first WAN (WAN1).
- Additional WAN namespace: `wan2`.

**New IP plan:**

- WAN1: `203.0.113.0/24`
  - Router WAN1: `203.0.113.1/24`
  - WAN1 host: `203.0.113.2/24`
- WAN2: `198.51.100.0/24`
  - Router WAN2: `198.51.100.1/24`
  - WAN2 host: `198.51.100.2/24`

### Steps

1. Create `wan2` and link to router:

```

sudo ip netns add wan2
sudo ip netns exec wan2 ip link set lo up

sudo ip link add veth-wan2 type veth peer name veth-wan2h
sudo ip link set veth-wan2 netns router
sudo ip link set veth-wan2h netns wan2

sudo ip netns exec router ip link set veth-wan2 up
sudo ip netns exec wan2 ip link set veth-wan2h up

sudo ip netns exec router ip addr add 198.51.100.1/24 dev veth-wan2
sudo ip netns exec wan2 ip addr add 198.51.100.2/24 dev veth-wan2h
sudo ip netns exec wan2 ip route add default via 198.51.100.1

```

2. Define routing table IDs on router (e.g. in `/etc/iproute2/rt_tables`):

```

100 wan1
200 wan2

```

3. Configure per-table defaults:

```


# Table 100 via WAN1

sudo ip netns exec router ip route add default via 203.0.113.2 dev veth-wan table 100

# Table 200 via WAN2

sudo ip netns exec router ip route add default via 198.51.100.2 dev veth-wan2 table 200

```

4. Add policy rules based on source addresses:

```


# Traffic from host1 uses table 100

sudo ip netns exec router ip rule add from 192.168.10.11/32 table 100

# Traffic from host2 uses table 200

sudo ip netns exec router ip rule add from 192.168.10.12/32 table 200

```

5. Verify rules and tables:

```

sudo ip netns exec router ip rule list
sudo ip netns exec router ip route show table 100
sudo ip netns exec router ip route show table 200

```

### Tasks

- From `host1` and `host2`, generate outbound traffic and capture on both WAN links:

```


# In router namespace, two terminals

sudo ip netns exec router tcpdump -i veth-wan -n
sudo ip netns exec router tcpdump -i veth-wan2 -n

```

- Confirm that `host1` traffic exits via WAN1 and `host2` traffic exits via WAN2.

---

## Part 4: Overlay experiment with VXLAN

**Objective:** Stretch an L2 segment over an IP underlay using VXLAN between two bridges.

### Concept

- Two “sites” (`siteA`, `siteB`), each with:
- A bridge (`brA`, `brB`)
- One or more local host namespaces
- Underlay IP connectivity between `siteA` and `siteB`
- VXLAN interfaces (same VNI) on each site connected to the underlay and bound to local bridges, forming one logical L2 segment.

### High-level steps (simplified sketch)

1. Create `siteA` and `siteB` namespaces and, inside each, create a bridge and connect local hosts via veth pairs.

2. Create an underlay veth pair between `siteA` and `siteB`, configure underlay IPs (e.g., `10.100.0.1/30` on `siteA`, `10.100.0.2/30` on `siteB`), and set appropriate routes.

3. In `siteA`, create VXLAN device and attach to `brA`:

```

ip netns exec siteA ip link add vxlanA type vxlan id 42 \
dev <underlay-if-A> remote <underlay-ip-B> dstport 4789
ip netns exec siteA ip link set vxlanA up
ip netns exec siteA ip link set vxlanA master brA

```

4. In `siteB`, create VXLAN device and attach to `brB`:

```

ip netns exec siteB ip link add vxlanB type vxlan id 42 \
dev <underlay-if-B> remote <underlay-ip-A> dstport 4789
ip netns exec siteB ip link set vxlanB up
ip netns exec siteB ip link set vxlanB master brB

```

5. Put hosts at both sites into the same subnet (e.g., `10.200.0.0/24`) and test that cross-site pings work, showing a stretched L2 domain over the IP underlay.

---

## Part 5: Reporting and verification

For each part, capture:

- A small topology diagram (ASCII or drawn).
- Key command outputs:
- `ip netns list`
- `ip link`
- `ip addr`
- `ip route`
- `ip rule` (for policy routing)
- `iptables -L -v -t nat` or `nft list ruleset` (if using nftables)
- Short observations:
- What changed when routes or rules were added/removed?
- Which interfaces saw which packets in `tcpdump`?
- How forwarding, NAT, and policy routing interact in each scenario.

---

## Cleanup

Adjust as needed depending on what you created; for example:

```

sudo ip netns del host1
sudo ip netns del host2
sudo ip netns del host3
sudo ip netns del router
sudo ip netns del wan
sudo ip netns del wan2
sudo ip netns del siteA
sudo ip netns del siteB

# Remove bridge(s) and any leftover veths in the root namespace

sudo ip link del br0

# similarly: ip link del brA, brB, or any other devices you added

```

Run cleanup only when you are sure you no longer need the lab state.
```



