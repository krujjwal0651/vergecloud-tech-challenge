# Secure Connectivity Between Provider-B Bare Metal and Provider-A Private Services

## Case Scenario

### Two Private Networks:

1\. **Provider A** (Subnet: `172.28.0.0/24`, Gateway: `172.28.0.1`)
- Redis (`172.28.0.20:6379`)
- Kind Cluster 
  - Control Plane and Worker Nodes: `172.28.0.10`, `172.28.0.11`, `172.28.0.12`
  - `Demo-Nginx-App` is reachable through NodePort `30080`.
2\. **Provider B** (Subnet: `172.29.0.0/24`, Gateway: `172.29.0.1`)
- Bare-Metal-WorkLoad (`172.29.0.10`)
3\. **WireGuard-Gateway**
- Primary interface with Provider B network (`172.29.0.2`) and secondary interface with Provider A network (`172.28.0.2`).
- Tunnel IPs: 
  - Gateway tunnel interface: `10.250.0.1/24`
  - Bare-metal tunnel interface: `10.250.0.2/24`
  - Gateway WireGuard endpoint: `172.29.0.2:51820/UDP`.

## Infra Setup

Replicating the above infrastructure using Docker.

### Create Private VPC Networks

```sh
# Provider A
docker network create --driver bridge --internal --subnet 172.28.0.0/24 provider-a-vpc

# Provider B
docker network create --driver bridge --subnet 172.29.0.0/24 provider-b-vpc

# Verify
docker network ls
docker network inspect provider-a-vpc
docker network inspect provider-b-vpc
```

### Create Kind Kubernetes Cluster

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: provider-a

networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"

nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```sh
kind create cluster --config kind-config.yaml
kubectl cluster-info --context kind-provider-a
kubectl get nodes -o wide
docker ps
```

### Attach Kind Nodes to VPC A

```sh
docker network connect --ip 172.28.0.10 provider-a-vpc <control-plane-node>
docker network connect --ip 172.28.0.11 provider-a-vpc <worker-node-1>
docker network connect --ip 172.28.0.12 provider-a-vpc <worker-node-2>

docker network inspect provider-a-vpc
docker exec provider-a-control-plane ip -4 -br addr
```

### Deploy `demo-nginx-app` on Kind K8s

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-web
  labels:
    app: demo-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-web
  template:
    metadata:
      labels:
        app: demo-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo-web
spec:
  type: NodePort
  selector:
    app: demo-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

```sh
kubectl apply -f demo-app.yaml
kubectl get pods -o wide
kubectl get service demo-web
```

### Create a Redis Container on Provider A Network

```sh
docker run -d --name provider-a-redis --network provider-a-vpc \
--ip 172.28.0.20 redis:7-alpine

# Verify
docker ps
docker network inspect provider-a-vpc
docker exec provider-a-redis redis-cli PING
```

### Bare-Metal Workload Setup on Provider B Network

```sh
docker run -d --name bare-metal-workload --network provider-b-vpc \
  --ip 172.29.0.10 --cap-add NET_ADMIN alpine:3.20 sleep infinity

# Verify
docker ps
docker exec bare-metal-workload ip -4 -br addr
docker network inspect provider-b-vpc

# Install basic tools
docker exec bare-metal-workload apk add --no-cache curl redis iproute2 busybox-extras
```

### Verify Bare-Metal Workload Cannot Reach Redis or K8s Cluster

```sh
# Redis:
docker exec bare-metal-workload nc -zvw 3 172.28.0.20 6379

# K8s:
docker exec bare-metal-workload curl --connect-timeout 3 --max-time 5 http://172.28.0.10:30080
```

## WireGuard Gateway Setup

Start a WireGuard gateway container with Alpine Image using Provider B network. `net.ipv4.ip_forward=1` lets it act as a router and forward traffic from bare-metal workload to Redis and Kind.

```sh
docker run -d --name wireguard-gateway --network provider-b-vpc \
--ip 172.29.0.2 --cap-add NET_ADMIN --sysctl net.ipv4.ip_forward=1 \
alpine:3.20 sleep infinity

# Attach secondary interface to Provider A network
docker network connect --ip 172.28.0.2 provider-a-vpc wireguard-gateway

# Verify interfaces and forwarding
docker network inspect provider-b-vpc
docker network inspect provider-a-vpc
docker exec wireguard-gateway ip addr show
docker exec wireguard-gateway sysctl net.ipv4.ip_forward
```

### Install and Verify WireGuard Tools

```sh
docker exec wireguard-gateway apk add --no-cache wireguard-tools iproute2 iptables
docker exec bare-metal-workload apk add --no-cache wireguard-tools

docker exec wireguard-gateway wg --version
docker exec bare-metal-workload wg --version

# Test kernel support; delete wg-test if creation succeeds
docker exec wireguard-gateway ip link add wg-test type wireguard
docker exec wireguard-gateway ip addr show
docker exec wireguard-gateway ip link delete wg-test
```

