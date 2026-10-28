# Desing Notes
## Problem
- Provider-B bare-metal workloads need secure access to private Provider-A services over the public internet. 
- No VPC peering, private interconnect, or transit network is available.
- The required private targets of Provider-A  are:
  * Redis, representing a managed SaaS service.
  * Selected services running in Kubernetes.
---
## Traffic Flow Design
- Use WireGuard as a hub-and-spoke Layer-3 VPN.
  ```
  Provider-B bare-metal --> Encrypted WireGuard UDP traffic --> Provider-A WireGuard gateway --> Private VPC routing --> Redis and selected Kubernetes services
  ```

- Redis and selected Kubernetes services
  - Each bare-metal host is a WireGuard peer with:
    * a unique WireGuard key pair.
    * a unique tunnel IP.
    * routes only for approved Provider-A private ranges.
  - The Provider-A gateway has:
    * one public-facing interface.
    * one private VPC interface.
    * IP forwarding enabled.
    * firewall/NAT rules controlling traffic between WireGuard and the private VPC.

- Mock environment mapping to Production environment:
  ```
  Mock Environment                    Production Environment
  ----------------                    ----------------------
  
  Internal Docker network:             Provider-A private VPC
  provider-a-vpc
  
  bare-metal-workload                  Provider-B bare-metal host
  Docker container
  
  Docker network                       Public internet
  
  wireguard-gateway                    Provider-A VPN gateway
  Docker container
  
  Redis container                      Private SaaS service
  
  kind cluster                         Kubernetes cluster
  
  NGINX application 
  exposed through NodePort             Selected Kubernetes service
  ```

- **Traffic flow:**
  - Redis:
    ```
    bare-metal workload --> WireGuard tunnel --> WireGuard gateway --> Provider-A VPC --> Redis at 172.28.0.20:6379
    ```
  
  - Kubernetes application:
    ```
    bare-metal workload --> WireGuard tunnel --> WireGuard gateway --> kind control-plane VPC address: 172.28.0.10:30080 --> Kubernetes NodePort --> demo-web Pods
    ```
---
### Why WireGuard Is the Right Choice :
- WireGuard was selected because it is small, fast, encrypted, key-based, and operationally simpler than a larger VPN appliance stack for approximately thirty hosts.
- Hub-and-spoke topology:
  - Each bare-metal host connects to Provider-A gateway infrastructure rather than connecting to every other host.
- Benefits:
  * simpler key and route management.
  * central policy enforcement.
  * minimal change to Kubernetes and SaaS services.
  * no direct Provider-B to Provider-A-VPC connectivity.
  * Only Provider-A private CIDRs are routed through WireGuard:
    ```
    172.28.0.0/24 (Provider-A-VPC) --> WireGuard
    all other traffic (Provider-B) --> normal default route
    ```
    Only traffic for Provider-A private services goes through WireGuard. Normal internet traffic from Provider-B servers continues to use its usual internet connection.

---