# Antrieb MCP Server

**Instant clusters for AI agents. Real VMs. Real images.**

Say you're troubleshooting SELinux on CentOS Stream 10 with a little help from Claude Code, but you are worried about hallucinations. What do you do? You tell it: 'give me a CentOS Stream 10 node' and in under a second, it has a disposable node to prove the fix works before it touches your environment. Can you do that today?

This is where Antrieb fits. It gives your AI access to real VMs to validate its output. Tell it to spin up 3 nodes and install a k3s cluster: it provisions, installs, verifies, and iterates entirely on its own. Or stay in the loop and go step by step. Either way, same OS, same packages, same behavior. Not a container, a microVM, or some unknown Linux. Your AI must validate against the same distros you actually run: CentOS Stream, Ubuntu, Alma, Arch, Alpine.

Root access, private networking, and passwordless SSH between nodes. Ten minutes per cluster. Clean slate every time.

Antrieb is a remote MCP server. Nothing to install. Add it to your config and start provisioning.

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

> **Note:** You can try Antrieb without an API key, but you won't be able to view your command history, save custom images, or view active clusters.

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
You: provision 3 ubuntu nodes and install nginx on all nodes


Agent: provision(cluster: ["ubuntu24.04", "ubuntu24.04", "ubuntu24.04"])
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
| `ubuntu24.04` | Ubuntu 24.04 LTS: apt, bash, Python 3, curl, wget, jq |
| `almalinux9` | AlmaLinux 9 (RHEL-compatible): dnf, bash, Python 3 |
| `archlinux` | Arch Linux (rolling): pacman, bash, Python 3 |
| `centos-stream10` | CentOS Stream 10: dnf, bash, Python 3 |
| `alpine` | Alpine Linux 3.23: apk, minimal, musl libc |

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

- **Private IPs:** each node on a shared network
- **Hostname resolution:** `node1`, `node2`, `node3` in `/etc/hosts`
- **Passwordless SSH:** ed25519 keys distributed to all nodes
- **Firewall isolation:** clusters are isolated from each other
- **Full internet access:** nodes can reach the public internet directly (package registries, APIs, etc.)

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

> **Note:** Saving custom images requires an API key.




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

## FAQ

**Why Antrieb?**

AI is moving into every layer of operations: cloud, on-prem, legacy systems, security, networking, and edge. Misconfiguring a cloud server is painful. Misconfiguring ten thousand edge devices in the field is catastrophic.

Antrieb is the validation layer between AI and Operations. Before AI touches your environment, it validates against the real thing first. Same OS, same packages, same behavior. Not a container. Not an approximation.



**Is it free?**

We're in early access and free while we figure out what people actually need. We plan to always have a free tier, and we'll give plenty of notice before anything changes.

**What are the resource specs of each VM?**

Each VM gets 4 vCPUs, 8 GB of RAM, and 20 GB of disk.

**How many nodes can a cluster have?**

Up to 4 nodes per cluster. If your topology requires a dedicated controller node (for example, an Ansible control node managing a fleet), one additional node is allowed, bringing the maximum to 5.

**How many clusters can I run at once?**

Up to 2 concurrent clusters per account.

**Do nodes have internet access?**

Yes. Every node has full, direct internet access. Package managers, curl, pip, npm: all work as you'd expect.

**How long do clusters last?**

Each cluster has a hard TTL of 10 minutes. After that it is fully discarded: compute, networking, everything. TTLs cannot be extended. If you ever get the same IP address as a previous session, the VM is completely fresh.

**What happens to a cluster when my chat session ends?**

The cluster is destroyed and the IP addresses are reused for future VMs. Your commands are logged separately and remain accessible in your dashboard.

**Are VMs truly ephemeral? Could I ever get a dirty node?**

Yes, fully ephemeral. VMs are never saved or snapshotted between sessions. Once you're done, they are destroyed.  Every node you provision is completely fresh.

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

Claude Desktop, Cursor, Windsurf, and Claude Code. If your client supports remote MCP servers over HTTP, it should work.

**Does it work with models other than Claude?**

Yes. Antrieb is just an MCP server; the intelligence comes entirely from your AI agent. It works with any model your MCP client supports.

**How long does a saved image take to be ready?**

We target 2 minutes. The maximum is 5 minutes.

**Is my custom image private or visible to other users?**

Custom images are private by default. To share images within a team, go to your profile at [antrieb.sh/dash](https://antrieb.sh/dash) and set a namespace for your organization. From that point on, all your images are accessible only to members of your org.

**How is this different from E2B, Morph?**

**vs E2B:** E2B is excellent for code execution sandboxes. It's optimized for running code, not infrastructure. You get a single sandbox, not a cluster, and the environment doesn't match yours at the OS level. If you're testing infra (Ansible playbooks, systemd behavior, SELinux policies), you need the real OS and real packages.

**vs Morph Cloud:** Morph gives your agent a dev environment. Morph's devboxes are great for coding agents building and iterating on software. Antrieb is for infrastructure work, where fidelity to your actual distro matters: Ubuntu, AlmaLinux, Alpine, Arch, the same OS you actually run, not a customized devbox.

**How is this different from just using a Docker container or GitHub Codespaces?**

Docker containers share a host kernel, so you can't reliably test SELinux policies, kernel modules, systemd behavior, or distro-specific package quirks in a container. Antrieb gives your agent real VMs running the exact same OS you'd run in your environment.

Codespaces is built for humans. You open a browser or editor, click around, and type. Antrieb is built for agents: no UI to navigate, no workspace to configure. Your agent provisions a cluster, runs commands, reads output, and iterates, all through tool calls in the same conversation. It's the difference between giving your AI a screenshot to click through versus a direct API.


## Tool Reference

### `provision`

Spin up a VM cluster.  Nodes get private IPs, `/etc/hosts` hostnames (`node1`, `node2`, ...), and passwordless SSH between all nodes.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `cluster` | array | yes | VM topology (e.g. `["ubuntu24.04", "ubuntu24.04", "ubuntu24.04"]`) |

Returns `session_id`, `nodes`, `provision_time_ms`, `ttl_seconds`, `expires_at`.

### `exec`

Run a shell command on a specific node. Returns stdout, stderr, and exit code.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | yes | From `provision` |
| `node` | string | yes | Node name (e.g. `"node1"`) |
| `command` | string | yes | Shell command to execute |

### `save`

Save a node's current state as a reusable image. Antrieb generates build scripts and documentation from commands and prompt.

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

## License

Apache 2.0
