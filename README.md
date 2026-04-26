# Antrieb MCP Server

**Instant disposable clusters for LLM-generated networks and infra.**


Say an LLM writes a VyOS NAT config, an OpenWrt firewall rule set, or a bash script to harden a CentOS cluster. Where do you run it before it touches your environment? Antrieb gives the LLM a disposable multi-node cluster, with real VMs and real networking, to try what it generates. Break it, quickly reprovision, try again. No cleanup.

Same OS and appliances your LLM-generated code will target in production: CentOS Stream, Ubuntu, Alma, Arch, Alpine, VyOS, OPNsense, SONiC, OpenWrt. Multi-network topologies with per-NIC assignment. Not a container, not a microVM, not some unknown Linux — the same kernels and packages your real fleet runs.

Root access, private networking, passwordless SSH between nodes. Ten minutes per cluster. Clean slate every time.

Antrieb is a remote MCP server. Nothing to install. Add it to your config and start provisioning.


## Quick Start

Antrieb is a remote MCP server. Two ways to connect:

### Claude Web / ChatGPT

With web-based AI clients, add Antrieb as a custom connector.

In **claude.ai**: go to **Settings → Connectors**, click **Add custom connector**. Enter:

- **Name:** `Antrieb`
- **URL:** `https://antrieb.sh/mcp`

Click **Add**, then **Connect** and sign in. Back in your chat, try something like:

> *Create a 3-node Kubernetes cluster, with node1 running both as control-plane and worker, and node2/node3 as pure workers. Install nginx with 3 pods and pod anti-affinity.*

Walkthrough video: <https://youtu.be/8nts8ol-yeA>

### MCP Clients (Claude Desktop, Claude Code, Codex, custom clients)

