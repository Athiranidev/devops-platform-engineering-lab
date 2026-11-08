## 1. Learning Objectives

- Understand IP addresses and network interfaces.
- Identify IPv4 and IPv6 addresses.
- Understand private and public IP addresses.
- Understand subnet prefixes and default gateways.
- Read Linux routing tables.
- Understand DNS and common DNS records.
- Test network connectivity using ping.
- Distinguish DNS resolution from network connectivity.
- Perform basic IPv4 and IPv6 troubleshooting.

---

## 2. Network Interfaces and IP Addresses

An IP address is a logical address assigned to a network interface. It helps devices identify where network traffic should be delivered.

A network interface is the connection through which a device communicates with a network.

### Inspect network interfaces

```bash
ip addr
```

This command displays:
- Network interfaces.
- Assigned IPv4 and IPv6 addresses.
- Interface state.
- MAC addresses.
- Network prefixes.

### Interfaces observed on our Ubuntu machine

| Interface | Purpose |
|---|---|
| `lo` | Loopback interface |
| `wlo1` | Wi-Fi network interface |
| `docker0` | Docker default bridge |
| `br-*` | Docker bridge networks |
| `veth*` | Virtual Ethernet interfaces used by containers |

### Important addresses

- `127.0.0.1` — IPv4 loopback address.
- `::1` — IPv6 loopback address.
- `192.168.1.8/24` — Example of our laptop's private IPv4 configuration.
- `172.x.x.x` — Docker bridge network addresses in our lab.

### Understanding interface states

- `UP` — Interface is administratively enabled.
- `LOWER_UP` — Its underlying link is operational.
- `DOWN` — Interface or link is not operational.

The `lo` interface is used for communication within the same machine.

---

## 3. IPv4 and IPv6

There are two major IP versions.

| Feature | IPv4 | IPv6 |
|---|---|---|
| Address size | 32 bits | 128 bits |
| Format | Dotted decimal | Hexadecimal separated by colons |
| Example | `192.168.1.8` | `2001:db8::1` |

IPv4 has a smaller address space than IPv6.

IPv6 was developed to provide a much larger address space and additional networking capabilities.

### IPv4 private address ranges

| Range | Private address block |
|---|---|
| Class A | `10.0.0.0/8` |
| Class B | `172.16.0.0/12` |
| Class C | `192.168.0.0/16` |

These addresses are commonly used in home networks, company networks and cloud private networks.

### Private IP vs Public IP

**Private IP:**
- Used inside private networks.
- Not directly routable across the public internet.
- Commonly used for internal communication.

**Public IP:**
- Globally routable address.
- Used for communication across the internet when routing and security rules permit it.

### DevOps and AWS connection

An EC2 instance can have:
- A private IP address for internal VPC communication.
- A public IP address for internet communication, when configured.

Instances in the same VPC normally communicate using private IP addresses.

However, routing tables, security groups and network ACLs must permit the communication.

---

## 4. Subnet Prefix and Network Identification

Our laptop had an IPv4 configuration similar to:

```text
192.168.1.8/24
```

The `/24` represents the network prefix length.

For this example:

```text
Network: 192.168.1.0/24
Usable hosts: 192.168.1.1 – 192.168.1.254
Broadcast: 192.168.1.255
```

### Examples

| Address | Network | Routing decision |
|---|---|---|
| `192.168.1.8` | `192.168.1.0/24` | Local network |
| `192.168.1.20` | `192.168.1.0/24` | Local network |
| `192.168.2.20` | `192.168.2.0/24` | Use another matching route, or default route |

The network prefix helps Linux determine whether a destination belongs to a directly connected network.

**Important:** A router's IP address does not always end in `.1`. The actual gateway is determined by network configuration.

---

## 5. Routing and Default Gateway

A routing table tells Linux where to send network traffic.

### Inspect IPv4 routes

```bash
ip route
```

Our routing table contained entries similar to:

```text
default via 192.168.1.1 dev wlo1
192.168.1.0/24 dev wlo1
```

### Understanding the default route

Example:

```text
default via 192.168.1.1 dev wlo1
```

| Field | Meaning |
|---|---|
| `default` | Default route |
| `via 192.168.1.1` | Default gateway |
| `dev wlo1` | Network interface |
| `proto dhcp` | Route configured through DHCP, when shown |
| `src` | Preferred source IP address, when shown |
| `metric` | Route preference value |

The default gateway is usually the local router.

