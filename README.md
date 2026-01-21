# FortiGate MCP Server for kagent

This repository contains an MCP (Model Context Protocol) server for FortiGate firewalls that can be deployed in a kagent Kubernetes environment. The server provides AI assistants with tools to query and manage FortiGate firewall objects.

## 📚 Documentation

- **[Architecture Overview](ARCHITECTURE.md)** - Detailed system architecture, component diagrams, and technical design
- **[Blog Post](BLOG_POST.md)** - Comprehensive guide with real-world examples and use cases
- **[Quick Start](#quick-start-for-kubernetes-deployment)** - Get started in minutes

## Features

### MCP Tools (8 tools available)
- **list_firewall_policies**: List and query FortiGate firewall policies
- **list_firewall_addresses**: Manage FortiGate address objects
- **list_firewall_services**: Query firewall service objects
- **get_system_status**: Get FortiGate system status and health information
- **list_interfaces**: List network interfaces and their configurations
- **list_static_routes**: Query static routing table entries
- **list_virtual_ips**: List virtual IP (VIP) configurations
- **discover_vdoms**: Discover available virtual domains (VDOMs)

### AI Agent
- **FortiGate Troubleshooting Agent**: Expert AI agent that uses all 8 MCP tools to diagnose firewall issues, analyze security policies, and troubleshoot connectivity problems

## Prerequisites

- Docker or Podman for building the container image
- Access to a container registry
- Kubernetes cluster with kagent installed
- FortiGate device with API access
- kubectl configured to access your cluster

## Project Structure

```
fortinet-mcp-kagent/
├── src/
│   ├── main.py              # MCP server implementation
│   └── requirements.txt     # Python dependencies
├── k8s/
│   ├── deploy.sh                            # Automated deployment script
│   ├── cleanup.sh                           # Cleanup script
│   ├── secret.yaml                          # FortiGate credentials (base64 encoded)
│   ├── deployment.yaml                      # Kubernetes Deployment
│   ├── service.yaml                         # Kubernetes Service
│   ├── remotemcpserver.yaml                 # kagent RemoteMCPServer resource
│   └── fortigate-troubleshooting-agent.yaml # AI Troubleshooting Agent
├── Dockerfile               # Container image definition
├── Makefile                 # Build and deployment automation
└── README.md               # This file
```

## Quick Start for Kubernetes Deployment

### Quick Deploy

```bash
cd k8s
./deploy.sh
```

### Manual Deploy

```bash
cd k8s
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f remotemcpserver.yaml
kubectl apply -f fortigate-troubleshooting-agent.yaml
```

### What Gets Deployed

The deployment creates the following resources in the `kagent` namespace:

1. **Secret** (`fortigate-credentials`) - Stores FortiGate credentials securely
2. **Deployment** (`fortigate-mcp-server`) - Runs the MCP server container
3. **Service** (`fortigate-mcp-server`) - Exposes the MCP server within the cluster
4. **RemoteMCPServer** (`fortigate-mcp-remote`) - Registers the MCP tools with kagent
5. **Agent** (`fortigate-troubleshooting-agent`) - AI agent for troubleshooting

## Configuration

### FortiGate Credentials

The FortiGate credentials are stored in `k8s/secret.yaml`. Update with your FortiGate device details:
- **FORTIGATE_HOST**: IP address only (e.g., `192.168.1.99` - no `https://` prefix)
- **FORTIGATE_USER**: Username with API access (e.g., `admin`)
- **FORTIGATE_PASS**: Password for the user
- **FORTIGATE_TOKEN**: (Optional) API token - preferred over username/password
- **FORTIGATE_VDOM**: (Optional) Virtual domain name, defaults to `root`

To encode credentials:
```bash
echo -n "192.168.1.99" | base64
echo -n "admin" | base64
echo -n "your-password" | base64
```

**Security Note**: In production, use external secret management solutions like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault.

### Using API Token (Recommended)

API tokens are more secure than username/password authentication. To use an API token:

1. Generate an API token in your FortiGate device:
   - Go to System > Administrators
   - Create a new API user or edit existing
   - Generate an API token

2. Encode and add to `k8s/secret.yaml`:
```bash
echo -n "your-api-token" | base64
```

3. Update `k8s/secret.yaml`:
```yaml
data:
  FORTIGATE_TOKEN: <base64-encoded-token>
```

4. Uncomment the FORTIGATE_TOKEN section in `k8s/deployment.yaml`

## Building the Container Image

If you need to build and push your own image:

```bash
# Using Make
make build REGISTRY=sebbycorp
make push REGISTRY=sebbycorp

# Or manually with Docker
docker build -t sebbycorp/fortigate-mcp-server:latest .
docker push sebbycorp/fortigate-mcp-server:latest
```

Then update `k8s/deployment.yaml` with your image location.

## Testing the MCP Server

### Test from within the cluster

You can test the MCP server directly using a debug pod:

```bash
# Create a debug pod
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n kagent -- sh

# From within the pod, test the MCP endpoint (JSON-RPC 2.0)
curl -X POST http://fortigate-mcp-server.kagent.svc.cluster.local:8082/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```

### Test via kagent

Once registered with kagent, you can use the FortiGate MCP tools through your AI assistant.

#### Using the FortiGate Troubleshooting Agent (Recommended)

The FortiGate Troubleshooting Agent is an expert AI that automatically uses the right tools and workflows to diagnose issues:

**Traffic Troubleshooting:**
- "My traffic is being blocked to 10.0.1.50 on port 443, can you help?"
- "Why can't I access the web server at 192.168.10.100?"
- "Check if there's a firewall policy blocking SSH traffic"

**NAT/VIP Issues:**
- "My port forwarding isn't working for VIP web-server"
- "Check the NAT configuration for external IP 203.0.113.10"
- "Why isn't my virtual IP working?"

**General Health Checks:**
- "Show me the overall health of the FortiGate"
- "Are there any interfaces that are down?"
- "Check if there are any disabled firewall policies"

**Policy Analysis:**
- "Show me all firewall policies and highlight any issues"
- "Is there a policy allowing traffic from DMZ to internal network?"
- "Check policy ID 5 for any misconfigurations"

**Routing Problems:**
- "I can't reach network 172.16.0.0/16"
- "Check the routing table for the default gateway"
- "Why isn't traffic routing to the internet?"

#### Using MCP Tools Directly

You can also call individual tools:

- "List all firewall policies on the FortiGate"
- "Show me the firewall address objects"
- "What is the FortiGate system status?"
- "List all network interfaces"
- "Show me the static routes"
- "What virtual IPs are configured?"

## FortiGate Troubleshooting Agent

The deployment includes an AI agent specifically trained to troubleshoot FortiGate firewalls. This agent:

- **Automatically selects the right tools** for your question
- **Follows expert troubleshooting workflows** for common scenarios
- **Provides actionable recommendations** based on configuration analysis
- **Understands FortiGate concepts** like policies, VIPs, VDOMs, and routing

### Agent Capabilities

1. **Security Policy Analysis** - Reviews firewall rules, identifies blocked traffic, finds policy conflicts
2. **Network Connectivity Troubleshooting** - Diagnoses routing issues, interface problems, and connectivity failures
3. **NAT/VIP Configuration** - Analyzes port forwarding, virtual IPs, and external mappings
4. **System Health Monitoring** - Checks device status, interface states, and overall health
5. **Configuration Discovery** - Explores and documents firewall objects and settings

### Example Workflows

The agent uses sophisticated workflows to solve problems:

**Traffic Blocked Workflow:**
1. Checks system status
2. Reviews relevant firewall policies
3. Verifies source/destination address objects
4. Checks service definitions
5. Reviews interface status
6. Analyzes routing
7. Provides specific recommendations

**NAT Not Working Workflow:**
1. Examines VIP configuration
2. Reviews associated policies
3. Verifies IP mappings
4. Checks interface bindings
5. Analyzes return traffic routing
6. Identifies misconfigurations

### When to Use the Agent vs Direct Tools

**Use the Agent when:**
- Troubleshooting an issue (traffic blocked, NAT not working, routing problems)
- Need comprehensive analysis across multiple configuration areas
- Want expert recommendations and explanations
- Diagnosing complex multi-component issues

**Use Direct Tools when:**
- You know exactly which tool you need
- Simple queries (list all policies, show interfaces)
- Building your own scripts or automation
- Quick lookups of specific objects

## Available MCP Tools

### list_firewall_policies

Lists FortiGate firewall policies.

**Parameters:**
- `policy_id` (optional): Specific policy ID for detailed info

**Example:**
```
list_firewall_policies()
list_firewall_policies(policy_id=1)
```

### list_firewall_addresses

Lists FortiGate firewall address objects.

**Parameters:**
- `address_name` (optional): Specific address name for detailed info

**Example:**
```
list_firewall_addresses()
list_firewall_addresses(address_name="webserver")
```

### list_firewall_services

Lists FortiGate firewall service objects.

**Parameters:**
- `service_name` (optional): Specific service name for detailed info

**Example:**
```
list_firewall_services()
list_firewall_services(service_name="custom-http")
```

### get_system_status

Gets FortiGate system status information including hostname, version, uptime, and HA mode.

**Example:**
```
get_system_status()
```

### list_interfaces

Lists FortiGate network interfaces.

**Parameters:**
- `interface_name` (optional): Specific interface name for detailed info

**Example:**
```
list_interfaces()
list_interfaces(interface_name="port1")
```

### list_static_routes

Lists FortiGate static routes.

**Parameters:**
- `route_id` (optional): Specific route sequence number for detailed info

**Example:**
```
list_static_routes()
list_static_routes(route_id=1)
```

### list_virtual_ips

Lists FortiGate virtual IPs (VIPs).

**Parameters:**
- `vip_name` (optional): Specific VIP name for detailed info

**Example:**
```
list_virtual_ips()
list_virtual_ips(vip_name="web-vip")
```

### discover_vdoms

Discovers available VDOMs on the FortiGate device.

**Example:**
```
discover_vdoms()
```

## Troubleshooting

### Pod not starting

Check the pod logs:
```bash
kubectl logs -l app=fortigate-mcp-server -n kagent
```

Common issues:
- Missing FortiGate credentials in secret
- Network connectivity to FortiGate device
- Invalid FortiGate credentials or API token
- SSL certificate verification issues

### Connection timeout to FortiGate

If you see connection timeout errors, verify:
- The FortiGate device is reachable from the Kubernetes cluster
- The FORTIGATE_HOST environment variable is correct
- Firewall rules allow traffic from the cluster to FortiGate
- FortiGate management interface is accessible

### MCP server not responding

Check the service and endpoints:
```bash
kubectl get endpoints fortigate-mcp-server -n kagent
kubectl describe service fortigate-mcp-server -n kagent
```

### RemoteMCPServer not registered

Check the RemoteMCPServer status:
```bash
kubectl describe remotemcpserver fortigate-mcp-remote -n kagent
```

## Security Considerations

1. **TLS/SSL**: The current setup uses `verify=False` to bypass SSL certificate validation. In production, consider:
   - Using valid SSL certificates on FortiGate
   - Implementing proper certificate validation
   - Using mutual TLS

2. **API Token**: Always prefer API tokens over username/password authentication

3. **Secrets Management**: Consider using:
   - Kubernetes external secrets operators
   - Cloud provider secret managers (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager)
   - HashiCorp Vault

4. **Network Policies**: Implement Kubernetes Network Policies to restrict:
   - Ingress traffic to the MCP server
   - Egress traffic to only the FortiGate device

5. **RBAC**: Ensure proper Kubernetes RBAC is configured for accessing the FortiGate MCP resources

6. **VDOM Isolation**: Use appropriate VDOM settings to limit scope of access

## Updating Credentials

To update FortiGate credentials:

1. Edit the credentials:
```bash
# Encode new credentials
echo -n "new-password" | base64
```

2. Update `k8s/secret.yaml` with the new base64 encoded values

3. Apply the updated secret:
```bash
kubectl apply -f k8s/secret.yaml
```

4. Restart the deployment:
```bash
kubectl rollout restart deployment fortigate-mcp-server -n kagent
```

## Scaling

The current deployment runs a single replica. To scale:

```bash
kubectl scale deployment fortigate-mcp-server -n kagent --replicas=3
```

Note: Multiple replicas will all connect to the same FortiGate device. Ensure your FortiGate device can handle the connection load.

## Monitoring

To monitor the FortiGate MCP server:

```bash
# Watch pod status
kubectl get pods -l app=fortigate-mcp-server -n kagent -w

# Stream logs
kubectl logs -f -l app=fortigate-mcp-server -n kagent

# Check resource usage
kubectl top pod -l app=fortigate-mcp-server -n kagent
```

## Makefile Commands

```bash
make help        # Show available commands
make build       # Build Docker image
make push        # Push image to registry
make deploy      # Deploy to Kubernetes
make undeploy    # Remove from Kubernetes
make status      # Show deployment status
make logs        # Show server logs
make restart     # Restart the deployment
```

## Additional Documentation

### Architecture Deep Dive
See [ARCHITECTURE.md](ARCHITECTURE.md) for:
- Complete component architecture with diagrams
- Data flow and integration points
- Security architecture and best practices
- Scaling and high availability considerations
- Troubleshooting guide

### Blog Post & Tutorial
See [BLOG_POST.md](BLOG_POST.md) for:
- Real-world usage examples
- Step-by-step implementation guide
- Lessons learned and best practices
- Performance metrics and ROI analysis
- Comparison with alternatives

## Contributing

We welcome contributions! Please:
1. Fork the repository
2. Create a feature branch
3. Submit a pull request

Areas for contribution:
- Additional MCP tools
- Enhanced agent workflows
- Documentation improvements
- Bug fixes and optimizations

## License

Apache License 2.0

## Support

For issues and questions:
- Check the logs: `kubectl logs -l app=fortigate-mcp-server -n kagent`
- Review FortiGate connectivity from the pod
- Verify kagent configuration
- Ensure FortiGate API access is properly configured
- See [ARCHITECTURE.md](ARCHITECTURE.md) for troubleshooting guide

## Related Projects

- **[F5 BIG-IP MCP Server](../f5-mcp-kagent/)** - Similar implementation for F5 load balancers
- Built using the same architecture pattern and best practices
