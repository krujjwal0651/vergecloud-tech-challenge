# Production Plan
### 1\. Deploy two WireGuard gateways
- Run two hardened gateway instances in Provider A.
  * removes a single-gateway host failure.
  * supports failover.
  * allows controlled maintenance.
* the challenge requires a single-zone Kubernetes cluster, so a full zone-outage solution is outside the scope of this design.

### 2. Reserve non-overlapping CIDRs
- non-overlapping CIDRs for :
  * Provider-A VPC networks.
  * Provider-B host networks.
  * WireGuard tunnel addresses.
  * Kubernetes Pod and Service networks.
* overlapping CIDRs cause incorrect routing and difficult outages.

### 3. Use default-deny firewall policy
- Allow only approved traffic.
- for example:
  ```	
  bare-metal peer --> Redis: TCP/6379
  
  bare-metal peer --> private Kubernetes endpoint: TCP/443 or approved application port
  ```

- a VPN establishes private connectivity but should not grant unrestricted network access.

### 4. Implement WireGuard peer identity and key lifecycle
- Provide each bare-metal server with:
  * a unique WireGuard key pair.
  * a unique tunnel address.
  * an ownership record;
  * an approval/access policy;
  * automated revocation on decommissioning.
* shared keys prevent reliable revocation and accountability.

### 5. Store secrets securely
- Store private keys in the secret-management system like HashiCorp Valut, AWS Secret Manager.
- We should avoid:
  * commit private keys to Git.
  * paste private keys into tickets or chat.
  * reuse a private key across multiple servers.

### 6. Add observability and alerting
- Monitor:
  * handshake age.
  * tunnel availability.
  * packets sent/received.
  * gateway CPU, memory, packet drops, and bandwidth.
  * denied firewall traffic.
  * private-service connection failures.

- Alert when:
  * a peer loses handshake unexpectedly.
  * gateway capacity is near limit.
  * suspicious connection attempts occur.
  * a previously revoked peer reconnects.

### 7\. Manage infrastructure as code
- Use infrastructure-as-code (like Terraform) system for:
  * gateway instances.
  * public IPs.
  * private network interfaces.
  * firewall rules (security groups).
  * routes.
  * monitoring resources.

### 8. Automate host configuration
- Use Configuration Management tools (like Ansible or SaltStack) for automation :
  * install WireGuard.
  * deploy peer configuration.
  * enable the service at boot.
  * distribute routes.
  * rotate keys.
  * remove revoked peers.

### 9. Automate policy validation
- Add CI validation for:
  * duplicate tunnel IPs.
  * overlapping CIDRs.
  * malformed WireGuard configurations.
  * unapproved destination CIDRs/ports.
  * missing peer ownership or expiry metadata.

### 10. Write operational runbooks
- Document procedures for adding a host, revoking a host, rotating keys, investigating a failed handshake, gateway failover, incident response, rollback.

### 11\. Replace NodePort with private ingress or internal load balancer
- Use an internal load balancer, private ingress, or Gateway API endpoint to provide stable service endpoints, health checks, TLS termination, clear service ownership, and less dependency on Kubernetes node addresses.

### 12. Add private DNS
- Provide stable names such as `redis.private.example`, `api.private.example` to avoids hard-coded IP addresses which supports controlled endpoint migration.

### 13. Add workload-level policy
- Use Kubernetes NetworkPolicy and application authentication/authorization in addition to VPN and network firewalls.
* network access alone should not be the only protection layer.
---