### How routing works

When communicating with another device in the same subnet, Linux can use the directly connected route.

When communicating with an external network, Linux normally uses the default route if no more specific route matches.

### Important routing principle

Linux uses the most specific matching route available.

This is called **Longest Prefix Match**.

---

## 6. IPv6 Routing

### Inspect IPv6 routes

```bash
ip -6 route
```

Our output contained:

```text
<local-ipv6-prefix>/64 dev wlo1 proto ra
default via <router-link-local-ipv6> dev wlo1 proto ra
```

This helped us identify:

- An IPv6 network prefix assigned to the Wi-Fi interface.
- An IPv6 default route.
- The router acting as the IPv6 default gateway.
- Routes learned through Router Advertisement.

### What is Router Advertisement?

RA stands for Router Advertisement.

An IPv6 router advertises network information to connected devices.

Linux can use this information to configure IPv6 addresses and routing.

### Link-local IPv6 addresses

Addresses beginning with:

```text
fe80::
```

are IPv6 link-local addresses.

They are used for communication within a local network link.

When using a link-local address, Linux may need the interface name to identify which network link to use.

Example:

```bash
ping -6 -c 4 fe80::example%wlo1
```

The `%wlo1` identifies the network interface.

---

## 7. DNS — Domain Name System

DNS translates domain names into IP addresses using the system's name-resolution mechanism.

Example:

```text
google.com → IP address
```

Computers use IP addresses for network communication, while humans find domain names easier to remember.

### Important DNS records

| Record | Purpose |
|---|---|
| A | Maps a hostname to an IPv4 address |
| AAAA | Maps a hostname to an IPv6 address |
| CNAME | Points one hostname to another hostname |
| MX | Specifies mail servers |
| TXT | Stores text information, often used for verification |

### Practical DNS lookup

```bash
getent hosts google.com
```

This uses the system's configured hostname-resolution mechanism to look up the hostname.

Our output returned an IPv6 address.

### Inspect available address information

```bash
getent ahosts google.com
```

Our output included:
- An IPv6 address.
- An IPv4 address.
- Socket types such as `STREAM`, `DGRAM` and `RAW`.

These socket types describe the types of address information returned by the system's `getaddrinfo` mechanism. They are not separate DNS records.

### Important distinction

Successful hostname resolution does not prove that the destination is reachable.

DNS resolution and network connectivity are separate operations.

---

## 8. Testing Network Connectivity with ping

`ping` uses ICMP Echo Request and Echo Reply messages to test IP connectivity.

### Test general connectivity

```bash
ping -c 4 google.com
```

`-c 4` sends four packets and stops.

Our initial test selected IPv6 and failed.

### Force IPv4

```bash
ping -4 -c 4 google.com
```

Observed result:

```text
4 packets transmitted
4 received
0% packet loss
```

IPv4 connectivity to the destination worked during this test.

### Force IPv6

```bash
ping -6 -c 4 google.com
```

Observed result:

```text
4 packets transmitted
0 received
100% packet loss

Destination unreachable: Address unreachable
```

The error was returned by our router.

### Important difference

- `-4` forces IPv4.
- `-6` forces IPv6.
- Without either option, the system selects an address according to its address-selection behaviour.

IPv4 is not inherently faster than IPv6. Both depend on network configuration and routing.

---

## 9. Practical IPv6 Troubleshooting

We discovered an interesting real-world networking issue:

IPv4 connectivity worked, but external IPv6 connectivity failed.

Instead of randomly changing network settings, we investigated each part of the network path.

### Step 1: Inspect network configuration

```bash
ip addr
```

Confirmed:
- Wi-Fi interface was `wlo1`.
- An IPv4 address was assigned.
- IPv6 addresses were assigned.
- Docker interfaces were also present.

### Step 2: Inspect IPv4 routing

```bash
ip route
```

Identified:
- The default IPv4 route.
- The Wi-Fi interface used for external traffic.
- The local IPv4 subnet.

### Step 3: Test DNS resolution

```bash
getent hosts google.com
getent ahosts google.com
```

Both IPv4 and IPv6 address information was returned.

### Step 4: Test IPv4 connectivity

```bash
ping -4 -c 4 google.com
```

Result: Four packets received, zero packet loss.

### Step 5: Inspect IPv6 routing

```bash
ip -6 route
```

Identified:
- An IPv6 network prefix.
- An IPv6 default route.
- A default gateway learned through Router Advertisement.

