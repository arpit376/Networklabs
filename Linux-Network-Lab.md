# Linux Network Creation Lab Plan

## Lab Overview and Prerequisites

- **System**: Any modern Linux with `iproute2` (`ip` command) and `ping` available.
- **Goal**: By the end, you will create virtual hosts, connect them with virtual links, and configure routes so they can communicate.

---

## Step 1: Explore Existing Networking

Begin by understanding your current network setup:

1. **List interfaces and their state**:
   ```bash
   ip link
   ip addr
   ```

2. **Check current routes**:
   ```bash
   ip route
   ```
   Identify: default gateway, directly connected networks.

3. **Verify connectivity**:
   ```bash
   ping <gateway>
   ping 8.8.8.8
   ```

**Focus Points**: Interface names, IP/mask notation (CIDR), default route meaning.

---

## Step 2: Learn Key `ip` Subcommands

Practice on the real namespace (not yet using `ip netns`):

### Adding and Removing IP Addresses

Add a temporary IP to an interface:
```bash
sudo ip addr add 192.168.100.10/24 dev eth0
```

Remove the IP:
```bash
sudo ip addr del 192.168.100.10/24 dev eth0
```

### Bring Interfaces Up and Down

```bash
sudo ip link set eth0 up
sudo ip link set eth0 down
```

### Add and Remove Routes

Add a static route:
```bash
sudo ip route add 10.10.10.0/24 via <gateway-ip>
```

Remove the route:
```bash
sudo ip route del 10.10.10.0/24
```

**Goal**: Understand how address assignment and routing table entries affect reachability.

---

## Step 3: Introduce Network Namespaces ("Virtual Hosts")

Network namespaces provide isolated network stacks, allowing you to run multiple "virtual hosts" on a single machine.

### Background

- Each namespace has its own interfaces, IP addresses, routing tables, and firewall rules.
- Namespaces are created on-demand and can be removed when done.

### Create Namespaces

```bash
sudo ip netns add hostA
sudo ip netns add hostB
```

### Enable Loopback Interface

Each namespace needs its loopback interface enabled:

```bash
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostB ip link set lo up
```

### Verification

List all namespaces:
```bash
ip netns list
```

Inspect interfaces in a namespace:
```bash
sudo ip netns exec hostA ip addr
sudo ip netns exec hostB ip addr
```

**Key Concept**: Each namespace is completely isolated; `hostA` cannot see interfaces or routes in `hostB` by default.

---

## Step 4: Create a Point-to-Point Virtual Link

### Objective

Connect `hostA` and `hostB` directly using a virtual Ethernet pair (veth).

### Create a Virtual Ethernet Pair

```bash
sudo ip link add vethA type veth peer name vethB
```

This creates two interfaces: `vethA` and `vethB`, which act like a virtual Ethernet cable.

### Move Interfaces to Namespaces

```bash
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB
```

Now `vethA` belongs to `hostA`'s namespace and `vethB` to `hostB`'s namespace.

### Assign IP Addresses

```bash
sudo ip netns exec hostA ip addr add 10.0.0.1/24 dev vethA
sudo ip netns exec hostB ip addr add 10.0.0.2/24 dev vethB
```

### Bring Interfaces Up

```bash
sudo ip netns exec hostA ip link set vethA up
sudo ip netns exec hostB ip link set vethB up
```

### Test Connectivity

Ping from `hostA` to `hostB`:

```bash
sudo ip netns exec hostA ping -c 3 10.0.0.2
```

You should see successful ICMP echo replies. This confirms that `hostA` and `hostB` are directly connected.

---

## Step 5: Build a Routed Three-Node Topology

### Objective

Create a more complex topology with two hosts connected through a router namespace:

```
hostA --- router --- hostB
10.0.1.0/24  |  10.0.2.0/24
```

### Step 5.1: Create the Router Namespace

```bash
sudo ip netns add router
sudo ip netns exec router ip link set lo up
```

### Step 5.2: Create Two Virtual Links

Link between `hostA` and `router`:
```bash
sudo ip link add vethA type veth peer name vethAR
sudo ip link set vethA netns hostA
sudo ip link set vethAR netns router
```

Link between `router` and `hostB`:
```bash
sudo ip link add vethB type veth peer name vethBR
sudo ip link set vethB netns hostB
sudo ip link set vethBR netns router
```

### Step 5.3: Assign IP Addresses

**On hostA:**
```bash
sudo ip netns exec hostA ip addr add 10.0.1.2/24 dev vethA
sudo ip netns exec hostA ip link set vethA up
```

**On router (interface facing hostA):**
```bash
sudo ip netns exec router ip addr add 10.0.1.1/24 dev vethAR
sudo ip netns exec router ip link set vethAR up
```

**On router (interface facing hostB):**
```bash
sudo ip netns exec router ip addr add 10.0.2.1/24 dev vethBR
sudo ip netns exec router ip link set vethBR up
```

**On hostB:**
```bash
sudo ip netns exec hostB ip addr add 10.0.2.2/24 dev vethB
sudo ip netns exec hostB ip link set vethB up
```

### Step 5.4: Verify Local Connectivity

Test that each host can reach its directly connected router interface:

```bash
sudo ip netns exec hostA ping -c 3 10.0.1.1
sudo ip netns exec hostB ping -c 3 10.0.2.1
```

Both should succeed.

### Step 5.5: Inspect Routing Tables

```bash
sudo ip netns exec hostA ip route
sudo ip netns exec hostB ip route
sudo ip netns exec router ip route
```

Notice: Each namespace only knows about its directly connected networks.

### Step 5.6: Enable IP Forwarding on the Router

