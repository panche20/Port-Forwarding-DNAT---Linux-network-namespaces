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

<img width="537" height="87" alt="image" src="https://github.com/user-attachments/assets/1d6c8b65-1534-41da-a6f2-104eb97befcb" />


Step 2 - veth Pair (Virtual Ethernet). A veth pair acts like a virtual cable:
`vethA <------> vethB`
One end is placed inside a namespace, The other stays on the host. Used to connect:
- Container ↔ Host
- Container ↔ Bridge
# Create veth pair
sudo ip link add veth-ns1 type veth peer name veth-host

<img width="1217" height="235" alt="image" src="https://github.com/user-attachments/assets/49879995-b2e8-45f5-8c23-dbe7b413e427" />

# Move one end to namespace
sudo ip link set veth-ns1 netns ns1
```

Step 3 — Assign IPs
```
# Namespace side
sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth-ns1
<img width="945" height="166" alt="image" src="https://github.com/user-attachments/assets/05bec643-c947-4fdc-87e7-2753345e35cf" />

# Host side
sudo ip addr add 10.0.0.1/24 dev veth-host
```

Step 4 — Bring Interfaces UP
```
sudo ip netns exec ns1 ip link set veth-ns1 up
sudo ip netns exec ns1 ip link set lo up
sudo ip link set veth-host up
```
<img width="1277" height="337" alt="image" src="https://github.com/user-attachments/assets/0ad75e82-70fa-4f1c-be34-d77b94ec3ed4" />

Step 5 — Add Default Route
`sudo ip netns exec ns1 ip route add default via 10.0.0.1`
<img width="852" height="107" alt="image" src="https://github.com/user-attachments/assets/e73f6bc9-7f27-4bff-a708-cc8bc74a2fc2" />

Step 6 — Enable Forwarding + NAT
```
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
<img width="850" height="407" alt="image" src="https://github.com/user-attachments/assets/90af397d-35e8-412f-be88-ff4979ca8d25" />

(Replace eth0 with ens5 if needed)

Step 7 — Run Web Server Inside Namespace
We’ll use Python HTTP server:
`sudo ip netns exec ns1 python3 -m http.server 80`
Now ns1 is running a web server on port 80
<img width="842" height="70" alt="image" src="https://github.com/user-attachments/assets/b4589e20-bd69-4ebf-b2a3-7bdeffa50139" />

Step 8 — Try Access (Will Fail)
From host:
`curl 10.0.0.2`
<img width="936" height="458" alt="image" src="https://github.com/user-attachments/assets/869ba2f3-8722-48ad-be7b-7468190fd793" />

This should work but `curl localhost:8080` This will FAIL ❌ (expected)

Step 9 — Add DNAT Rule (Core of Lab)
`sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.2:80`
<img width="1237" height="307" alt="image" src="https://github.com/user-attachments/assets/a578e1ab-685e-47c7-b021-aa7bea5ff8dd" />

Step 10 — Allow Forwarding (VERY IMPORTANT)
`sudo iptables -A FORWARD -p tcp -d 10.0.0.2 --dport 80 -j ACCEPT`

Step 11 — Test
`curl http://localhost:8080` - You should see directory listing
<img width="666" height="303" alt="image" src="https://github.com/user-attachments/assets/ad134ad8-807a-4120-a1f0-6625422cdf29" />

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
















