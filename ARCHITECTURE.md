# FortiGate MCP Server Architecture

This document describes the architecture of the FortiGate MCP Server for kagent, including components, data flow, and integration points.

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster (kagent)                         │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                          kagent Namespace                            │  │
│  │                                                                      │  │
│  │  ┌────────────────┐         ┌──────────────────────────────────┐    │  │
│  │  │                │         │  FortiGate Troubleshooting Agent │    │  │
│  │  │  User / AI     │         │                                  │    │  │
│  │  │  Assistant     │────────▶│  - Expert AI Agent               │    │  │
│  │  │                │         │  - Troubleshooting Workflows     │    │  │
│  │  └────────────────┘         │  - 8 MCP Tools                   │    │  │
│  │         │                   └──────────────┬───────────────────┘    │  │
│  │         │                                  │                        │  │
│  │         │                                  │                        │  │
│  │         │ Manual Tool Calls                │ Automated Workflows    │  │
│  │         │                                  │                        │  │
│  │         ▼                                  ▼                        │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │          RemoteMCPServer (fortigate-mcp-remote)             │   │  │
│  │  │                                                              │   │  │
│  │  │  Registered Tools:                                           │   │  │
│  │  │    • list_firewall_policies    • get_system_status          │   │  │
│  │  │    • list_firewall_addresses   • list_interfaces            │   │  │
│  │  │    • list_firewall_services    • list_static_routes         │   │  │
│  │  │    • list_virtual_ips          • discover_vdoms             │   │  │
│  │  └──────────────────────┬──────────────────────────────────────┘   │  │
│  │                         │                                           │  │
│  │                         │ HTTP/JSON-RPC 2.0                         │  │
│  │                         │ http://fortigate-mcp-server:8082/mcp      │  │
│  │                         ▼                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │          Service (fortigate-mcp-server)                     │   │  │
│  │  │          Type: ClusterIP                                    │   │  │
│  │  │          Port: 8082                                         │   │  │
│  │  └──────────────────────┬──────────────────────────────────────┘   │  │
│  │                         │                                           │  │
│  │                         ▼                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │     Deployment (fortigate-mcp-server)                       │   │  │
│  │  │                                                              │   │  │
│  │  │  ┌────────────────────────────────────────────────────┐     │   │  │
│  │  │  │  Pod: fortigate-mcp-server                         │     │   │  │
│  │  │  │                                                     │     │   │  │
│  │  │  │  ┌──────────────────────────────────────────────┐  │     │   │  │
│  │  │  │  │  Container: fortigate-mcp                    │  │     │   │  │
│  │  │  │  │                                              │  │     │   │  │
│  │  │  │  │  Image: sebbycorp/fortigate-mcp-server      │  │     │   │  │
│  │  │  │  │  Port: 8082                                 │  │     │   │  │
│  │  │  │  │                                              │  │     │   │  │
│  │  │  │  │  ┌────────────────────────────────────────┐ │  │     │   │  │
│  │  │  │  │  │  FastMCP Server (main.py)              │ │  │     │   │  │
│  │  │  │  │  │                                        │ │  │     │   │  │
│  │  │  │  │  │  • HTTP Transport (stateless)         │ │  │     │   │  │
│  │  │  │  │  │  • 8 MCP Tool Handlers                │ │  │     │   │  │
│  │  │  │  │  │  • httpx Client for FortiGate API     │ │  │     │   │  │
│  │  │  │  │  └────────────────────────────────────────┘ │  │     │   │  │
│  │  │  │  │                                              │  │     │   │  │
│  │  │  │  │  Environment Variables:                     │  │     │   │  │
│  │  │  │  │    • FORTIGATE_HOST (from secret)           │  │     │   │  │
│  │  │  │  │    • FORTIGATE_TOKEN (from secret)          │  │     │   │  │
│  │  │  │  │    • FORTIGATE_VDOM (root)                  │  │     │   │  │
│  │  │  │  └──────────────────────────────────────────────┘  │     │   │  │
│  │  │  └────────────────────────────────────────────────────┘     │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  │                                                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │         Secret (fortigate-credentials)                      │   │  │
│  │  │                                                              │   │  │
│  │  │  • FORTIGATE_HOST: 172.16.10.1 (base64)                     │   │  │
│  │  │  • FORTIGATE_TOKEN: py58Gw... (base64)                      │   │  │
│  │  │  • FORTIGATE_VDOM: root                                     │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ HTTPS (API Token Auth)
                                   │ FortiGate REST API v2
                                   │ SSL Verification: Disabled
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │    FortiGate Firewall        │
                    │    172.16.10.1               │
                    │                              │
                    │  • Firewall Policies         │
                    │  • Address Objects           │
                    │  • Service Objects           │
                    │  • Network Interfaces        │
                    │  • Static Routes             │
                    │  • Virtual IPs (NAT)         │
                    │  • VDOMs                     │
                    └──────────────────────────────┘