Turn the `router` namespace into a functioning L3 router by enabling packet forwarding:

```bash
sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1
```

Verify:
```bash
sudo ip netns exec router sysctl net.ipv4.ip_forward
```

Output should be: `net.ipv4.ip_forward = 1`

### Step 5.7: Add Static Routes

**On hostA** (route to hostB's network via router):
```bash
sudo ip netns exec hostA ip route add 10.0.2.0/24 via 10.0.1.1
```

**On hostB** (route to hostA's network via router):
```bash
sudo ip netns exec hostB ip route add 10.0.1.0/24 via 10.0.2.1
```

### Step 5.8: Verify Routing Tables

```bash
sudo ip netns exec hostA ip route
sudo ip netns exec hostB ip route
sudo ip netns exec router ip route
```

You should now see the new static routes in `hostA` and `hostB`.

### Step 5.9: Test End-to-End Connectivity

**Ping from hostA to hostB:**
```bash
sudo ip netns exec hostA ping -c 4 10.0.2.2
```

**Ping from hostB to hostA:**
```bash
sudo ip netns exec hostB ping -c 4 10.0.1.2
```

Both should succeed, with the `router` forwarding packets between the two networks.

---

## Step 6: Observation and Troubleshooting Tasks

### Task 6.1: Route Modification and Impact

1. Verify that `hostA` can ping `10.0.2.2`:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```

2. Delete the route from `hostA`:
   ```bash
   sudo ip netns exec hostA ip route del 10.0.2.0/24 via 10.0.1.1
   ```

3. Attempt to ping again:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```
   
   Expected: `Network is unreachable`.

4. Re-add the route and verify connectivity is restored:
   ```bash
   sudo ip netns exec hostA ip route add 10.0.2.0/24 via 10.0.1.1
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```

**Learning**: Routing table entries directly determine reachability.

### Task 6.2: Interface State and Communication

1. Bring down the interface on the router facing `hostB`:
   ```bash
   sudo ip netns exec router ip link set vethBR down
   ```

2. Attempt to ping from `hostA` to `hostB`:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```
   
   Expected: No response or timeout.

3. Bring the interface back up:
   ```bash
   sudo ip netns exec router ip link set vethBR up
   ```

4. Verify connectivity is restored:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```

**Learning**: Interface state (up/down) affects packet forwarding.

### Task 6.3: Packet Capture and Observation

1. In one terminal, start capturing packets on the router's interface facing `hostB`:
   ```bash
   sudo ip netns exec router tcpdump -i vethBR -n
   ```

2. In another terminal, ping from `hostA` to `hostB`:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```

3. Observe in the `tcpdump` output:
   - ICMP Echo Request from `hostA` arriving on `vethBR`
   - ICMP Echo Reply from `hostB` leaving via `vethBR`

**Learning**: Packet forwarding can be observed and traced.

### Task 6.4: IP Forwarding Disabled

1. Disable IP forwarding on the router:
   ```bash
   sudo ip netns exec router sysctl -w net.ipv4.ip_forward=0
   ```

2. Attempt to ping from `hostA` to `hostB`:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.2.2
   ```
   
   Expected: No response or timeout (router drops forwarded packets).

3. Note the difference from local connectivity:
   ```bash
   sudo ip netns exec hostA ping -c 3 10.0.1.1
   ```
   
   This should still work (direct connection, no forwarding needed).

4. Re-enable forwarding:
   ```bash
   sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1
   ```

**Learning**: IP forwarding is the mechanism enabling routers to relay packets between subnets.

---

## Cleanup

To remove all created namespaces and links (optional):

```bash
sudo ip netns del hostA
sudo ip netns del hostB
sudo ip netns del router
```

This automatically removes associated veth pairs and interfaces.

---

## Summary of Commands Reference

### Namespace Management
```bash
sudo ip netns add <name>              # Create namespace
sudo ip netns del <name>              # Delete namespace
sudo ip netns list                    # List all namespaces
sudo ip netns exec <ns> <command>     # Execute command in namespace
```

### Link (Interface) Management
```bash
ip link                               # List interfaces
sudo ip link add <name> type veth peer name <peer>
                                      # Create veth pair
sudo ip link set <iface> netns <ns>   # Move interface to namespace
sudo ip link set <iface> up|down      # Bring interface up/down
```

### Address Management
```bash
ip addr                               # List IP addresses
sudo ip addr add <ip>/<mask> dev <iface>
                                      # Assign IP
sudo ip addr del <ip>/<mask> dev <iface>
                                      # Remove IP
```

### Routing
```bash
ip route                              # Show routing table
sudo ip route add <net>/<mask> via <gateway>
                                      # Add static route
sudo ip route del <net>/<mask>        # Remove route
sudo sysctl net.ipv4.ip_forward       # Check forwarding status
sudo sysctl -w net.ipv4.ip_forward=1  # Enable forwarding
```

### Diagnostics
```bash
ping -c <count> <ip>                  # Test connectivity
sudo tcpdump -i <iface>               # Capture packets
ip route get <ip>                     # Show route to destination
```

---

## Conclusion

This lab plan progressively builds networking skills:
- **Step 1-2**: Understand existing Linux networking and `ip` tools.
- **Step 3**: Isolate multiple network stacks using namespaces.
- **Step 4**: Create direct point-to-point links with veth.
- **Step 5**: Build multi-node routed topologies and practice static routing.
- **Step 6**: Observe behavior through troubleshooting exercises.

Repeat these steps with different topologies (e.g., adding more routers, bridges, or multiple subnets) to deepen understanding of Linux networking concepts.