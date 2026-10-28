# Secure Connectivity Between Provider-B Bare Metal and Provider-A Private Services
This repository contains a solution for secure access from a Provider-B bare-metal workload to private Provider-A services over a WireGuard tunnel. 

**The mock includes:**
* A kind Kubernetes cluster with a sample NGINX service.
* A private Redis container.
* An isolated bare-metal workload container.
* A WireGuard gateway that securely connects the bare-metal workload to Provider-A services.

**Documentation:**
* ~[Mock setup](docs/mock-setup.md)~ - Manual steps to reproduce the local environment and validate connectivity.
* ~[Design notes](docs/desing-notes.md)~ - Why WireGuard was chosen, the overall approach, assumptions, and risks.
* ~[Production plan](docs/production-plan.md)~ - Prioritized improvements required beyond the mock.

**Core validation:**
- The bare-metal workload can reach Redis and the selected Kubernetes service only while the WireGuard tunnel is active. 
- When the tunnel is stopped, both private endpoints become unreachable.