```

## Component Details

### 1. FortiGate Troubleshooting Agent

**Type:** Declarative AI Agent (kagent Agent CRD)

**Purpose:** Expert AI assistant that automates FortiGate troubleshooting workflows

**Capabilities:**
- Traffic troubleshooting (blocked connections, policy analysis)
- NAT/VIP configuration verification
- Routing diagnostics
- System health monitoring
- Security policy recommendations

**Integration:**
- Uses all 8 MCP tools through the RemoteMCPServer
- Follows predefined troubleshooting workflows
- Provides context-aware recommendations

**Model:** xai-grok-config (configurable)

### 2. RemoteMCPServer

**Type:** Kubernetes Custom Resource (kagent RemoteMCPServer CRD)

**Purpose:** Registers and exposes MCP tools to kagent

**Configuration:**
- **URL:** `http://fortigate-mcp-server.kagent.svc.cluster.local:8082/mcp`
- **Protocol:** STREAMABLE_HTTP
- **Timeout:** 30s
- **Registered Tools:** 8 FortiGate management tools

**Function:**
- Discovers available tools from the MCP server
- Makes tools available to agents and users
- Handles tool invocation routing

### 3. Kubernetes Service

**Type:** ClusterIP

**Purpose:** Internal service discovery and load balancing

**Configuration:**
- **Name:** fortigate-mcp-server
- **Namespace:** kagent
- **Port:** 8082
- **Selector:** app=fortigate-mcp-server

**DNS:** `fortigate-mcp-server.kagent.svc.cluster.local`

### 4. Kubernetes Deployment

**Purpose:** Manages the MCP server pod lifecycle

**Configuration:**
- **Replicas:** 1 (can be scaled)
- **Container Image:** sebbycorp/fortigate-mcp-server:latest
- **Resources:**
  - Requests: 128Mi memory, 100m CPU
  - Limits: 512Mi memory, 500m CPU

**Probes:**
- **Liveness:** TCP socket on port 8082
- **Readiness:** TCP socket on port 8082

### 5. MCP Server Container

**Base Image:** Python 3.11-slim

**Application:** FastMCP HTTP Server

**Key Components:**
- **main.py:** Python application implementing 8 MCP tools
- **FastMCP:** Framework for building MCP servers
- **httpx:** HTTP client for FortiGate REST API calls
- **Port:** 8082 (HTTP)

**Authentication:**
- Reads credentials from environment variables
- Uses Bearer token authentication (preferred)
- Falls back to username/password if token not available

### 6. Kubernetes Secret

**Purpose:** Secure storage of FortiGate credentials

**Contents:**
- FORTIGATE_HOST: FortiGate IP address (base64 encoded)
- FORTIGATE_TOKEN: API token (base64 encoded)
- FORTIGATE_USER: Username (base64 encoded, fallback)
- FORTIGATE_PASS: Password (base64 encoded, fallback)

**Security Considerations:**
- Base64 encoded (not encrypted)
- Should be managed by external secret manager in production
- Mounted as environment variables in the pod

### 7. FortiGate Device

**Access Method:** FortiGate REST API v2

**Endpoints Used:**
- `/api/v2/cmdb/firewall/policy` - Firewall policies
- `/api/v2/cmdb/firewall/address` - Address objects
- `/api/v2/cmdb/firewall.service/custom` - Service objects
- `/api/v2/monitor/system/status` - System status
- `/api/v2/cmdb/system/interface` - Network interfaces
- `/api/v2/cmdb/router/static` - Static routes
- `/api/v2/cmdb/firewall/vip` - Virtual IPs
- `/api/v2/cmdb/system/vdom` - Virtual domains

**Authentication:** Bearer Token (API Token)

**VDOM Context:** All requests scoped to configured VDOM (default: root)

## Data Flow

### User Query Flow

1. **User submits query** to AI assistant or directly calls MCP tool
2. **kagent routes request** to appropriate handler:
   - For agent queries: Routes to FortiGate Troubleshooting Agent
   - For direct tool calls: Routes to RemoteMCPServer
3. **Agent analyzes query** and determines which tools to call
4. **Agent invokes tools** through RemoteMCPServer
5. **RemoteMCPServer forwards request** to MCP server HTTP endpoint
6. **Service routes traffic** to MCP server pod
7. **MCP server receives JSON-RPC 2.0 request**
8. **Tool handler executed** in main.py
9. **httpx client makes HTTPS request** to FortiGate REST API
10. **FortiGate processes request** and returns JSON response
11. **MCP server formats response** and returns to caller
12. **Agent analyzes results** and formulates recommendation
13. **Response returned to user**

