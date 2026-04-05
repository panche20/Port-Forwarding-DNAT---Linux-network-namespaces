Linux Network Namespaces & Container Networking (DevOps Project)
📌 Overview

This project demonstrates how container networking actually works under the hood by building everything manually using Linux primitives.

Instead of relying on Docker abstractions, we recreate:

Container isolation (network namespaces)
Container connectivity (veth pairs + bridge)
Internet access (SNAT / MASQUERADE)
Port forwarding (DNAT)

👉 This is the exact foundation used by Docker and Kubernetes.

## What You’ll Build
`Browser → EC2:8080 → iptables (DNAT) → ns1:80 (web server)`
- External traffic hits your host
- Gets forwarded to namespace
- Namespace reponds back

Step 1 — Network Namespace. A network namespace provides an isolated network stack.
Each namespace has: Its own interfaces (eth0, lo), Its own IP addresses, Its own routing table.
In Docker, each container runs inside its own network namespace
```
# Create namespace
sudo ip netns add ns1

Step 2 - veth Pair (Virtual Ethernet). A veth pair acts like a virtual cable:
`vethA <------> vethB`
One end is placed inside a namespace, The other stays on the host. Used to connect:
- Container ↔ Host
- Container ↔ Bridge
# Create veth pair
sudo ip link add veth-ns1 type veth peer name veth-host

# Move one end to namespace
sudo ip link set veth-ns1 netns ns1
```

Step 3 — Assign IPs
```
# Namespace side
sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth-ns1

# Host side
sudo ip addr add 10.0.0.1/24 dev veth-host
```

Step 4 — Bring Interfaces UP
```
sudo ip netns exec ns1 ip link set veth-ns1 up
sudo ip netns exec ns1 ip link set lo up
sudo ip link set veth-host up
```

Step 5 — Add Default Route
`sudo ip netns exec ns1 ip route add default via 10.0.0.1`

Step 6 — Enable Forwarding + NAT
```
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
(Replace eth0 with ens5 if needed)

Step 7 — Run Web Server Inside Namespace
We’ll use Python HTTP server:
`sudo ip netns exec ns1 python3 -m http.server 80`
Now ns1 is running a web server on port 80

Step 8 — Try Access (Will Fail)
From host:
`curl 10.0.0.2`
This should work but `curl localhost:8080` This will FAIL ❌ (expected)

Step 9 — Add DNAT Rule (Core of Lab)
`sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.2:80`

Step 10 — Allow Forwarding (VERY IMPORTANT)
`sudo iptables -A FORWARD -p tcp -d 10.0.0.2 --dport 80 -j ACCEPT`

Step 11 — Test
`curl http://localhost:8080` - You should see directory listing

Bonus — Access from Browser
`http://<your-ec2-public-ip>:8080`

What Just Happened (CRITICAL)
Incoming request:
`Client → EC2:8080`
iptables DNAT:
`8080 → 10.0.0.2:80`
Packet flow:
`Internet → Host → NAT → Namespace → Response → Host → Internet`

## Real Docker Mapping
When you run:
`docker run -p 8080:80 nginx`

What Docker does: 
- creates namespace
- creates veth pair
- adds NAT rules (DNAT + MASQUERADE)
