### Generate WireGuard Key Pairs

#### WireGuard Gateway

```sh
docker exec wireguard-gateway mkdir -p /etc/wireguard
# Private Key
docker exec wireguard-gateway sh -c 'umask 077; wg genkey > /etc/wireguard/privatekey'
# Public Key
docker exec wireguard-gateway sh -c 'wg pubkey < /etc/wireguard/privatekey > /etc/wireguard/publickey'
# Verify permissions and public key
docker exec wireguard-gateway ls -l /etc/wireguard
docker exec wireguard-gateway cat /etc/wireguard/publickey
```

#### Bare-Metal Workload

```sh
docker exec bare-metal-workload mkdir -p /etc/wireguard
# Private Key
docker exec bare-metal-workload sh -c 'umask 077; wg genkey > /etc/wireguard/privatekey'
# Public Key
docker exec bare-metal-workload sh -c 'wg pubkey < /etc/wireguard/privatekey > /etc/wireguard/publickey'
# Verify permissions and public key
docker exec bare-metal-workload ls -l /etc/wireguard
docker exec bare-metal-workload cat /etc/wireguard/publickey
```

### Configure WireGuard on the Gateway

```sh
docker exec -it wireguard-gateway sh
vi /etc/wireguard/wg0.conf
```

`wg0.conf`:

```ini
[Interface]
Address = 10.250.0.1/24
ListenPort = 51820
PrivateKey = <GATEWAY_PRIVATE_KEY>

[Peer]
PublicKey = <BareMetal_PUBLIC_KEY>
AllowedIPs = 10.250.0.2/32
```

```sh
chmod 600 /etc/wireguard/wg0.conf
docker exec wireguard-gateway wg-quick up wg0
docker exec wireguard-gateway ip -4 addr show wg0
```

### Configure WireGuard on the Bare-Metal Container

```sh
# Enter the bare-metal container
docker exec -it bare-metal-workload sh
vi /etc/wireguard/wg0.conf
```

`wg0.conf`:

```ini
[Interface]
Address = 10.250.0.2/24
PrivateKey = <BARE_METAL_PRIVATE_KEY>

[Peer]
PublicKey = <GATEWAY_PUBLIC_KEY>
Endpoint = 172.29.0.2:51820
AllowedIPs = 10.250.0.1/32, 172.28.0.0/24
PersistentKeepalive = 25
```

```sh
chmod 600 /etc/wireguard/wg0.conf
docker exec bare-metal-workload wg-quick up wg0
docker exec bare-metal-workload ip -4 addr show wg0
docker exec bare-metal-workload ip route
docker exec bare-metal-workload ping -c 3 10.250.0.1

# Inspect WireGuard interfaces
docker exec bare-metal-workload wg show
docker exec wireguard-gateway wg show
```

## Add Gateway Forwarding and NAT Rules in WireGuard IPTABLES

Add rules allowing WireGuard traffic into the Provider A network:

```sh
# Redis
docker exec wireguard-gateway iptables -A FORWARD -i wg0 -o eth1 -p tcp -d 172.28.0.20 --dport 6379 -j ACCEPT

# K8s-NodePort
docker exec wireguard-gateway iptables -A FORWARD -i wg0 -o eth1 -p tcp -d 172.28.0.10 --dport 30080 -j ACCEPT
```

Allow established Provider A network responses back through WireGuard:

```sh
docker exec wireguard-gateway \
  iptables -A FORWARD \
  -i eth1 \
  -o wg0 \
  -s 172.28.0.0/24 \
  -m conntrack \
  --ctstate ESTABLISHED,RELATED \
  -j ACCEPT
```

NAT WireGuard clients when entering Provider A network:

```sh
docker exec wireguard-gateway \
  iptables -t nat -A POSTROUTING \
  -s 10.250.0.0/24 \
  -d 172.28.0.0/24 \
  -o eth1 \
  -j MASQUERADE
```

Verify the rules:

```sh
docker exec wireguard-gateway iptables -L FORWARD -n -v
docker exec wireguard-gateway iptables -t nat -L POSTROUTING -n -v
```

## Verify Provider A Private Services Are Reachable

```sh
# Redis
docker exec bare-metal-workload redis-cli -h 172.28.0.20 -p 6379 PING

# Kubernetes NodePort service
docker exec bare-metal-workload curl -fsS --max-time 5 http://172.28.0.10:30080
```

Stop the WireGuard tunnel and verify both private endpoints become unreachable again:

```sh
docker exec bare-metal-workload wg-quick down wg0
docker exec bare-metal-workload redis-cli -h 172.28.0.20 -p 6379 PING
docker exec bare-metal-workload curl -fsS --max-time 5 http://172.28.0.10:30080
```

