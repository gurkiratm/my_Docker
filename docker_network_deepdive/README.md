# Docker Networking Deep Dive  
### Host ↔ Container Packet Flow, docker0, veth Pairs, Bridges, NAT, DNAT, SNAT  
*(Based on provided source document.)*

---

## Table of Contents
- 1. Packet Flow Between Host and Docker Container
- 2. Why docker0 Has an IP
- 3. How Docker Selects Bridge for host:PORT
- 4. Port Collision Behavior
- 5. Full Host → Container Packet Path
- 6. Why Docker Uses veth Pairs
- 7. What docker0 Really Is (L2 or L3)
- 8. Summary

---

## 1. Packet Flow Between Host and Docker Container

```
                               +---------------------------+
                               |         HOST OS           |
                               |   (Network Namespace)     |
                               +---------------------------+
                                          |
                                          | 1. Packet created on Host
                                          v
                               +---------------------------+
                               |       Routing Table       |
                               +---------------------------+
                                          |
                                          | 2. Route: 172.17.0.0/16 via docker0
                                          v
                               +---------------------------+
                               |        docker0 bridge     |
                               |   IP: 172.17.0.1/16       |
                               |   (Virtual L2 switch)     |
                               +---------------------------+
                               |                |
                               |3. MAC fwd      |3a. MAC fwd
                               v                v
                        +-----------------+   +-----------------+
                        |   veth0-host    |   |   veth1-host    |
                        +-----------------+   +-----------------+
                              ||   4. veth link pair     ||
                              ||                         ||
                     +-----------------+        +-----------------+
                     | veth0-container |        | veth1-container |
                     +-----------------+        +-----------------+
                               |                          |
                               v                          v
                +------------------------+     +------------------------+
                |  Container A eth0      |     |   Container B eth0     |
                |      172.17.0.2        |     |      172.17.0.3        |
                +------------------------+     +------------------------+
```

---

## 2. Why docker0 Has an IP

docker0 serves as:
- Default gateway for containers  
- NAT/MASQUERADE origin  
- Host ↔ container route  
- L3 interface  

---

## 3. How Docker Selects Bridge for host:PORT

Example:
```
docker run -p 5001:5001 my-app
```

DNAT rule:
```
PREROUTING -p tcp --dport 5001 -j DNAT --to-destination 172.17.0.2:5001
```

Routing selects the correct bridge based on rewritten IP.

---

## 4. Port Collision Behavior

```
docker run -p 5001:5001 A
docker run -p 5001:5001 B
```

Error:
```
port is already allocated
```

---

## 5. Full Host → Container Packet Path

```
Client → Host:5001
          |
          | DNAT
          v
172.18.0.2:5001
          |
       bridge-A
          |
       veth-host
          |
       veth-container
          |
       eth0 (container)
```

---

## 6. Why Docker Uses veth Pairs

```
+---------------------+           +------------------------------+
|   Host Namespace    |           |     Container Namespace      |
+---------------------+           +------------------------------+
            |                                    |
     veth-host-end ---------------------- veth-container-end
            |              (virtual cable)      |
            |
        docker0 bridge
```

Reasons:
- Interfaces cannot span namespaces  
- docker0 cannot live inside container  
- Containers need unique NIC/MAC/IP  
- Isolated routing and firewall tables  

---

## 7. What docker0 Really Is (L2 or L3)

docker0 acts as:
- L2 switch  
- L3 gateway  
- NAT endpoint  

Final:
```
docker0 = L2 switch with L3 IP for gateway functionality.
```

---

## 8. Summary

- docker0 = virtual switch + gateway  
- veth pairs mandatory for namespace bridging  
- DNAT decides container  
- SNAT handles outbound  
- One host port → one container  
- docker0 cannot be placed inside container namespace  

