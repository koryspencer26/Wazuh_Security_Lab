# Phase 2: Transport Security & SIEM Manager Hardening

## Objective
Configure secure communication pipelines between distributed Windows 11 endpoints and the centralized Ubuntu core manager, ensuring the infrastructure host securely accepts agent enrollment and telemetry streams while denying unauthorized traffic.

---

## 1. Technical Obstacle: Agent-Auth Dropout (Error 1208)
During the initial deployment of the first Windows 11 endpoint, the agent failed to connect to the manager, throwing a critical communication fault: `Error 1208 - Unable to connect to password/enrollment service`. 

### Root Cause Analysis (RCA)
1. **Network Ingress Disconnect (Firewall Dropping Packets):** The Ubuntu infrastructure host was running a default "deny all incoming" firewall policy. The transport packets sent by the Windows agent were being dropped at the host boundary.
2. **Service Daemon Desynchronization:** Due to previous Phase 1 network restructuring, the internal Wazuh enrollment daemon (`authd`) was not actively binding to or listening on its designated network sockets.

---

## 2. Resolution & Remediation Steps

### Step 1: Firewall Optimization (UFW)
To allow secure telemetry ingestion without exposing unnecessary host services, the Uncomplicated Firewall (UFW) was configured to open specific "valves" dedicated to SIEM traffic.

```bash
# Explicitly permit agent telemetry ingestion (Layer 4 UDP)
sudo ufw allow <PORT>/udp

# Explicitly permit secure agent enrollment/handshaking (Layer 4 TCP)
sudo ufw allow <PORT>/tcp

# Reload the firewall engine to apply changes
sudo ufw reload

# Validate the active firewall policy profile
sudo ufw status verbose
```

### Step 2. Sockets & Listener Auditing
To verify that the operating system was correctly hosting the enrollment service after the firewall adjustment, the network utility layer was queried to inspect active socket listeners.

```
# Check if the manager is actively listening on both ports, tcp|udp
sudo ss -tlpn | grep -E '<PORT>|<PORT>'
```

### Step 3: Initialization & Handshake Validation
To force the manager to spin up its listeners on the newly stabilized static IP (<SERVER_IP>), a full stack restart command was sent to the management engine.

```
# Restart the core SIEM manager system service
sudo systemctl restart wazuh-manager
```
Verification: Re-running ```sudo ss -tlpn | grep <tcp PORT>``` confirmed the process was successfully binding to the socket and waiting for a connection.

---

## 3. Endpoint Onboarding & Scaling

### Step 1. Manual Authentication Handshake (Windows 11)
With the server side fully primed and listening, an administrative shell was initialized on the Windows endpoints to execute the authentication binary, forcing a secure key exchange with the manager.

```
# Executed as Administrator in Windows PowerShell
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m <SERVER_IP>
```
Result: ```INFO: Valid key received.```

### Step 2: Multi-Agent Deployment & Fleet Validation
To scale the environment out of a simple "proof of concept" into a multi-node architecture, a second personal Windows device was onboarded utilizing the same enrollment pipeline. To verify both agents were streaming live telemetry side-by-side, the management core database was queried via the CLI:
```
# List all registered and active agents on the server
sudo /var/ossec/bin/agent_control -l
```
### Final Metrics & Status Verification
* Agent 001 (Primary Device): Status: Active
* Agent 002 (Secondary Device): Status: Active
* Telemetry Verification: Confirmed end-to-end telemetry generation by filtering log events inside the web interface for standard system privilege discovery execution commands (```whoami```).

---

## 4. Key Takeaways for the Enterprise
* Socket vs. Firewall Verification: Network troubleshooting must cross-examine both network boundaries (Firewalls) and internal OS realities (Socket listeners using ```ss```). A port can be open on a firewall, but if a service isn't listening, the packet drops.

* Scalable Fleet Visibility: Isolating endpoint events by unique cryptographic IDs (```001, 002```) allows a SOC analyst to track lateral movement across distinct segments of a network.