### Example: Traffic Troubleshooting Flow

```
User: "My traffic is being blocked to 10.0.1.50 on port 443"
  │
  ├─▶ FortiGate Troubleshooting Agent
  │     │
  │     ├─▶ Tool: get_system_status
  │     │     └─▶ GET /api/v2/monitor/system/status
  │     │
  │     ├─▶ Tool: list_firewall_policies
  │     │     └─▶ GET /api/v2/cmdb/firewall/policy
  │     │
  │     ├─▶ Tool: list_firewall_addresses
  │     │     └─▶ GET /api/v2/cmdb/firewall/address
  │     │
  │     ├─▶ Tool: list_firewall_services (service for port 443)
  │     │     └─▶ GET /api/v2/cmdb/firewall.service/custom
  │     │
  │     └─▶ Analyzes results, identifies policy issue
  │
  └─▶ Response: "No policy found allowing traffic from your source to
      10.0.1.50 on HTTPS/443. Recommendation: Create firewall policy..."
```

## Network Communication

### Internal (Kubernetes Cluster)

**Protocol:** HTTP (unencrypted within cluster)
**Port:** 8082
**Path:** /mcp
**Format:** JSON-RPC 2.0

**Example Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "list_firewall_policies",
    "arguments": {}
  },
  "id": 1
}
```

### External (to FortiGate)

**Protocol:** HTTPS (TLS 1.2+)
**Port:** 443 (default)
**Base URL:** https://172.16.10.1
**API Path:** /api/v2/*
**Authentication:** Bearer Token
**SSL Verification:** Disabled (for self-signed certs)

**Example Request:**
```http
GET /api/v2/cmdb/firewall/policy?vdom=root HTTP/1.1
Host: 172.16.10.1
Authorization: Bearer py58GwNstdhmQ79y5qtpm6wgtwtnxy
```

## Security Architecture

### Authentication Chain

```
User/Agent → kagent RBAC → RemoteMCPServer → MCP Server → API Token → FortiGate
```

1. **kagent RBAC:** Controls who can access agents and tools
2. **RemoteMCPServer:** No additional auth (trusts cluster network)
3. **MCP Server:** No authentication (cluster-internal only)
4. **FortiGate API:** Bearer token authentication

### Security Boundaries

1. **Kubernetes Network:** ClusterIP service (not externally exposed)
2. **Namespace Isolation:** kagent namespace with RBAC
3. **Secret Management:** Kubernetes secrets (should use external secret manager)
4. **TLS Termination:** At FortiGate device (self-signed cert accepted)

### Security Best Practices

**Implemented:**
- ✅ API token authentication (preferred over username/password)
- ✅ ClusterIP service (internal only)
- ✅ Kubernetes secrets for credentials
- ✅ Non-root container user
- ✅ Read-only access to FortiGate
- ✅ Resource limits on pod

**Recommended for Production:**
- ⚠️ External secret management (Vault, AWS Secrets Manager, etc.)
- ⚠️ Network policies to restrict egress/ingress
- ⚠️ Valid SSL certificates on FortiGate
- ⚠️ SSL verification enabled
- ⚠️ Mutual TLS between MCP server and FortiGate
- ⚠️ Regular credential rotation
- ⚠️ Audit logging of all API calls

## Scalability Considerations

### Current Architecture

- **Single pod:** 1 replica by default
- **Stateless design:** Can scale horizontally
- **Connection pooling:** httpx client creates new connections per request

### Scaling Options

**Horizontal Pod Scaling:**
```bash
kubectl scale deployment fortigate-mcp-server -n kagent --replicas=3
```

**Considerations:**
- FortiGate API rate limits
- Concurrent connection limits on FortiGate
- Session exhaustion on FortiGate device

**Recommendations:**
- Monitor FortiGate CPU and connection count
- Scale conservatively (3-5 replicas max)
- Implement connection pooling if needed
- Consider FortiGate HA cluster for high availability

## High Availability

### Component HA

| Component | HA Support | Notes |
|-----------|------------|-------|
| MCP Server Pod | ✅ Yes | Can run multiple replicas |
| Kubernetes Service | ✅ Yes | Automatically load balances |
| RemoteMCPServer | ✅ Yes | CRD managed by kagent |
| Agent | ✅ Yes | Stateless, managed by kagent |
| FortiGate Device | ⚠️ External | Requires FortiGate HA cluster |

### Failure Scenarios

**Pod Failure:**
- Kubernetes automatically restarts pod
- Service continues routing to healthy pods
- Liveness/readiness probes detect failures

**Node Failure:**
- Pod rescheduled to healthy node
- Brief service interruption during rescheduling

**FortiGate Failure:**
- All API calls fail
- Agent reports FortiGate unreachable
- Requires FortiGate HA setup for resilience

## Monitoring and Observability

### Metrics to Monitor

**Kubernetes Metrics:**
- Pod CPU/memory usage
- Pod restart count
- Service endpoint health

**Application Metrics:**
- API call success/failure rate
- API call latency
- Tool invocation frequency

**FortiGate Metrics:**
- API response times
- Connection count
- Error rates

### Logging

**Pod Logs:**
```bash
kubectl logs -n kagent -l app=fortigate-mcp-server --tail=100 -f
```

**Log Sources:**
- FastMCP server logs
- httpx request/response logs
- Tool invocation logs
- Error traces

### Health Checks

**Liveness Probe:**
- Type: TCP Socket
- Port: 8082
- Initial Delay: 10s
- Period: 30s

**Readiness Probe:**
- Type: TCP Socket
- Port: 8082
- Initial Delay: 5s
- Period: 10s

## Deployment Architecture

### Deployment Methods

1. **Automated (deploy.sh):**
   - Pre-deployment validation
   - Credential checking
   - Sequential resource creation
   - Health verification

2. **Makefile:**
   - Build, push, deploy lifecycle
   - Status checking
   - Log streaming

3. **Manual (kubectl):**
   - Direct kubectl apply
   - Full control over timing

### Deployment Order

1. Secret (fortigate-credentials)
2. Deployment (fortigate-mcp-server)
3. Service (fortigate-mcp-server)
4. RemoteMCPServer (fortigate-mcp-remote)
5. Agent (fortigate-troubleshooting-agent)

### Update Strategy

**Deployment Strategy:** RollingUpdate
- Max Surge: 1
- Max Unavailable: 0

**Zero-Downtime Updates:**
- New pod started before old pod terminated
- Readiness probes ensure traffic only to healthy pods
- Automated rollback on failure

## Integration Points

### kagent Integration

**Custom Resources Used:**
- `RemoteMCPServer`: Registers MCP tools
- `Agent`: Defines AI troubleshooting agent

**Service Discovery:**
- DNS-based: fortigate-mcp-server.kagent.svc.cluster.local
- Port: 8082
- Protocol: HTTP

### FortiGate API Integration

**API Version:** v2
**Documentation:** FortiGate REST API Reference
**Rate Limits:** FortiGate device-dependent
**Timeout:** 30 seconds per request

### Extensibility

**Adding New Tools:**
1. Add @mcp.tool decorated function in main.py
2. Implement FortiGate API calls
3. Register tool name in agent tools list
4. Rebuild container image
5. Redeploy

**Supporting Multiple FortiGates:**
- Modify to accept device parameter
- Store multiple credentials in secret
- Route requests to appropriate device

## Troubleshooting Architecture

### Debug Flow

1. **Check Pod Status:** `kubectl get pods -n kagent`
2. **View Logs:** `kubectl logs -n kagent -l app=fortigate-mcp-server`
3. **Test Service:** Port-forward and curl MCP endpoint
4. **Verify Secret:** Check environment variables in pod
5. **Test FortiGate API:** Direct API call from debug pod

### Common Issues

**Pod Not Starting:**
- Missing secret
- Image pull errors
- Resource limits

**MCP Server Not Responding:**
- Service misconfiguration
- Pod not ready
- Network policies blocking traffic

**FortiGate Connection Failures:**
- Invalid credentials/token
- Network connectivity
- SSL certificate issues
- FortiGate API disabled

## Future Enhancements

### Planned Improvements

1. **Caching Layer:** Redis cache for frequently accessed objects
2. **Webhook Integration:** Real-time FortiGate event notifications
3. **Multi-Device Support:** Manage multiple FortiGate devices
4. **Enhanced Monitoring:** Prometheus metrics export
5. **Configuration Backup:** Automated FortiGate config backups
6. **Change Management:** Track and audit configuration changes
7. **Policy Simulation:** Test policy changes before applying
8. **Compliance Scanning:** Automated security policy audits

### Architecture Evolution

**Phase 1 (Current):** Read-only MCP server with AI agent
**Phase 2:** Add write capabilities for configuration changes
**Phase 3:** Multi-device management and orchestration
**Phase 4:** Full automation platform with change control

## Conclusion

This architecture provides a secure, scalable, and maintainable integration between kagent and FortiGate firewalls. The AI agent automates complex troubleshooting workflows, while the MCP server provides a clean abstraction layer over the FortiGate REST API.

The stateless design allows for horizontal scaling, and the Kubernetes-native deployment ensures high availability and operational simplicity.