### Step 6: Test the local IPv6 gateway

```bash
ping -6 -c 4 <router-link-local-address>%wlo1
```

Result:

```text
4 packets transmitted
4 received
0% packet loss
```

This confirmed that the laptop could communicate with the local IPv6 router.

### Step 7: Test external IPv6 connectivity

```bash
ping -6 -c 4 google.com
```

Result: Destination unreachable.

### Step 8: Test another external IPv6 destination

```bash
ping -6 -c 4 2606:4700:4700::1111
```

Result: Destination unreachable again, with the error returned by the router.

### Troubleshooting findings

| Investigation | Result |
|---|---|
| IPv4 configuration | Present |
| IPv4 connectivity | Working |
| DNS resolution | Working |
| IPv6 address configuration | Present |
| IPv6 default route | Present |
| Local IPv6 gateway connectivity | Working |
| External IPv6 connectivity | Failing |

### Current hypothesis

The laptop can reach its IPv6 router, but external IPv6 traffic is reported as unreachable by that router.

Possible causes include:
- Missing or unusable upstream IPv6 connectivity.
- Router IPv6 forwarding or routing configuration.
- ISP IPv6 routing problems.
- Network filtering or other router-side issues.

We have not yet identified the exact root cause.

**Important:** A successful gateway ping does not prove that the router can forward traffic to the internet.

---

## 10. Commands Revision

| Command | Purpose |
|---|---|
| `ip addr` | Inspect network interfaces and assigned addresses |
| `ip route` | Inspect IPv4 routing table |
| `ip -6 route` | Inspect IPv6 routing table |
| `getent hosts DOMAIN` | Resolve a hostname using system name resolution |
| `getent ahosts DOMAIN` | Retrieve address information for a hostname |
| `ping -c 4 DOMAIN` | Test connectivity using ICMP |
| `ping -4 -c 4 DOMAIN` | Test IPv4 connectivity |
| `ping -6 -c 4 DOMAIN` | Test IPv6 connectivity |
| `ss -ltnp` | Inspect listening TCP sockets |

---

## 11. Practical Work Completed vs Pending

### Completed

- Inspected actual network interfaces on Ubuntu.
- Identified Wi-Fi, loopback and Docker interfaces.
- Identified local IPv4 network configuration.
- Inspected the IPv4 routing table.
- Identified the default gateway.
- Understood subnet prefixes and local network routing.
- Studied private and public IP addresses.
- Understood DNS fundamentals and common record types.
- Performed hostname resolution using `getent`.
- Retrieved IPv4 and IPv6 address information.
- Successfully tested IPv4 connectivity using ping.
- Inspected the IPv6 routing table.
- Successfully tested local IPv6 gateway connectivity.
- Investigated failed external IPv6 connectivity using two destinations.

### Pending

- Test HTTPS connectivity over IPv6 using curl.
- Investigate the IPv6 connectivity failure further if useful.
- Study TCP vs UDP.
- Study common ports.
- Understand HTTP vs HTTPS.
- Study DNS troubleshooting in more depth.
- Learn basic network connectivity troubleshooting scenarios.

### Next practical command

```bash
curl -6 -I --connect-timeout 5 https://google.com
```

This tests an HTTPS connection using IPv6 and requests response headers only.

---

## 12. Interview Revision

### Q1. What is an IP address?

An IP address is a logical address assigned to a network interface to identify a device or interface on a network.

### Q2. What is a default gateway?

A default gateway is the router used to forward traffic when no more specific matching route exists.

### Q3. What is DNS?

DNS translates domain names into IP addresses using a name-resolution system.

### Q4. What is the difference between IPv4 and IPv6?

IPv4 uses 32-bit addresses, whereas IPv6 uses 128-bit addresses.

### Q5. What happens when DNS resolution succeeds but ping fails?

The hostname has been resolved, but the connectivity test has failed. Possible causes include routing problems, firewall rules, network failures or ICMP filtering.

### Q6. Can two EC2 instances in the same VPC communicate using private IPs?

Yes, provided routing and security rules permit the communication.

### Q7. Does having an IPv6 address mean internet connectivity is available?

No. A device can have an IPv6 address and local route but still lack working external IPv6 connectivity.

### Q8. How would you investigate a connectivity problem?

I would inspect network interfaces, IP configuration, routing, DNS resolution and connectivity. I would isolate the failure by testing the local gateway and external destinations, then investigate the relevant network component.

---


