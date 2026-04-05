# Antrieb MCP Server

**Instant clusters for AI agents. Real VMs. Real images.**

Say you're troubleshooting SELinux on CentOS 9 with a little help from AI, but you are worried about hallucinations. What do you do? You tell your AI: 'give me a CentOS 9 node' and in under a second, it has a clean CentOS 9 node to try fixes on. Can you do that today?

This is where Antrieb fits. It gives your AI access to real VMs. Your AI can provision multi-node clusters in under a second per node and run commands node by node. Same OS, same packages, same behavior. Not a container, a microVM, or some unknown Linux.

Other AI agent infrastructure providers offer customized distros. This works when building apps. For real-world infrastructure, your AI must use the same distros you run in prod: Ubuntu, CentOS, Alma, Arch, Alpine.

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

> **Note:** You can try Antrieb without an API key, but you won't be able to view your command history or create custom images. Get a free key at [antrieb.sh/dash](https://antrieb.sh/dash).

## How It Works

Your AI agent controls real VMs through 5 tools:

```
1. provision  →  Spin up VMs (sub-second per VM)
2. exec       →  Run commands on any node
3. save       →  Snapshot a node as a reusable image
4. search     →  Discover available images / clusters
5. delete     →  Destroy clusters / images
```

The agent drives the entire workflow: provision a cluster, install software command by command, verify each step, and iterate until it works. Antrieb provides the infrastructure; the AI provides the intelligence.

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

## Available Images

### Example Base Images

| ANI | Description |
|-----|-------------|
| `ubuntu24.04` | Ubuntu 24.04 LTS (apt, bash, Python 3, curl, wget, jq) |
| `almalinux9` | AlmaLinux 9, RHEL-compatible (dnf, bash, Python 3) |
| `archlinux` | Arch Linux rolling (pacman, bash, Python 3) |
| `centos-stream10` | CentOS Stream 10 (dnf, bash, Python 3) |
| `alpine` | Alpine Linux 3.23, minimal, musl libc (apk) |

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

- **Private IPs** on a shared network
- **Hostname resolution** (`node1`, `node2`, `node3` in `/etc/hosts`)
- **Passwordless SSH** with ed25519 keys distributed to all nodes
- **Firewall isolation** between clusters

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

## Try It Now

No MCP client needed. Just `curl`:

```bash
# 1. Provision a single Ubuntu node
curl -s -X POST https://antrieb.sh/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"provision","arguments":{"cluster":["ubuntu24.04"]}}}'
```

```json
→ { "session_id": "abc12...", "nodes": ["node1"], "provision_time_ms": 680 }
```

```bash
# 2. Run a command on the node
curl -s -X POST https://antrieb.sh/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"exec","arguments":{"session_id":"abc12...","node":"node1","command":"cat /etc/os-release | head -3"}}}'
```

```json
→ { "node": "node1", "exit_code": 0, "stdout": "PRETTY_NAME=\"Ubuntu 24.04 LTS\"\nNAME=\"Ubuntu\"\nVERSION_ID=\"24.04\"" }
```

```bash
# 3. Tear it down
curl -s -X POST https://antrieb.sh/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"delete","arguments":{"session_id":"abc12..."}}}'
```

> Replace `abc12...` with the `session_id` from step 1. Add `-H "Authorization: Bearer ant_YOUR_KEY"` to use your API key.

## Tool Reference

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

Save a node's current state as a reusable image. Antrieb generates build scripts and documentation from the commands and prompt.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | yes | From `provision` |
| `node` | string | yes | Node to save |
| `name` | string | yes | Image name (becomes `antrieb:<n>:v1`) |
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

## FAQ

**Is it free?**

We're in early access and free while we figure out what people actually need. We plan to always have a free tier, and we'll give plenty of notice before anything changes.

**What are the resource specs of each VM?**

Each VM gets 4 vCPUs and 8 GB of RAM.

**How long do clusters last?**

Each cluster has a TTL of 10 minutes. After that it's fully discarded: compute, networking, everything. If you ever get the same IP address as a previous session, the VM is completely fresh.

**What happens to a cluster when my chat session ends?**

The cluster is destroyed and the IP addresses are reused for future VMs. Your commands are logged separately and remain accessible in your dashboard.

**Are VMs truly ephemeral? Could I ever get a dirty node?**

Yes, fully ephemeral. VMs are never saved or snapshotted between sessions. Once you're done, they are destroyed. Gone. No undo. Every node you provision is completely fresh.

**Who can see the commands running on my VMs?**

Only you. No one else has access to your command history.

**Can my agent accidentally expose sensitive data in command logs?**

Please do not include sensitive data such as secrets or credentials in your commands. Command logs are accessible to you via the dashboard, so treat them accordingly.

**Are my VMs isolated from other users?**

Yes. Nodes within a cluster can reach each other, but a node cannot reach a node in a different cluster, including other clusters you own. To verify this yourself, tell your agent: *"Provision two single-node clusters, get each node's IP, then SSH into each and try to ping the other."* You'll see the ping fail.

**Can I SSH into a node myself, not just through the agent?**

No. All interactions go through the agent. Tell it what you want to do and it will do it for you. Everything is conversational.

**What data is logged? Do you store my commands?**

Yes. All commands are logged and made available to you in your dashboard at [antrieb.sh/dash](https://antrieb.sh/dash). Command history requires an API key.

**Which MCP clients does this work with?**

Cursor, Windsurf, and Claude Code. If your client supports remote MCP servers over HTTP, it should work.

**Does it work with models other than Claude?**

Yes. Antrieb is just an MCP server — the intelligence comes entirely from your AI agent. It works with any model your MCP client supports.

**How long does a saved image take to be ready?**

We target 2 minutes. The maximum is 5 minutes.

**Is my custom image private or visible to other users?**

Custom images are private by default. To share images within a team, go to your profile at [antrieb.sh/dash](https://antrieb.sh/dash) and set a namespace for your organization. From that point on, all your images are accessible only to members of your org.

**How is this different from E2B or Morph?**

E2B is excellent for code execution sandboxes, optimized for running code rather than infrastructure. You get a single sandbox, not a cluster, and the environment doesn't match production at the OS level. If you're testing infrastructure — Ansible playbooks, systemd behavior, SELinux policies — you need the real OS and real packages.

Morph is built for dev environments. Their devboxes are great for coding agents building and iterating on software. Antrieb is for infrastructure work, where fidelity to your actual distro matters: Ubuntu, AlmaLinux, Alpine, Arch. The same OS you run in prod, not a customized devbox.

**How is this different from Docker or GitHub Codespaces?**

Docker containers share a host kernel, so you can't reliably test SELinux policies, kernel modules, systemd behavior, or distro-specific package quirks. Antrieb gives your agent real VMs running the exact same OS you'd run in production.

Codespaces is built for humans: open a browser or editor, click around, type. Antrieb is built for agents. There's no UI to navigate, no workspace to configure. Your agent provisions a cluster, runs commands, reads output, and iterates, all through tool calls in the same conversation.

## License

Apache 2.0
