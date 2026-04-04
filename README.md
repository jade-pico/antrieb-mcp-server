# Antrieb MCP Server

**Instant clusters for AI agents. Real VMs. Real images.**

Say you're troubleshooting SELinux on CentOS 9 and you don't want to dirty your cluster with hallucinations. What do you do? You tell your AI: 'give me a CentOS 9 node' and in under a second, it has a clean CentOS 9 node to try fixes on. Can you do that today?

This is where Antrieb fits. It gives your AI access to real VM clusters. Your AI can provision multi-node clusters in under a second per node and run commands node by node. Same OS, same packages, same behavior. Not a container, a microVM, or some unknown Linux distro that approximates it. Full cloud VMs (e.g. Ubuntu, Alma, Arch, Alpine).

Most AI agent infrastructure providers offer custom distros. This works when building new apps. For the enterprise and most of the programmable real world, your AI agent must use real VMs for high fidelity. 
This is where Antrieb fits. It gives your AI access to real VM clusters. Your AI can provision multi-node clusters in under a second per node and run commands node by node. Same OS, same packages, same behavior. Not a container, a microVM, or some unknown Linux distro that approximates it. Full cloud VMs (CentOS Stream, Ubuntu, Alma, Arch, Alpine).

Most AI agent infrastructure providers offer custom distros. This works when building new apps. For the enterprise and most of the programmable real world, your AI agent must use real VMs for high fidelity. 

Root access, private networking, and passwordless SSH between nodes.

Antrieb is a remote MCP server — nothing to install. Add it to your config and start provisioning.

> **No credentials or cloud account required.** Antrieb runs entirely on its own infrastructure.

## Quick Start

Add this to your MCP client config (e.g. `.mcp.json` for Claude Desktop):

```json
{
  "mcpServers": {
    "antrieb": {
      "type": "http",
      "url": "https://antrieb.sh/mcp"
    }
  }
}
```

With an API key (get one at [antrieb.sh/dash](https://antrieb.sh/dash)):

```json
{
  "mcpServers": {
    "antrieb": {
      "type": "http",
      "url": "https://antrieb.sh/mcp",
      "headers": {
        "Authorization": "Bearer ant_YOUR_API_KEY"
      }
    }
  }
}
```

No local install, no dependencies, no Docker.

## How It Works

Your AI agent controls real VMs through 5 tools:

```
1. provision  →  Spin up VMs (sub-second per VM)
2. exec       →  Run commands on any node
3. save       →  Snapshot a node as a reusable image
4. search     →  Discover available images / clusters
5. delete     →  Destroy clusters / images
```

The agent drives the entire workflow — provision a cluster, install software command by command, verify each step, and iterate until it works. Antrieb provides the infrastructure; the AI provides the intelligence.

### Example Workflow

```
You: provision an 3 node ubuntu cluster and install nginx on all nodes


Agent: provision(cluster: ["ubuntu24.04 x3"])
→ { session_id: "abc12", nodes: ["node1", "node2", "node3"], provision_time_ms: 720 }

Agent: exec(session_id: "abc12", node: "node1", command: "apt-get update && apt-get install -y nginx")
→ { exit_code: 0, stdout: "..." }

Agent: exec(session_id: "abc12", node: "node1", command: "systemctl start nginx && curl -s localhost")
→ { exit_code: 0, stdout: "<html>Welcome to nginx...</html>" }

Agent: save(session_id: "abc12", node: "node1", name: "my-nginx", commands: [...])
→ { ani: "antrieb:my-nginx:v1" }

Agent: delete(session_id: "abc12")
→ { success: true }
```

## Tools

### `provision`

Spin up a VM cluster. Returns in under a second. Nodes get private IPs, `/etc/hosts` hostnames (`node1`, `node2`, ...), and passwordless SSH between all nodes.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `cluster` | array | yes | VM topology (e.g. `["ubuntu24.04 x3"]`) |

Returns `session_id`, `nodes`, `provision_time_ms`, `ttl_seconds`, `expires_at`.

### `exec`

Run a shell command on a specific node. Returns stdout, stderr, and exit code.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | yes | From `provision` |
| `node` | string | yes | Node name (e.g. `"node1"`) |
| `command` | string | yes | Shell command to execute |

### `save`

Save a node's current state as a reusable image. Antrieb generates build scripts and documentation from  commands and prompt.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | yes | From `provision` |
| `node` | string | yes | Node to save |
| `name` | string | yes | Image name (becomes `antrieb:<name>:v1`) |
| `commands` | array | yes | Ordered list of successful commands executed |

### `search`

Discover available images or list your active clusters.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | no | `"images"` (default) or `"clusters"` |
| `keywords` | string | no | Filter images by keyword |

### `delete`

Destroy a cluster or decommission an image.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | no | Cluster to destroy (use `"*"` for all) |
| `image` | string | no | Image to decommission |

## Available Images

### Example Base Images

| ANI | Description |
|-----|-------------|
| `ubuntu24.04` | Ubuntu 24.04 LTS — apt, bash, Python 3, curl, wget, jq |
| `almalinux9` | AlmaLinux 9 (RHEL-compatible) — dnf, bash, Python 3 |
| `archlinux` | Arch Linux (rolling) — pacman, bash, Python 3 |
| `centos-stream10` | CentOS Stream 10 — dnf, bash, Python 3 |
| `alpine` | Alpine Linux 3.23 — apk, minimal, musl libc |

### Example Stack Images

| ANI | Description |
|-----|-------------|
| `terraform-aws` | Terraform with real AWS (free-tier, Antrieb's credentials) |
| `cloudformation-aws` | CloudFormation with real AWS |
| `ansible-controller` | Ansible control node with collections |
| `podman-docker` | Podman, Buildah, Skopeo |

Use `search` to discover all available images with full descriptions, or just specify a distro name and Antrieb picks the right one.

## Cluster Networking

Every multi-node cluster gets:

- **Private IPs** — each node on a shared network
- **Hostname resolution** — `node1`, `node2`, `node3` in `/etc/hosts`
- **Passwordless SSH** — ed25519 keys distributed to all nodes
- **Firewall isolation** — clusters are isolated from each other

```
Agent: exec(node: "node1", command: "ssh node2 hostname")
→ { stdout: "node2" }
```

## Custom Images

Save any configured node as a reusable image:

1. Provision a base image
2. Install and configure software via `exec`
3. Call `save` with the list of successful commands
4. Antrieb generates `build-image.sh`, `startup.sh`, and a comprehensive description
5. The image is immediately available for future `provision` calls
6. Antrieb's replenisher automatically builds a pool of ready-to-use VMs from your image



## License

Apache 2.0