Get a free API key at [antrieb.sh/dash](https://antrieb.sh/dash), then add this to your MCP client config (e.g. `.mcp.json` for Claude Desktop):

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

Your LLM controls real VMs through 5 tools:

```
1. provision  →  Spin up a cluster (sub-second per VM, declare custom networks and NICs)
2. exec       →  Run commands on any node (cluster-scoped env vars auto-injected)
3. save       →  Snapshot a node as an image, or save a runbook / network spec
4. search     →  Discover images, runbooks, networks, or active clusters
5. delete     →  Destroy a cluster, or decommission a saved artifact
```

The LLM drives the entire workflow: provision a cluster, install software command by command, verify each step, and iterate until it works. Antrieb provides the infrastructure; the LLM provides the intelligence.

### Example Workflow

```
You: provision 3 ubuntu nodes and install nginx on all nodes


LLM: provision(cluster: ["ubuntu24.04 x3"])
→ { session_id: "abc12", nodes: ["node1", "node2", "node3"], provision_time_ms: 720 }

LLM: exec(session_id: "abc12", node: "node1", command: "apt-get update && apt-get install -y nginx")
→ { exit_code: 0, stdout: "..." }

LLM: exec(session_id: "abc12", node: "node1", command: "systemctl start nginx && curl -s localhost")
→ { exit_code: 0, stdout: "<html>Welcome to nginx...</html>" }

LLM: save(type: "image", session_id: "abc12", node: "node1", name: "my-nginx", commands: [...])
→ { ani: "antrieb:my-nginx:v1" }

LLM: delete(type: "cluster", name: "abc12")
→ { success: true }
```

## Available Images

### Base Images

| ANI | Description |
|-----|-------------|
| `ubuntu24.04` | Ubuntu 24.04 LTS: apt, bash, Python 3, curl, wget, jq |
| `almalinux9` | AlmaLinux 9 (RHEL-compatible): dnf, bash, Python 3 |
| `archlinux` | Arch Linux (rolling): pacman, bash, Python 3 |
| `centos-stream10` | CentOS Stream 10: dnf, bash, Python 3 |
| `alpine` | Alpine Linux 3.23: apk, minimal, musl libc |
| `sonic` | SONiC: open-source network OS for data-center switches |
| `vyos` | VyOS: Linux-based router / firewall with a unified CLI |
| `opnsense` | OPNsense: FreeBSD-based firewall / router with web UI |

### Stack Images

Stack images are custom images created by enriching the base images with additional packages.

| ANI | Description |
|-----|-------------|
| `terraform-aws` | Terraform with real AWS (free-tier, Antrieb's credentials, Vault-brokered STS) |
| `cloudformation-aws` | CloudFormation with real AWS |
| `ansible-controller` | Ansible control node with collections |
| `podman` | Podman, Buildah, Skopeo (rootless, Docker-API-compatible) |

Use `search` to discover all available images with full descriptions, or just specify a distro name and Antrieb picks the right one.

## Cluster Networking

Networking is a first-class input to `provision`, not an afterthought. Every cluster is **L2-isolated from every other cluster** (even ones you own), with hostname resolution, passwordless SSH between its own nodes, and cluster-scoped env vars auto-injected into every `exec` call. What *varies* between runs is how nodes are wired and controlled by the optional `networks` and `nics` parameters on `provision`.

There are two modes: the **default network** (flat, one-subnet, internet-reachable) and **declared networks** (first-class multi-subnet, multi-NIC topologies).

### Every cluster (both modes)

- **Hostname resolution:** `node1`, `node2`, `node3` in `/etc/hosts` on every node.
- **Passwordless SSH:** a per-cluster ed25519 keypair distributed to all nodes. `ssh node2 hostname` just works.
- **L2 isolation:** nodes across different clusters cannot reach each other, regardless of CIDR overlap or account.
- **Env vars on every `exec`:** `NODE_NAME`, `NODE_INDEX`, `NODE_IP`, `CLUSTER_HOSTS` (multiline `IP NAME` — drop-in for `/etc/hosts`), `CLUSTER_SSH_PUBKEY`, `CLUSTER_SSH_PRIVKEY`.

```
LLM: exec(node: "node1", command: "ssh node2 hostname")
→ { stdout: "node2" }
```

### Default network (when `networks` / `nics` are omitted)

```
LLM: provision(cluster: ["ubuntu24.04 x3"])
```

You get:

- One auto-allocated `/24` network named `default`, one NIC per node, DHCP-assigned IPs.
- **Egress to the internet** via a curated port allowlist (HTTP, HTTPS, DNS, SSH, package managers). Arbitrary outbound is blocked.
- No router VM, no firewall zones, no topology to configure. Nodes talk to each other and to the outside world.

Use this whenever the *network* isn't what you're testing — application-level work, distro-specific configuration, Ansible playbooks, CI pipelines, Kubernetes-on-three-ubuntus.

### Declared networks (multi-network, multi-NIC)

Pass `networks` and `nics` when the topology itself is the thing under test: routers, firewalls, DMZs, isolated back-end subnets, dual-homed nodes, BGP meshes.

```
LLM: provision(
  cluster: ["vyos", "ubuntu24.04", "ubuntu24.04"],
  networks: [
    { name: "wan", egress: true },
    { name: "lan", cidr: "10.10.1.0/24", egress: false }
  ],
  nics: {
    node1: [{ net: "wan" }, { net: "lan" }],   // vyos: dual-homed router
    node2: [{ net: "lan" }],                    // ubuntu backend
    node3: [{ net: "lan" }]                     // ubuntu backend
  }
)
```

Contract:

- **Each declared network is its own L2 segment** with its own DHCP server and its own `/24`. CIDRs are auto-allocated when omitted; only `/24` is supported.
- **`egress: true`** — internet-reachable on the default port allowlist. **`egress: false`** — traffic stays inside the network; no default route out.
- **`dhcp: true`** (default) — the platform assigns IPs. **`dhcp: false`** — you assign IPs (use this when a router VM owns addressing or you need precise /31 links).
- **NIC order matters.** Each entry in `nics.<node>` becomes `eth0`, `eth1`, ... in the listed order.
- **`.1` of every DHCP-enabled network is reserved** for the platform bridge gateway. Router VMs should claim `.254` (or DHCP-client) — assigning `.1` to a VM causes ARP flip-flop.

This unlocks serious networking work: VyOS / OPNsense firewalls with WAN/LAN zones, SONiC as an L2 switch or L3 router, OpenWrt with its `fw3` zone model, nginx behind a reverse-proxy LAN, BGP peering over /31 transit links, hairpin NAT, DNAT port-forwards. Full walkthroughs for each pattern live in the `@antrieb/*` runbook library.

### Saved network specs

Reusable topology fragments can be saved as network specs and referenced in subsequent provisions as `@namespace/name`:

```
LLM: save(type: "network", name: "isolated-lan", egress: false, cidr: "10.50.0.0/24")
→ { fq_name: "yourorg/isolated-lan" }

LLM: provision(
  cluster: [...],
  networks: ["@yourorg/isolated-lan", { name: "wan", egress: true }]
)
```

Useful when the same LAN shape gets reused across many scenarios — e.g., an org's standard DMZ layout.

## Runbooks

Runbooks are the way to preserve non-trivial multi-node scenarios — "set up HA Postgres with a pgbouncer frontend", "three-node VyOS mesh with BGP", "ansible controller managing two targets". They're markdown documents with structure the LLM can execute:

- `## Topology required` — cluster shape this applies to (images, networks, NICs)
- `## Variables` — placeholders like `<LAN_SUBNET>` callers substitute at apply time
- `## Steps` — numbered; prose explains *why*, fenced shell blocks are the actual commands, each annotated with the node it targets
- `## Verify` — how to confirm success

Save one with `save(type: "runbook", name: "...", body: "...markdown...")`. Apply one by running `search(type: "runbook", fq_name: "namespace/name")` to fetch the body, then issuing the `exec` calls it describes.

Runbooks complement images: an image captures **installed state** (packages, configs baked into a qcow2). A runbook captures **a workflow** (the ordered, multi-node actions that produce a working system). Prefer runbooks for anything with more than one node, because a single image can't express coordination between VMs.

Everything you save lives in your org's namespace; the `antrieb/*` namespace is read-only defaults.

## Custom Images

Save any configured node as a reusable image:

1. Provision a base image
2. Install and configure software via `exec`
3. Call `save(type: "image", ...)` with the list of successful commands
4. Antrieb generates `build-image.sh`, `startup.sh`, and a description
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
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"delete","arguments":{"type":"cluster","name":"abc12..."}}}'
```

> Replace `abc12...` with the `session_id` from step 1. Add `-H "Authorization: Bearer ant_YOUR_KEY"` to use your API key.

## FAQ

**Why Antrieb?**

AI is moving into every layer of operations: cloud, on-prem, legacy systems, security, networking, and edge. Misconfiguring a cloud server is painful. Misconfiguring ten thousand edge devices in the field is catastrophic.

Antrieb is the validation layer between LLMs and Operations. Before an LLM touches your environment, it validates against the real thing first. Same OS, same packages, same behavior. Not a container. Not an approximation.

**Why 10-minute clusters?**

The 10-min TTL is not about compute limits. Clusters are cheap and fast to recreate. The TTL exists to prevent **state drift**, **structural drift**, and **cognitive drift**.
- **State drift:** the LLM cannot rely on leftover files, partial fixes, or hidden changes from previous attempts. Each solution must work from a clean slate.
- **Structural drift:** clusters are immutable after provisioning. Nodes and NICs cannot be added later, so the LLM must choose the topology upfront instead of continuously adding and removing component.
- **Cognitive drift:** the reset prevents long debugging rabbit holes. The LLM must stop, assess progress, distill what it learned, and restart with a better plan.


If a solution cannot be reproduced on a fresh cluster, it does not count.


**Is it free?**

Yes.

**What are the resource specs of each VM?**

Each VM gets 4 vCPUs, 2 GB of RAM, and 20 GB of disk.

**How many nodes can a cluster have?**

Up to 4 nodes per cluster. If your topology requires a dedicated controller node (for example, an Ansible control node managing a fleet), one additional node is allowed, bringing the maximum to 5.

**How many clusters can I run at once?**

Up to 2 concurrent clusters per account.

**Do nodes have internet access?**

Yes, by default. Every node on the default network has direct internet access, though for security reasons only certain ports are allowed. Package managers, curl, pip, npm: all work as you'd expect. If you declare explicit networks with `egress: false`, nodes on those networks are fully isolated from the internet.

**How long do clusters last?**

Each cluster has a hard TTL of 10 minutes. After that it is fully discarded: compute, networking, everything. TTLs cannot be extended. If you ever get the same IP address as a previous session, the VM is completely fresh.

**What happens to a cluster when my chat session ends?**

The cluster is destroyed and the IP addresses are reused for future VMs. Your commands are logged separately and remain accessible in your dashboard.

**Are VMs truly ephemeral? Could I ever get a dirty node?**

Yes, fully ephemeral. VMs are never saved or snapshotted between sessions. Once you're done, they are destroyed.  Every node you provision is completely fresh.

**Who can see the commands running on my VMs?**

Only you. No one else has access to your command history.

**Can my LLM accidentally expose sensitive data in command logs?**

Please do not include sensitive data such as secrets or credentials in your commands. Command logs are accessible to you via the dashboard, so treat them accordingly.

**Are my VMs isolated from other users?**

Yes. Nodes within a cluster can reach each other, but a node cannot reach a node in a different cluster, including other clusters you own. To verify this yourself, tell your LLM: *"Provision two single-node clusters, get each node's IP, then SSH into each and try to ping the other."* You'll see the ping fail.

**Can I SSH into a node myself, not just through the LLM?**

No. All interactions go through the LLM. Tell it what you want to do and it will do it for you. Everything is conversational.

**What data is logged? Do you store my commands?**

Yes. All commands are logged and made available to you in your dashboard at [antrieb.sh/dash](https://antrieb.sh/dash).

**Which  clients does this work with?**

Claude.ai, ChatGPT Web, Claude Desktop, Cursor, Windsurf, and Claude Code. If your client supports remote MCP servers over HTTP, it should work.

**Does it work with models other than Claude?**

Yes. Antrieb is just an MCP server; the intelligence comes entirely from your LLM. It works with any model your MCP client supports.

**How long does a saved image take to be ready?**

We target 2 minutes. The maximum is 5 minutes. Runbooks and network specs save instantly — they're metadata, not qcow2 images.

**Are custom images, runbooks, and network specs private?**

Yes, private by default. To share within a team, go to your profile at [antrieb.sh/dash](https://antrieb.sh/dash) and set a namespace for your organization. From that point on, everything you save is accessible only to members of your org.

**How does Antrieb prevent abuse?**

- **Command logging and AI monitoring.** Every command executed on every node is logged. We use AI to aggressively detect and prevent abuse, including crypto mining, DDoS, spam, and unauthorized scanning.
- **Cluster limits.** Maximum 2 concurrent clusters per user. Up to 4 nodes per cluster.
- **Cluster-to-cluster isolation.** Nodes within a cluster can communicate. Nodes across clusters cannot, even within the same account.
- **Egress port allowlist.** Outbound traffic on the default network is restricted to a curated set of ports (HTTP, HTTPS, DNS, SSH, package managers). Arbitrary outbound connections are blocked. Networks declared with `egress: false` have no outbound path at all.

**How is this different from E2B, Morph?**

**vs E2B:** E2B is excellent for code execution sandboxes. It's optimized for running code, not infrastructure. You get a single sandbox, not a cluster, and the environment doesn't match yours at the OS level. If you're testing infra (Ansible playbooks, systemd behavior, SELinux policies, network), you need the real OS and real packages.

**vs Morph Cloud:** Morph gives your agent a dev environment. Morph's devboxes are great for coding agents building and iterating on software. Antrieb is for infrastructure work, where fidelity to your actual distro matters: Ubuntu, AlmaLinux, Alpine, Arch, SONiC, the same OS you actually run, not a customized devbox.

**How is this different from just using a Docker container or GitHub Codespaces?**

Docker containers share a host kernel, so you can't reliably test SELinux policies, kernel modules, systemd behavior, or distro-specific package quirks in a container. Antrieb gives your LLM real VMs running the exact same OS you'd run in your environment.

Beyond Linux distros, many network appliances and infrastructure platforms (SONiC, Cisco IOS-XE, Palo Alto, F5, Juniper) only ship as VM images or have limited container availability. When a container version exists, it is almost always a subset of the real thing: missing features, different APIs, incomplete behavior. Tracking what each vendor's containerized version actually supports becomes a problem in itself. With VMs, you run the real appliance image and get full fidelity.

Codespaces is built for humans. You open a browser or editor, click around, and type. Antrieb is built for LLMs to validate their work. Your LLM provisions a cluster, runs commands, reads output, and iterates, all through tool calls in the same conversation. It's the difference between giving your LLM a screenshot to click through versus a direct API.


## Tool Reference

### `provision`

Spin up a VM cluster. Returns a session handle, node names, TTL, and how long the provision took. Nodes get private IPs, `/etc/hosts` entries, and a shared ed25519 keypair for passwordless SSH.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `cluster` | array | yes | VM topology. Each entry is either a shorthand string (`"ubuntu24.04"` = 1 VM, `"ubuntu24.04 x3"` = 3 VMs) or an object `{image, count?}`. |
| `networks` | array | no | Per-cluster private networks. Each is `{name, cidr?, egress?, dhcp?}` or a string `@namespace/name` referring to a saved spec. Default: one network named `default` with NAT egress. |
| `nics` | object | no | Per-node NIC assignments keyed by node name, e.g. `{ "node1": [{"net":"lan"}] }`. Default: every node gets one NIC on the first network. |

Returns `session_id`, `nodes`, `provision_time_ms`, `ttl_seconds`, `expires_at`.

### `exec`

Run a shell command on a specific node. Returns stdout, stderr, and exit code. Every command has access to cluster env vars (`NODE_NAME`, `NODE_INDEX`, `NODE_IP`, `CLUSTER_HOSTS`, `CLUSTER_SSH_PUBKEY`, `CLUSTER_SSH_PRIVKEY`) for coordination across nodes.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | yes | From `provision` |
| `node` | string | yes | Node name (e.g. `"node1"`) |
| `command` | string | yes | Literal shell command to execute |

### `save`

Persist an artifact. Saved artifacts live in your org's namespace and are immediately available to future `provision` / `search` calls.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | no | `"image"` (default), `"runbook"`, or `"network"` |
| `name` | string | yes | Lowercase-and-hyphens identifier (becomes `antrieb:<name>:v1` for images, `@<namespace>/<name>` for runbooks/networks) |
| `description` | string | no | Short human description used by search (strongly recommended for runbooks — it's what the LLM sees in browse mode) |
| `session_id` | string | if `type=image` | Session containing the node to snapshot |
| `node` | string | if `type=image` | Node to save |
| `commands` | array | if `type=image` | Ordered list of successful commands executed on the node |
| `body` | string | if `type=runbook` | Markdown document (see [Runbooks](#runbooks) for structure) |
| `cidr` | string | if `type=network` | Optional `/24` CIDR like `"10.10.1.0/24"`. Auto-allocated if omitted. |
| `egress` | boolean | if `type=network` | `true` = NAT to internet, `false` = isolated |
| `dhcp` | boolean | if `type=network` | `true` = framework DHCP, `false` = bring your own |

### `search`

Browse catalogs, or fetch a specific artifact. Two modes: **browse** (keywords or no params → list of metadata) and **fetch** (`fq_name=namespace/name` → single artifact including full body, required for runbooks before applying them).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | no | `"image"` (default), `"cluster"`, `"runbook"`, or `"network"` |
| `keywords` | string | no | Full-text filter on name, description, and fq_name |
| `fq_name` | string | no | Fetch a specific runbook/network/image by fully-qualified name. For runbooks, returns the full markdown body. |
| `limit` | number | no | Max results (default 20, max 100) |

### `delete`

Destroy a cluster or decommission a saved artifact. `antrieb/*` defaults are never deletable.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | yes | `"cluster"`, `"image"`, `"runbook"`, or `"network"` |
| `name` | string | yes | `session_id` for clusters; short name or `namespace/short` fq_name for everything else |

## License

Apache 2.0
