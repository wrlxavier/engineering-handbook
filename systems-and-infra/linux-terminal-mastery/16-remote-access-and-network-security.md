# 16. Remote Access and Network Security

Secure remote administration and network boundary enforcement represent the primary operational defenses of a Linux system. Rather than treating remote shell access and firewalling as isolated tasks, modern Linux engineering integrates cryptographic transport layers (SSH protocol architecture), kernel-level packet inspection engines (`netfilter`, `nftables`, and `iptables`), and low-level audit accounting (`auditd`, PAM, and `utmp`/`wtmp`/`btmp` subsystems).

This module details the inner mechanics of SSH client optimization, transport tunneling, packet filtering lifecycles, and forensic auditing of network interactions.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Linux Userspace Boundary                        │
│   SSH Client/Daemon, Firewall CLIs (nft, iptables), Audit Daemons      │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │ Cryptographic Transport        │ Ruleset Injection
                    │ AF_INET / AF_INET6 Sockets     │ Netlink (NETLINK_NETFILTER)
                    ▼                                ▼
┌────────────────────────────────────────────────────────────────────────┐
│                       Kernel Network Subsystem                         │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                 Netfilter Hook Subsystem                       │   │
│   │   [NF_INET_PRE_ROUTING] ──► Routing Decision ──► [FORWARD]    │   │
│   │           │                                         │          │   │
│   │           ▼                                         ▼          │   │
│   │     [LOCAL_IN]                                 [POSTROUTING]   │   │
│   │           │                                         ▲          │   │
│   │           ▼                                         │          │   │
│   │     Local Process ────────► [LOCAL_OUT] ────────────┘          │   │
│   │                                                                │   │
│   │   - nftables Bytecode Virtual Machine & Expression Engine      │   │
│   │   - Connection Tracking Engine (conntrack / struct nf_conn)    │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │                                    │
│   ┌───────────────────────────────▼────────────────────────────────┐   │
│   │                 Security & Accounting Subsystems               │   │
│   │   - Linux Audit Framework (auditd / kauditd message bus)       │   │
│   │   - Socket Allocation Tables (sock_diag Netlink queries)       │   │
│   └────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Packets In / Packets Out
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Network Interface Cards (NICs)                     │
│               Physical Interfaces, VLANs, Bridges, Tunnels             │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Advanced SSH Client Architecture & `~/.ssh/config`

The Secure Shell version 2 (SSH-2) protocol (RFC 4251–4254) is structured into three distinct layers:
1. **Transport Layer Protocol (RFC 4253)**: Negotiates key exchange (KEX), verifies server host keys, establishes symmetric cipher suites and message authentication codes (MACs), and handles pseudo-packet re-keying.
2. **User Authentication Protocol (RFC 4252)**: Authenticates the client to the server via public key cryptography, keyboard-interactive PAM challenges, or GSSAPI/Kerberos tickets over the encrypted transport.
3. **Connection Protocol (RFC 4254)**: Multiplexes multiple logical, bidirectional communication streams—termed **channels**—over the single underlying encrypted transport. These channels handle interactive terminal sessions (`session`), forwarded TCP sockets (`direct-tcpip`, `forwarded-tcpip`), and X11 graphic tunnels.

```
                      SSH-2 Protocol Layer Architecture
 ┌─────────────────────────────────────────────────────────────────────────┐
 │               Connection Protocol (RFC 4254)                            │
 │  ┌───────────────────┐  ┌────────────────────┐  ┌───────────────────┐  │
 │  │ Channel 0:        │  │ Channel 1:         │  │ Channel 2:        │  │
 │  │ Interactive PTY   │  │ Local -L Forward   │  │ Dynamic SOCKS5    │  │
 │  └─────────┬─────────┘  └─────────┬──────────┘  └─────────┬─────────┘  │
 ├────────────┴──────────────────────┴───────────────────────┴─────────────┤
 │               User Authentication Protocol (RFC 4252)                   │
 │   Public Key (ed25519/rsa)  │  Host-Based  │  Keyboard-Interactive/PAM  │
 ├─────────────────────────────────────────────────────────────────────────┤
 │               Transport Layer Protocol (RFC 4253)                       │
 │  KEX: Curve25519/ECDH │ Cipher: ChaCha20-Poly1305/AES-GCM │ Host Keys   │
 └─────────────────────────────────────────────────────────────────────────┘
```

### Configuration Parsing Precedence and Evaluation Semantics

OpenSSH evaluates client configuration directives across three cascading sources:
1. Command-line flags (e.g., `ssh -o Option=Value`)
2. User-specific configuration file (`~/.ssh/config`)
3. System-wide configuration file (`/etc/ssh/ssh_config`)

OpenSSH applies a **First-Found-Wins** rule: for any given configuration keyword, the *first value encountered* in parsing order is set permanently in the runtime state machine; subsequent declarations are silently ignored. 

*Exception*: The `IdentityFile` directive is additive. Declaring multiple `IdentityFile` keys appends them to a search list attempted sequentially during public key authentication.

### Granular Host and Match Directives

The `Host` directive establishes a pattern-matching block applied against the alias provided on the command line:

```sshconfig
Host bastion-prod
    HostName 203.0.113.10
    User opsadmin
    Port 2222
```

The `Match` directive enables conditional evaluation based on operational context, dynamic command execution, local users, or environment state:

```sshconfig
# Conditional rule based on local shell execution
Match exec "ping -c 1 -W 1 10.0.0.1 >/dev/null 2>&1"
    # Route directly if on the physical internal corporate network
    Host internal-node
        HostName 10.0.0.50
        ProxyJump none

# Match on local environment user and target host criteria
Match host *.corp.example.com user !root localuser developer
    ForwardAgent no
    PasswordAuthentication no
```

Supported `Match` criteria tokens:
* `canonical`: Matches when host canonicalization is completed.
* `final`: Evaluates the block in a secondary pass after all host substitutions are settled.
* `exec "command"`: Executes a shell command using `/bin/sh`. An exit status of 0 represents a match.
* `host`: Matches against the target hostname.
* `originalhost`: Matches against the hostname string explicitly typed on the CLI before any transformation.
* `user`: Matches the target remote username.
* `localuser`: Matches the local username executing the `ssh` binary.

### Bastion Traversals: `ProxyJump` vs. `ProxyCommand`

Modern bastions (jump hosts) forward traffic from untrusted networks into isolated internal zones. Historically, administrators relied on `ProxyCommand` coupled with `netcat` (`nc`) or secondary SSH invocations:

```sshconfig
# Legacy Bastion Traversal (ProxyCommand with netcat)
Host internal-db-legacy
    HostName 10.10.10.50
    User dbadmin
    ProxyCommand ssh -W %h:%p bastion.example.com
```

Tokens used by OpenSSH token expansion:
* `%h`: Target destination host.
* `%p`: Target destination port.
* `%r`: Target remote username.
* `%u`: Local username.

The `ProxyJump` directive automates end-to-end multi-hop tunneling using internal `direct-tcpip` connection channels, eliminating shell pipeline overhead and intermediate subprocess instantiation. Multiple hops are chained with commas:

```sshconfig
# Modern Multi-Hop Bastion Traversal
Host internal-db
    HostName 10.10.10.50
    User dbadmin
    Port 22
    IdentityFile ~/.ssh/id_ed25519_db
    # Traverses jump-dmz (hop 1), then jump-secure (hop 2)
    ProxyJump jump-dmz.example.com:22,jump-secure.internal:2222
```

```
                     ProxyJump Multi-Hop Architecture
 ┌──────────────┐         ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
 │ Local Client │ ──────► │ Jump Host 1  │ ──────► │ Jump Host 2  │ ──────► │ Target Host  │
 │  Workstation │ (SSH-1) │  (jump-dmz)  │ (SSH-2) │(jump-secure) │ (SSH-3) │(internal-db) │
 └──────────────┘         └──────────────┘         └──────────────┘         └──────────────┘
  │                                                                          ▲
  └──────────────────────── End-to-End Cryptography ─────────────────────────┘
        (Client authenticates directly to Target; Jump hosts see only ciphertext)
```

`ProxyJump` preserves end-to-end cryptographic confidentiality. The intermediate jump hosts act strictly as Layer 4 TCP stream relays using `direct-tcpip` SSH channels. Neither jump host possesses the private keys or session keys required to inspect, decrypt, or tamper with the payload traversing to `internal-db`.

### High-Performance Multiplexing: `ControlMaster`, `ControlPath`, and `ControlPersist`

Establishing an SSH connection requires an asymmetric key exchange, cipher parameter negotiation, user authentication, and PTY allocations. This introduces a 150–500ms latency penalty per connection. 

**Connection Multiplexing** reuses an existing authenticated transport connection for subsequent concurrent sessions:

```sshconfig
Host *
    ControlMaster auto
    ControlPath ~/.ssh/control-%C
    ControlPersist 10m
```

* `ControlMaster auto`: Instructs the OpenSSH client to listen on a local UNIX domain control socket specified by `ControlPath`. If the socket does not exist, the client becomes the "Master" process, spawns the transport connection, and provisions the socket. Subsequent client invocations detect the socket, register as "Slaves," and attach new logical channels directly over the established transport layer without repeating key exchanges or authentication handshakes.
* `ControlPath ~/.ssh/control-%C`: Defines the filesystem path for the UNIX domain socket. The `%C` token generates a 40-character cryptographic hash (SHA-1) of the tuple:
  $$\text{Hash} = H(\text{Local Hostname} \parallel \text{Target Host} \parallel \text{Port} \parallel \text{Username})$$
  This prevents pathname collisions and avoids exceeding the Linux kernel's `sun_path` limit of 108 bytes for `sockaddr_un` UNIX domain sockets.
* `ControlPersist 10m`: Forks the master SSH process into the background upon closing the primary interactive terminal shell. The master connection stays dormant, holding the encrypted TCP pipe open for 10 minutes. If a new slave process connects within that window, the timer resets; if the timer expires with zero active channels, the master process terminates the TCP session cleanly.

#### Checking and Terminating Multiplexed Sockets
```bash
# Verify the master multiplexing process status for a specific host
ssh -O check bastion-prod

# Gracefully request the master process to exit once active slaves finish
ssh -O stop bastion-prod

# Immediately terminate the master connection, dropping all concurrent sessions
ssh -O exit bastion-prod
```

### Identity and Authentication Hardening

#### Dedicated Keys and Identity Isolation
By default, the SSH client queries every private key loaded into `ssh-agent`, followed by standard disk locations (`~/.ssh/id_rsa`, `~/.ssh/id_ed25519`). If an enterprise server limits authentication attempts (`MaxAuthTries 3`) and the agent offers three non-matching keys first, the server drops the connection before the correct key is ever presented:

```text
Received disconnect from 192.0.2.1 port 22:2: Too many authentication failures
```

To prevent key probing and authentication lockout, bind specific keys to hosts and enforce strict identity evaluation:

```sshconfig
Host gitlab.internal
    HostName git.internal.example.com
    User git
    IdentityFile ~/.ssh/keys/id_ed25519_gitlab
    # Disallow all agent keys and fallback defaults; offer ONLY the specified IdentityFile
    IdentitiesOnly yes
    # Automatically add this key to ssh-agent if unlocked
    AddKeysToAgent yes
```

#### The Agent Forwarding Security Trap (`ForwardAgent`)
Enabling `ForwardAgent yes` creates an environment variable (`$SSH_AUTH_SOCK`) on the remote server pointing to a proxied UNIX domain socket linked back to your local `ssh-agent`. 

**Critical Vulnerability**: Anyone with `root` privileges on that remote server can access that UNIX socket and sign arbitrary authentication requests with your local private keys for as long as your session remains open:

```bash
# Malicious administrator or rootkit on remote server:
sudo -u compromised_user SSH_AUTH_SOCK=/tmp/ssh-XXXXXX/agent.1234 ssh production-vault
```

**Rule**: Never enable `ForwardAgent yes` globally. Use `ProxyJump` instead, which eliminates the need for agent forwarding entirely. If agent forwarding is mandatory for legacy tasks, use SSH Certificate authentication with restrictive cryptographic constraints, or use the SSH confirmation flag (`ssh-add -c`), which requires manual terminal confirmation on the client machine each time the agent signs a payload.

#### FIDO2 / U2F Hardware Token Authentication
Modern OpenSSH implementations natively support physical hardware security keys (YubiKey, Nitrokey) using the `ed25519-sk` and `ecdsa-sk` key types:

```bash
# Generate a hardware-backed security key requiring physical user presence (touch)
ssh-keygen -t ed25519-sk -O resident -O verify-required -f ~/.ssh/id_ed25519_sk
```

Configuration directive to enforce hardware token authentication:
```sshconfig
Host secure-infrastructure-*
    IdentityFile ~/.ssh/id_ed25519_sk
    # Require PIN verification on the physical token
    SecurityKeyProvider internal
```

### Modern Cryptographic Suites and Connection Keep-Alive

To comply with modern security standards, legacy algorithms must be explicitly eliminated from the client configuration:

```sshconfig
Host *
    # Enforce modern Key Exchange algorithms (Reject DH-group1-sha1, DH-group14-sha1)
    KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

    # Enforce authenticated encryption ciphers (Reject CBC modes, 3DES, blowfish, RC4)
    Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com

    # Enforce SHA-2 MACs for Encrypt-then-MAC (EtM) configurations
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

    # Restrict acceptable server public key types
    HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256

    # Cryptographic keep-alive to prevent NAT firewall state-table eviction
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

* `ServerAliveInterval 30`: Sends an encrypted, authenticated SSH protocol-level probe request (`SSH2_MSG_GLOBAL_REQUEST`) through the transport channel every 30 seconds of link inactivity. This forces stateful intermediate firewalls to refresh their NAT and tracking timers.
* `ServerAliveCountMax 3`: If three consecutive probe requests fail to receive a response, the client drops the broken socket, returning control to the local shell instead of hanging indefinitely.

---

## 2. Port Forwarding, Tunneling, and Encrypted Transport

SSH port forwarding translates TCP stream operations into encrypted SSH channel packets via the `direct-tcpip` or `forwarded-tcpip` channel protocol definitions.

```
                  RFC 4254 Channel Multiplexing Logic
   Client Machine                                        SSH Server
 ┌────────────────┐                                   ┌────────────────┐
 │ App (e.g. psql)│                                   │ Target Service │
 └───────┬────────┘                                   └────────▲───────┘
         │ Local Stream                                        │ Remote Stream
 ┌───────▼────────┐      Encrypted SSH Transport      ┌────────┴───────┐
 │   SSH Client   │ ════════════════════════════════► │   SSH Daemon   │
 │ (direct-tcpip) │    [Type=90: SSH_MSG_CHANNEL_OPEN]│ (Connects to   │
 └────────────────┘                                   │ target host:pt)│
                                                      └────────────────┘
```

### Local Port Forwarding (`-L`)

Local port forwarding instructs the local SSH client to bind to a local loopback or interface port. When a local application establishes a TCP connection to this port, the SSH client wraps the byte stream into `SSH_MSG_CHANNEL_DATA` frames, routes them through the encrypted SSH tunnel to the remote SSH server, and instructs the remote server to open a direct TCP socket to the designated destination address and port.

#### Syntax Variations
```bash
ssh -L [bind_address:]local_port:destination_host:destination_port user@ssh_server
```

```
                          Local Port Forwarding (-L)
 ┌────────────────────────────────────────────────────────┐
 │ Client Host                                            │
 │  ┌──────────────┐     TCP Connect     ┌──────────────┐ │
 │  │ Client App   │ ──────────────────► │  Local Port  │ │
 │  │ (Local Node) │                     │   127.0.0.1  │ │
 │  └──────────────┘                     │    :5432     │ │
 │                                       └──────┬───────┘ │
 └──────────────────────────────────────────────┼─────────┘
                                                │ Encrypted
                                                │ SSH Tunnel
                                                ▼
 ┌────────────────────────────────────────────────────────┐
 │ Remote SSH Server                                      │
 │  ┌──────────────┐                     ┌──────────────┐ │
 │  │ Target DB    │ ◄────────────────── │  sshd Engine │ │
 │  │ 10.0.1.25    │      Plain TCP      │ (Remote Node)│ │
 │  │ :5432        │                     └──────────────┘ │
 │  └──────────────┘                                      │
 └────────────────────────────────────────────────────────┘
```

#### Production Examples
1. **Accessing an isolated remote database via an edge bastion**:
   ```bash
   # Bind local 127.0.0.1:5433 -> Route via bastion -> Target 10.0.1.25:5432
   ssh -N -L 5433:10.0.1.25:5432 opsadmin@bastion.example.com
   ```
   Now point the local database client to the local tunnel endpoint:
   ```bash
   psql -h 127.0.0.1 -p 5433 -U postgres production_db
   ```

2. **Binding to all local interfaces to expose the remote service to an internal LAN**:
   ```bash
   # Binds to 0.0.0.0, allowing other hosts on the workstation's LAN to access it
   ssh -N -L 0.0.0.0:8080:internal-jenkins.corp.local:80 opsadmin@bastion.example.com
   ```

### Remote / Reverse Port Forwarding (`-R`)

Remote port forwarding instructs the remote SSH daemon to bind a listening port on the server. When connections arrive at the server's port, the traffic is tunneled through the established SSH connection back to the client machine, which then connects to a service on its local machine or local network.

#### Syntax Variations
```bash
ssh -R [remote_bind_address:]remote_port:destination_host:destination_port user@ssh_server
```

```
                         Remote Port Forwarding (-R)
 ┌────────────────────────────────────────────────────────┐
 │ Remote Server (Public / Intermediate)                  │
 │  ┌──────────────┐     TCP Connect     ┌──────────────┐ │
 │  │ Inbound Web  │ ──────────────────► │ Remote Port  │ │
 │  │ Traffic      │                     │ 0.0.0.0:8080 │ │
 │  └──────────────┘                     └──────┬───────┘ │
 └──────────────────────────────────────────────┼─────────┘
                                                │ Encrypted
                                                │ SSH Tunnel
                                                ▼
 ┌────────────────────────────────────────────────────────┐
 │ Client Host (Local / Behind NAT)                       │
 │  ┌──────────────┐                     ┌──────────────┐ │
 │  │ Dev Server   │ ◄────────────────── │  SSH Client  │ │
 │  │ 127.0.0.1    │      Plain TCP      │ (Local Node) │ │
 │  │ :3000        │                     └──────────────┘ │
 │  └──────────────┘                                      │
 └────────────────────────────────────────────────────────┘
```

#### Prerequisites on the Remote Server (`sshd_config`)
By default, the SSH daemon forces remote forward bindings to the loopback interface (`127.0.0.1`), preventing external nodes from accessing the exposed port. To permit remote forwards to bind to wildcard addresses, modify `/etc/ssh/sshd_config` on the remote server:

```text
# Permit remote forwarded ports to bind to external-facing interfaces
GatewayPorts yes
```
* `GatewayPorts no`: Prevents binding to non-loopback addresses (Default).
* `GatewayPorts yes`: Forces all remote port forwards to bind to wildcard addresses (`0.0.0.0` / `::`).
* `GatewayPorts clientspecified`: Honors the explicit bind address passed by the client.

#### Production Example
Expose a local microservice running on workstation port 3000 to a publicly accessible staging server:
```bash
ssh -N -R 0.0.0.0:8080:127.0.0.1:3000 deploy@staging.example.com
```

### Dynamic Port Forwarding (`-D`): SOCKS5 Proxying

While `-L` and `-R` forward strictly to static target `(IP, port)` pairs, Dynamic Forwarding turns the SSH client into an application-level SOCKS4/SOCKS5 proxy server:

```bash
ssh -N -D 127.0.0.1:1080 user@egress-proxy.example.com
```

#### Protocol Handling Mechanics
1. The local SSH client binds to TCP port 1080.
2. An application configured to use a SOCKS5 proxy (e.g., a web browser or `curl`) sends a proxy negotiation request.
3. The application instructs the SOCKS5 proxy to establish a connection to a specific remote destination (e.g., `internal-monitoring.corp.priv:443`).
4. **DNS Resolution Traversal**: With SOCKS5, the client passes domain names un-resolved directly to the proxy. The remote SSH server performs the DNS resolution query from its own network vantage point, preventing internal hostnames from leaking onto public resolvers and bypassing local DNS poisoning.

```bash
# Query an internal endpoint using the dynamic SOCKS5 tunnel with remote DNS resolution
curl --socks5-hostname 127.0.0.1:1080 https://vault.internal.corp.priv/v1/sys/health
```

### Layer 3 VPN Tunneling (`-w`)

OpenSSH can configure virtual Layer 3 point-to-point IP tunnels using Linux `tun` network interfaces. This encapsulates raw IP packets directly inside SSH transport channels without requiring OpenVPN or WireGuard.

#### Prerequisites (`sshd_config`)
```text
PermitTunnel yes
```

#### Tunnel Configuration Steps
```bash
# Step 1: Open a Layer 3 tunnel allocating tun0 on client and tun1 on server
sudo ssh -w 0:1 -N root@gateway.example.com

# Step 2: Configure point-to-point IP addressing on the client
sudo ip addr add 10.200.0.2/30 dev tun0
sudo ip link set dev tun0 up

# Step 3: Configure point-to-point IP addressing on the server
sudo ip addr add 10.200.0.1/30 dev tun1
sudo ip link set dev tun1 up

# Step 4: Route private corporate subnet over the new tun0 interface on the client
sudo ip route add 192.168.100.0/24 via 10.200.0.1 dev tun0
```

### Background Execution Flags and Runtime Control

When scripting SSH tunnels or embedding them in system startup services, decouple the SSH process from the active shell environment using these flags:

* `-N`: Do not execute remote commands. Allocates no PTY and spawns no remote shell; dedicated purely to port forwarding.
* `-f`: Requests the SSH client to fork into the background immediately before command execution. It prompts for passphrases/passwords on the foreground terminal, and background-forks as soon as authentication succeeds.
* `-T`: Disables pseudo-terminal (PTY) allocation on the server side.
* `-q`: Quiet mode; suppresses warnings and diagnostic messages.

```bash
# Launch a background, detached, persistent forward
ssh -f -N -T -L 6379:redis.internal:6379 opsadmin@bastion.example.com
```

#### The Escape Character Interface
During an interactive SSH session, typing the escape character `~` (at the beginning of a fresh line) exposes the SSH client control CLI:

* `~.` : Terminate the connection immediately.
* `~#` : List all active multiplexed channels, forwards, and file descriptors.
* `~C` : Open the dynamic command console. Allows adding/removing forwards without terminating the active shell:
  ```text
  ssh> -L 8080:localhost:80
  Forwarding port.
  ssh> -KL 8080
  Canceled forwarding.
  ```
* `~?` : Display the escape menu help reference.

---

## 3. Linux Packet Filtering: `netfilter`, `iptables`, and `nftables`

The Linux kernel processes all network traffic using the **Netfilter** framework. Netfilter exposes five architectural hooks inside the kernel's Layer 3/Layer 4 execution paths, allowing kernel modules to inspect, modify, redirect, or drop packets.

```
                           Netfilter Kernel Hook Points
                                 [Ingress Wire]
                                       │
                                       ▼
                             ┌───────────────────┐
                             │ NF_INET_PRE_ROUTE │
                             └─────────┬─────────┘
                                       │
                              [Routing Decision]
                                ┌──────┴──────┐
           Local Packet         │             │ Forwarded Packet
        ┌───────────────────────┘             └───────────────────────┐
        ▼                                                             ▼
┌───────────────┐                                             ┌───────────────┐
│ NF_INET_LOCAL │                                             │NF_INET_FORWARD│
│     _IN       │                                             └───────┬───────┘
└───────┬───────┘                                                     │
        ▼                                                             │
  [Local Process]                                                     │
        │                                                             │
        ▼                                                             │
┌───────────────┐                                                     │
│ NF_INET_LOCAL │                                                     │
│     _OUT      │                                                     │
└───────┬───────┘                                                     │
        │                                                             │
        └───────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
                       [Routing Decision]
                                │
                                ▼
                      ┌───────────────────┐
                      │NF_INET_POST_ROUTE │
                      └─────────┬─────────┘
                                │
                                ▼
                          [Egress Wire]
```

### The Five Netfilter Hooks
1. `NF_INET_PRE_ROUTING`: Intercepts packets immediately after the network driver processes the Layer 2 frame and wraps it in a `sk_buff`, before the kernel consults the Forwarding Information Base (FIB). Used for Destination NAT (DNAT) and early packet dropping.
2. `NF_INET_LOCAL_IN`: Intercepts packets whose destination IP address matches an address assigned to a local interface. All incoming traffic directed at local userspace daemons passes through this hook.
3. `NF_INET_FORWARD`: Intercepts packets destined for an external host that must be routed across the machine. Triggered only when the kernel has `net.ipv4.ip_forward = 1`.
4. `NF_INET_LOCAL_OUT`: Intercepts packets generated by local processes on the system immediately upon hitting the IP stack, before the kernel evaluates Layer 3 routing for outbound egress.
5. `NF_INET_POST_ROUTING`: Intercepts packets after all routing and forwarding operations are resolved, immediately prior to passing the buffer to the interface queue. Used for Source NAT (SNAT) and IP Masquerading.

### The Connection Tracking Engine (`conntrack`)

The Netfilter Connection Tracking subsystem (`nf_conntrack`) monitors the state of Layer 4 communication sessions. Every packet traversing the hooks is matched against an in-memory hash table containing state structures (`struct nf_conn`).

#### The Conntrack Packet Classifications
* `NEW`: The packet initiates a new connection (e.g., a TCP SYN segment). No bidirectional traffic has been observed yet.
* `ESTABLISHED`: The packet belongs to an active, acknowledged connection that has seen bidirectional traffic (e.g., TCP SYN-ACK received and subsequent data transfer).
* `RELATED`: The packet initiates a new separate connection directly associated with an already established connection. Common examples include FTP data channels (passive mode) or ICMP Type 3 (Destination Unreachable) error notifications triggered by an active UDP/TCP flow.
* `INVALID`: The packet could not be identified or matched against any tracked session. This occurs with out-of-window TCP packets, corrupted flags, or malformed protocol states. In production firewalls, `INVALID` traffic is dropped immediately.
* `UNTRACKED`: Packets explicitly flagged to bypass connection tracking (via the `NOTRACK` target in `iptables` raw table or `notrack` statement in `nftables`).

### Legacy `iptables` Architecture

Historically, userspace administration of Netfilter relied on `iptables` (IPv4), `ip6tables` (IPv6), `arptables`, and `ebtables`. The architecture segments logic into fixed **Tables**, which contain fixed **Chains**, which contain sequentially evaluated **Rules**.

#### Tables and Functional Roles
* `raw`: Bypasses connection tracking (`PREROUTING`, `OUTPUT`).
* `mangle`: Alters packet IP header fields like TOS, TTL, or MARK (`All Hooks`).
* `nat`: Translates source/destination addresses (`PREROUTING`, `INPUT`, `OUTPUT`, `POSTROUTING`).
* `filter`: Primary security policy table; permits or blocks packets (`INPUT`, `FORWARD`, `OUTPUT`).
* `security`: Applies Mandatory Access Control (MAC) security marks via SELinux (`INPUT`, `FORWARD`, `OUTPUT`).

#### Inherent Architectural Limitations of `iptables`
1. **Sequential Evaluation Overhead**: Rules within a chain are evaluated linearly. A table with 5,000 rules incurs $\mathcal{O}(N)$ worst-case packet evaluation latency.
2. **Lack of Atomic Incremental Updates**: Modifying a single rule requires copying the entire table from kernel space to userspace, editing the array, and writing the entire multimegabyte ruleset back into the kernel. This induces race conditions and CPU consumption on high-churn hosts (e.g., Kubernetes nodes).
3. **Protocol Fragmentation**: IPv4 and IPv6 require separate binaries and rulesets (`iptables` vs `ip6tables`), forcing administrators to duplicate dual-stack configurations.
4. **Monolithic Module Design**: Every match criteria (e.g., `-m state`, `-m multiport`, `-m string`) compiles to a dedicated kernel extension, increasing memory footprint and context shifts.

### Modern `nftables` Architecture

`nftables` completely replaces `iptables`, `ip6tables`, `ebtables`, and `arptables`. It operates via a lightweight **Virtual Machine (VM)** running inside the kernel that executes compact bytecode instructions.

```
                      nftables Execution Hierarchy
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Userspace: `nft` Command-Line Tool / libnftables                       │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │ Netlink Bytecode (NETLINK_NETFILTER)
                                     ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Kernel: nftables In-Kernel Virtual Machine                             │
 │                                                                        │
 │   Address Family: `inet` (Unified IPv4 and IPv6 Evaluation)           │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │ Table: filter                                                  │   │
 │   │   ┌────────────────────────────────────────────────────────┐   │   │
 │   │   │ Chain: input (Hook: NF_INET_LOCAL_IN, Priority: 0)     │   │   │
 │   │   │   - Rule 1: ct state established,related accept        │   │   │
 │   │   │   - Rule 2: iif "lo" accept                            │   │   │
 │   │   │   - Rule 3: tcp dport { 22, 80, 443 } accept           │   │   │
 │   │   │   - Rule 4: drop                                       │   │   │
 │   │   └────────────────────────────────────────────────────────┘   │   │
 │   └────────────────────────────────────────────────────────────────┘   │
 └────────────────────────────────────────────────────────────────────────┘
```

#### Core Enhancements in `nftables`
* **Unified Address Families**: The `inet` family handles both IPv4 and IPv6 traffic inside a single chain, avoiding rule duplication.
* **Fully Dynamic Framework**: Tables and chains do not exist by default. The administrator defines only the chains hooked to specific kernel points with explicit execution priorities.
* **Sets and Dictionaries**: High-performance set operations use Red-Black trees and hash tables directly in kernel memory. Lookup complexity drops from linear $\mathcal{O}(N)$ scanning to $\mathcal{O}(1)$ or $\mathcal{O}(\log N)$, allowing sets containing hundreds of thousands of IP addresses to evaluate instantly.
* **True Atomic Commits**: Rulesets are parsed and submitted via Netlink as a single transactional operation. The kernel either applies the entire ruleset or rejects it with zero intermediate state inconsistency.

### Production Firewall Implementation: Hardened Dual-Stack Firewall

The following configuration implements an enterprise-grade, stateful host firewall using native `nftables`. It defaults to dropping all incoming traffic, enforces stateful tracking, isolates loopback interfaces, drops malformed TCP states, mitigates SSH brute-force attempts via rate-limited sets, and manages port forwards.

Create the file `/etc/nftables.conf`:

```nftables
#!/usr/sbin/nft -f

# Flush existing tables to guarantee atomic, clean ruleset evaluation
flush ruleset

# Define variables for maintainability
define WAN_IFACE = "eth0"
define MGMT_IPS = { 192.168.1.100, 10.0.0.0/24 }

# Unified dual-stack table for IPv4 and IPv6
table inet host_firewall {

    # Set: IP address blocklist for malicious actors
    set dynamic_denylist {
        type ipv4_addr
        flags timeout
        size 65535
    }

    # Set: Track connection rate for SSH brute-force defense
    set ssh_flood_meter {
        type ipv4_addr
        flags dynamic
        timeout 1m
        size 65535
    }

    # Ingress Filter Chain: Evaluated at NF_INET_LOCAL_IN
    chain inbound_traffic {
        # Hook into incoming packets destined for local processes; default DROP
        type filter hook input priority filter; policy drop;

        # 1. Drop packets explicitly listed in the dynamic denylist
        ip saddr @dynamic_denylist drop

        # 2. Connection Tracking: Permit established and related return traffic
        ct state established,related accept

        # 3. Drop invalid connection states immediately
        ct state invalid drop

        # 4. Loopback isolation: Permit all internal system process communication
        iifname "lo" accept

        # 5. ICMP / ICMPv6 Rate Limiting (Permit essential path discovery and ping)
        ip protocol icmp icmp type { echo-request, destination-unreachable, time-exceeded } \
            limit rate 5/second burst 10 packets accept
        
        ip6 nexthdr icmpv6 icmpv6 type { echo-request, destination-unreachable, packet-too-big, \
            time-exceeded, parameter-problem, nd-router-solicit, nd-router-advert, \
            nd-neighbor-solicit, nd-neighbor-advert } \
            limit rate 5/second burst 10 packets accept

        # 6. Drop anomalous TCP flag combinations (Port scan & evasion signatures)
        tcp flags & (fin|syn|rst|psh|ack|urg) == 0 drop                    # Null scan
        tcp flags & (fin|syn|rst|psh|ack|urg) == fin|syn|rst|psh|ack|urg drop # Xmas scan
        tcp flags & (syn|rst) == syn|rst drop                             # SYN/RST invalid
        tcp flags & (fin|syn) == fin|syn drop                             # FIN/SYN invalid

        # 7. SSH Ingress with Dynamic Rate-Limiting Protection (Max 3 new handshakes per minute)
        tcp dport 22 ct state new meter ssh_flood_meter { ip saddr ct count over 3 } \
            add @dynamic_denylist { ip saddr timeout 10m } \
            log prefix "FIREWALL_SSH_ABUSE: " drop

        tcp dport 22 ct state new accept

        # 8. Expose Standard Public Web Services (HTTP / HTTPS)
        tcp dport { 80, 443 } ct state new accept

        # 9. Administrative Port Access: Restricted to specific management IPs
        ip saddr $MGMT_IPS tcp dport 9090 ct state new accept

        # 10. Audit Logging: Record remaining dropped attempts before policy execution
        limit rate 3/minute burst 5 packets log prefix "FIREWALL_DEFAULT_DROP: " flags all
    }

    # Forward Filter Chain: Evaluated at NF_INET_FORWARD
    chain forward_traffic {
        # Host does not act as an IP router; DROP all transit packets
        type filter hook forward priority filter; policy drop;
    }

    # Egress Filter Chain: Evaluated at NF_INET_LOCAL_OUT
    chain outbound_traffic {
        # Permit all outbound traffic originating from local host daemons
        type filter hook output priority filter; policy accept;
    }
}

# NAT Table for Local Service Redirection / DNAT
table ip nat_routing {
    chain prerouting_nat {
        type nat hook prerouting priority dstnat; policy accept;

        # Redirect incoming traffic on WAN interface port 8080 to internal service on 80
        iifname "eth0" tcp dport 8080 counter redirect to :80
    }

    chain postrouting_nat {
        type nat hook postrouting priority srcnat; policy accept;

        # Outbound masquerade (if host acts as dynamic NAT gateway)
        # oifname "eth0" masquerade
    }
}
```

#### Activating and Testing the Ruleset
```bash
# Check syntax of the configuration file without loading it into the kernel
sudo nft -c -f /etc/nftables.conf

# Atomically apply the configuration into the active Netfilter engine
sudo nft -f /etc/nftables.conf

# Inspect active tables, chains, and rules with hit counters
sudo nft list ruleset

# Real-time monitoring of packet events traversing the engine
sudo nft monitor
```

---

## 4. Auditing Inbound Connections and System Authentication History

Defending a Linux system requires active auditing of connection states and historical inspection of authentication events. Linux isolates these records across three distinct architectural components:
1. Low-level C-library system accounting databases (`utmp`, `wtmp`, `btmp`).
2. Pluggable Authentication Modules (PAM) and system logs (`/var/log/auth.log`, `/var/log/secure`, `journald`).
3. The Linux Audit Subsystem (`auditd` and `kauditd`).

```
                    Linux Security Auditing Telemetry
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      Authentication Sources                            │
 │          sshd, /bin/login, su, sudo, systemd-logind                    │
 └───────────────────┬────────────────────────────────┬───────────────────┘
                     │ C-Lib Accounting               │ PAM Framework
                     ▼                                ▼
 ┌─────────────────────────────────────┐  ┌───────────────────────────────┐
 │ Binary Accounting Databases         │  │ System Logging Framework      │
 │  - /run/utmp  (Active Sessions)     │  │  - /var/log/auth.log (Debian) │
 │  - /var/log/wtmp (Historical Logins)│  │  - /var/log/secure (RHEL)     │
 │  - /var/log/btmp (Failed Auth)      │  │  - systemd-journald (Binary)  │
 └───────────────────┬─────────────────┘  └───────────────┬───────────────┘
                     │ read by                            │ parsed by
                     ▼                                    ▼
 ┌─────────────────────────────────────┐  ┌───────────────────────────────┐
 │ CLI Tools: who, w, last, lastb      │  │ journalctl, fail2ban          │
 └─────────────────────────────────────┘  └───────────────────────────────┘
                     │                                    │
                     └──────────────────┬─────────────────┘
                                        ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Kernel Audit Framework (kauditd) ──► auditd ──► /var/log/audit/audit.log│
 │ Traps: sys_enter_execve, openat, file watches on ~/.ssh/authorized_keys│
 └────────────────────────────────────────────────────────────────────────┘
```

### The System Accounting Databases: UTMP, WTMP, and BTMP

The Linux C library (`glibc`) maintains fixed-structure binary logs documenting user logins, logouts, system boots, and runlevel transitions. These are defined by `struct utmp` in `<utmp.h>`:

```c
struct utmp {
    short   ut_type;              /* Type of record (USER_PROCESS, DEAD_PROCESS) */
    pid_t   ut_pid;               /* Process ID of login process */
    char    ut_line[UT_LINESIZE]; /* Device name of tty - "/dev/pts/1" */
    char    ut_id[4];             /* Terminal name suffix, or inittab ID */
    char    ut_user[UT_NAMESIZE]; /* Username */
    char    ut_host[UT_HOSTSIZE]; /* Hostname for remote login */
    struct  exit_status ut_exit;  /* Exit status of a process marked DEAD_PROCESS */
    int32_t ut_session;           /* Session ID */
    struct {
        int32_t tv_sec;           /* Seconds elapsed since UNIX epoch */
        int32_t tv_usec;          /* Microseconds elapsed */
    } ut_tv;                      /* Time timestamp */
    int32_t ut_addr_v6[4];        /* IPv4/IPv6 address of remote host */
    char    __unused[20];
};
```

#### Database Roles
* `/run/utmp` (or `/var/run/utmp`): Tracks **currently active** login sessions, PTY allocations, and system state. Evaluated by tools like `who`, `w`, and `uptime`. Because it maps live system state, it exists inside a `tmpfs` RAM-backed filesystem and does not survive reboots.
* `/var/log/wtmp`: Tracks the **historical archive** of successful logins, logouts, terminal disconnections, and system reboots. Appended sequentially each time a `utmp` entry transitions to `DEAD_PROCESS`. Evaluated via the `last` utility.
* `/var/log/btmp`: Tracks **failed authentication attempts** (bad logins). Appended when invalid passwords, unknown users, or malformed authentication tokens are offered. Accessible strictly by `root` due to sensitive credentials occasionally mistyped into the username field. Evaluated via `lastb`.

#### CLI Forensic Inspections
```bash
# Display currently logged-in users, terminal paths, login times, and client IPs
who -a

# Display active users with idle time and active foreground process execution
w

# Query successful authentication history across the lifetime of wtmp
# -i forces numeric IP display rather than slow/failing DNS PTR resolutions
last -i -F -x

# Query failed authentication attempts recorded in btmp (Brute-force audit)
sudo lastb -i -F -n 50

# Inspect the most recent login timestamp for all system accounts
lastlog
```

### Real-Time Socket and Connection Auditing

Network visibility requires inspecting active transport layer states using the kernel `sock_diag` Netlink subsystem via `ss`:

```bash
# Audit all established incoming SSH connections, showing numeric IPs and PIDs
sudo ss -tnpe state established '( sport = :22 or sport = :2222 )'
```

Output:
```text
Recv-Q  Send-Q   Local Address:Port      Peer Address:Port   Process                                     
0       0        192.168.1.50:22        203.0.113.85:51240   users:(("sshd",pid=14205,fd=4)) ino:45821 sk:1001
```

* `ino:45821`: The socket inode within VFS. Cross-reference against `/proc/[pid]/fd/` to map the exact descriptor.
* `sk:1001`: Kernel internal socket identifier.

Query the Netfilter connection tracking state table directly using the `conntrack` utility:
```bash
# Display all currently tracked TCP sessions passing through the kernel
sudo conntrack -L -p tcp

# Stream connection tracking state transitions in real time (Lifecycle debugging)
sudo conntrack -E -e NEW,DESTROY
```

### Authentication Auditing via PAM and System Journals

The Pluggable Authentication Modules (PAM) subsystem processes authentication challenges on modern Linux systems. Every SSH login event emits structured log entries to `systemd-journald` and flat-file mirrors (`/var/log/auth.log` on Debian/Ubuntu, `/var/log/secure` on RHEL/Fedora).

#### Identifying Attack Signatures in Logs

##### Failed Password Attempt
```text
sshd[18402]: Failed password for invalid user admin from 198.51.100.42 port 43210 ssh2
```
*Indicator*: Automated dictionary attack probing common unprivileged administrative usernames.

##### Public Key Rejected Due to File Permissions
```text
sshd[18410]: Authentication refused: bad ownership or modes for file /home/alice/.ssh/authorized_keys
```
*Root Cause*: OpenSSH enforces strict permission checking (`StrictModes yes`). If `~/.ssh` has permissions wider than `0700` or `~/.ssh/authorized_keys` has permissions wider than `0600`, authentication is rejected to prevent local file hijacking.

##### Successful Public Key Authentication
```text
sshd[18420]: Accepted publickey for alice from 203.0.113.15 port 58212 ssh2: ED25519 SHA256:d3b07384d113edec49eaa6238ad5ff00
```
*Indicator*: Cryptographic fingerprint tracking. The SHA256 hash maps directly to the specific public key stored inside `~alice/.ssh/authorized_keys`.

#### Structured Forensic Queries with `journalctl`
```bash
# Query all authentication events generated by sshd since a specific timestamp
journalctl -u ssh -u sshd --since "2 hours ago" -o verbose

# Extract all failed authentication attempts in structured JSON format
journalctl -u sshd _COMM=sshd -g "Failed password" -o json-pretty

# Monitor live authentication events in real time
journalctl -u sshd -f
```

### Deep System Call Auditing via `auditd`

While logs and `utmp` track high-level events, a sophisticated attacker can modify application binaries or wipe logs. The **Linux Audit Framework** operates directly inside the system call entry points of the kernel (`kauditd`), logging events to `/var/log/audit/audit.log` before userspace processes can intercept them.

```
                   Linux Audit Subsystem Architecture
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Userspace Workload: SSH Client / Attacker Commands                     │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │ Invokes system call (e.g. execve)
                                     ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Kernel Core: System Call Dispatcher                                    │
 │                                                                        │
 │   Audit Filter Engine (audit_filter_syscall)                           │
 │   - Matches syscalls against active rules loaded via auditctl          │
 │   - Captures credentials: AUID (Login UID), EUID, CWD, Comm, Syscall  │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │ Netlink Bus (NETLINK_AUDIT)
                                     ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Userspace Audit Daemon: auditd ──► Disk Storage: /var/log/audit/audit.log│
 └────────────────────────────────────────────────────────────────────────┘
```

#### The Immutable Audit Identity: `auid`
When a user authenticates, PAM assigns them an **Audit User ID (`auid`)** (or Login UID) via `/proc/self/loginuid`. Unlike the Effective UID (`euid`), which changes when executing `sudo` or SUID binaries, **the kernel locks `auid` permanently**. Even if a user invokes `sudo su -` to become `root` (EUID 0), every system call they trigger retains their original `auid` (e.g., `auid=1001`), providing accountability across privilege escalations.

#### Production Audit Rules for Remote Access Security
Add the following rules to `/etc/audit/rules.d/99-remote-security.rules`:

```text
# Clear all existing audit rules to ensure a clean state
-D

# Set kernel audit buffer size (prevent packet drop during high audit churn)
-b 8192

# Failure mode: 1 = printk warning, 2 = kernel panic (for ultra-secure enclaves)
-f 1

# Monitor changes to the SSH daemon configuration
-w /etc/ssh/sshd_config -p wa -k sshd_config_changes
-w /etc/ssh/sshd_config.d/ -p wa -k sshd_config_changes

# Monitor user authorized_keys and identities across the filesystem
-w /etc/pam.d/ -p wa -k pam_modifications
-w /etc/security/ -p wa -k pam_security_modifications

# Audit modifications to system authentication databases
-w /var/log/wtmp -p wa -k session_tampering
-w /var/log/btmp -p wa -k session_tampering
-w /run/utmp -p wa -k session_tampering

# Monitor the execution of network socket creation from unprivileged users
-a always,exit -F arch=b64 -S socket -F a0=2 -F success=1 -k network_socket_v4
-a always,exit -F arch=b64 -S socket -F a0=10 -F success=1 -k network_socket_v6

# Lock the audit ruleset (prevents modifications without reboot)
-e 2
```

#### Parsing and Interrogating Audit Logs
The raw audit log format is dense and difficult to read. Parse it using the dedicated audit toolsuite:

```bash
# Query audit events triggered by SSH configuration alterations
sudo ausearch -k sshd_config_changes --format text

# Search authentication attempts matching a specific Audit User ID
sudo ausearch -m USER_AUTH,USER_LOGIN --loginuid 1001

# Generate a high-level summary report of all system authentication events
sudo aureport --auth --summary

# Generate an execution report detailing commands run by authenticated sessions
sudo aureport --executable --summary
```

Sample decoded audit record (`ausearch` output):
```text
type=USER_LOGIN msg=audit(1769512800.124:402): pid=14205 uid=0 auid=1001 ses=3 subj=unconfined 
msg='op=login id=1001 exe="/usr/sbin/sshd" hostname=203.0.113.85 addr=203.0.113.85 terminal=ssh res=success'
```
* `auid=1001`: Original authenticated identity before any `sudo` escalation.
* `ses=3`: The distinct kernel audit session identifier.
* `res=success`: Confirmed authentication event.

---

## 5. Comprehensive Practical Laboratories

### Lab 1: Multi-Hop Bastion Tunnel with Connection Multiplexing and SOCKS5 Egress

#### Objective
Configure an advanced, production-ready SSH client architecture from scratch using a dedicated test environment. Build a script that establishes an automated multiplexed master connection through a simulated bastion host, verifies socket persistence, provisions a dynamic SOCKS5 proxy, and validates remote DNS isolation.

#### Implementation Script
Save this script as `setup_ssh_tunnel_lab.sh` and execute:

```bash
#!/usr/bin/env bash
set -euo pipefail

LAB_DIR="/tmp/ssh_lab_environment"
SSH_CONF_DIR="${LAB_DIR}/client_home/.ssh"
SOCKET_DIR="${LAB_DIR}/sockets"
MOCK_SERVER_DIR="${LAB_DIR}/server_runtime"

echo "=== [STEP 1] PROVISIONING CLEAN ISOLATED LAB DIRECTORIES ==="
rm -rf "${LAB_DIR}"
mkdir -p "${SSH_CONF_DIR}" "${SOCKET_DIR}" "${MOCK_SERVER_DIR}"
chmod 700 "${LAB_DIR}" "${SSH_CONF_DIR}" "${SOCKET_DIR}"

echo "=== [STEP 2] GENERATING ED25519 CRYPTOGRAPHIC IDENTITIES ==="
# Generate host key for simulated local bastion
ssh-keygen -t ed25519 -N "" -f "${MOCK_SERVER_DIR}/ssh_host_ed25519_key"
# Generate client authentication identity
ssh-keygen -t ed25519 -N "" -f "${SSH_CONF_DIR}/id_ed25519_lab" -C "lab-engineer@system"

# Authorize the client key on the mock server
cp "${SSH_CONF_DIR}/id_ed25519_lab.pub" "${MOCK_SERVER_DIR}/authorized_keys"
chmod 600 "${MOCK_SERVER_DIR}/authorized_keys"

echo "=== [STEP 3] SPAWNING LOCAL MOCK SSH DAEMON ==="
# Find available high-range port for mock SSH server
SSHD_PORT=22222
cat << EOF > "${MOCK_SERVER_DIR}/sshd_config"
Port ${SSHD_PORT}
ListenAddress 127.0.0.1
HostKey ${MOCK_SERVER_DIR}/ssh_host_ed25519_key
AuthorizedKeysFile ${MOCK_SERVER_DIR}/authorized_keys
PidFile ${MOCK_SERVER_DIR}/sshd.pid
LogLevel DEBUG
StrictModes no
GatewayPorts yes
AcceptEnv LANG LC_*
Subsystem sftp internal-sftp
EOF

# Launch custom isolated sshd instance
/usr/sbin/sshd -f "${MOCK_SERVER_DIR}/sshd_config"
sleep 1

if ! ss -tlpn | grep -q ":${SSHD_PORT}"; then
    echo "[-] FAILED TO SPAWN TEST SSH DAEMON ON PORT ${SSHD_PORT}"
    exit 1
fi
echo "[+] Test SSH Daemon running on 127.0.0.1:${SSHD_PORT} (PID: $(cat "${MOCK_SERVER_DIR}/sshd.pid"))"

echo "=== [STEP 4] CONSTRUCTING ADVANCED ~/.ssh/config ==="
cat << EOF > "${SSH_CONF_DIR}/config"
Host *
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    LogLevel ERROR

Host mock-bastion
    HostName 127.0.0.1
    Port ${SSHD_PORT}
    User ${USER}
    IdentityFile ${SSH_CONF_DIR}/id_ed25519_lab
    IdentitiesOnly yes
    ControlMaster auto
    ControlPath ${SOCKET_DIR}/ctrl-%C
    ControlPersist 30s
    ServerAliveInterval 15
    ServerAliveCountMax 2

Host dynamic-socks
    HostName 127.0.0.1
    Port ${SSHD_PORT}
    User ${USER}
    IdentityFile ${SSH_CONF_DIR}/id_ed25519_lab
    DynamicForward 127.0.0.1:10888
    ControlMaster auto
    ControlPath ${SOCKET_DIR}/ctrl-%C
    ControlPersist 30s
EOF
chmod 600 "${SSH_CONF_DIR}/config"

echo "=== [STEP 5] TESTING MULTIPLEXING LIFECYCLE ==="
echo "[*] Establishing initial connection and provisioning ControlMaster..."
HOME="${LAB_DIR}/client_home" ssh -F "${SSH_CONF_DIR}/config" -N -f mock-bastion

echo "[*] Verifying presence of UNIX control socket:"
ls -la "${SOCKET_DIR}"

echo "[*] Executing sub-command over existing master multiplex pipe:"
time HOME="${LAB_DIR}/client_home" ssh -F "${SSH_CONF_DIR}/config" mock-bastion "echo 'Executing channel 1'"
time HOME="${LAB_DIR}/client_home" ssh -F "${SSH_CONF_DIR}/config" mock-bastion "echo 'Executing channel 2'"

echo "=== [STEP 6] TESTING DYNAMIC SOCKS5 PROXY ==="
echo "[*] Spawning dynamic proxy tunnel..."
HOME="${LAB_DIR}/client_home" ssh -F "${SSH_CONF_DIR}/config" -N -f dynamic-socks

echo "[*] Verifying local SOCKS5 listener on port 10888:"
ss -tlpn | grep ":10888"

echo "[*] Executing network call through SOCKS5 proxy:"
# Start a transient dummy web server on port 18080 to query through the proxy
python3 -m http.server 18080 --bind 127.0.0.1 >/dev/null 2>&1 &
HTTP_PID=$!
sleep 1

curl --socks5-hostname 127.0.0.1:10888 http://127.0.0.1:18080/ -I

echo "=== [STEP 7] CLEANUP TEARDOWN WORKFLOW ==="
echo "[*] Terminating master sockets..."
HOME="${LAB_DIR}/client_home" ssh -F "${SSH_CONF_DIR}/config" -O exit mock-bastion 2>/dev/null || true
HOME="${LAB_DIR}/client_home" ssh -F "${SSH_CONF_DIR}/config" -O exit dynamic-socks 2>/dev/null || true

echo "[*] Killing mock daemon and web server..."
kill $(cat "${MOCK_SERVER_DIR}/sshd.pid") 2>/dev/null || true
kill "${HTTP_PID}" 2>/dev/null || true
rm -rf "${LAB_DIR}"

echo "[SUCCESS] Multi-hop, multiplexing, and SOCKS5 proxying lab completed successfully."
```

---

### Lab 2: Dual-Stack Stateful Host Firewall Implementation using `nftables` with Set-Based Rate Limiting

#### Objective
Use network namespaces (`ip netns`) to construct a multi-node virtual topology. Within this environment, implement, test, and validate an `nftables` security boundary featuring strict stateful packet filtering, abnormal TCP flag detection, and automated dynamic set rate-limiting against simulated volumetric connection abuse.

```
                                  Laboratory Topology
 ┌──────────────────────────────────────────────────────────────────────────────────┐
 │ Target Server Namespace (`ns-srv`)                                               │
 │ Interface: `veth-srv` (192.168.50.2/24)                                          │
 │ Firewall: `nftables` (Stateful tracking, Drop Policy, Set-Based Abuse Limiter)   │
 └────────────────────────────────────────┬─────────────────────────────────────────┘
                                          │ Virtual Wire
 ┌────────────────────────────────────────┴─────────────────────────────────────────┐
 │ Attacker / Client Namespace (`ns-cli`)                                           │
 │ Interface: `veth-cli` (192.168.50.1/24)                                          │
 │ Tools: `ping`, `nc`, `python3` TCP Handshake Generators                          │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

#### Implementation Script
Save this script as `nftables_firewall_lab.sh` and execute with superuser privileges:

```bash
#!/usr/bin/env bash
set -euo pipefail

NS_CLI="ns-cli"
NS_SRV="ns-srv"

echo "=== [STEP 1] INSTANTIATING ISOLATED NETWORK NAMESPACES ==="
ip netns del "${NS_CLI}" 2>/dev/null || true
ip netns del "${NS_SRV}" 2>/dev/null || true

ip netns add "${NS_CLI}"
ip netns add "${NS_SRV}"

# Activate loopbacks
ip netns exec "${NS_CLI}" ip link set lo up
ip netns exec "${NS_SRV}" ip link set lo up

echo "=== [STEP 2] CREATING AND CONFIGURING VETH INTERCONNECTS ==="
ip link add veth-cli type veth peer name veth-srv
ip link set veth-cli netns "${NS_CLI}"
ip link set veth-srv netns "${NS_SRV}"

ip netns exec "${NS_CLI}" ip addr add 192.168.50.1/24 dev veth-cli
ip netns exec "${NS_CLI}" ip link set veth-cli up

ip netns exec "${NS_SRV}" ip addr add 192.168.50.2/24 dev veth-srv
ip netns exec "${NS_SRV}" ip link set veth-srv up

echo "=== [STEP 3] DEPLOYING NFTABLES RULESET INSIDE SERVER NAMESPACE ==="
ip netns exec "${NS_SRV}" nft -f - << 'EOF'
flush ruleset

table inet lab_firewall {
    # Dynamic set: Automatically stores attacking IPs that exceed thresholds
    set blocklist {
        type ipv4_addr
        flags timeout
        size 65535
    }

    # Meter: Tracks rate of new connection initiations
    set rate_meter {
        type ipv4_addr
        flags dynamic
        timeout 30s
        size 65535
    }

    chain input_chain {
        type filter hook input priority 0; policy drop;

        # Drop IPs currently listed in the dynamic blocklist
        ip saddr @blocklist drop

        # Permit established return traffic
        ct state established,related accept

        # Drop invalid states
        ct state invalid drop

        # Loopback
        iifname "lo" accept

        # Allow controlled ICMP
        ip protocol icmp limit rate 2/second accept

        # Web service on port 80: Permit
        tcp dport 80 ct state new accept

        # Protected administrative service on port 2222:
        # Limit to 3 connections every 10 seconds; exceeders get added to blocklist for 15s
        tcp dport 2222 ct state new meter rate_meter { ip saddr ct count over 3 } \
            add @blocklist { ip saddr timeout 15s } drop

        tcp dport 2222 ct state new accept
    }
}
EOF

echo "=== [STEP 4] SPAWNING DUMMY SERVICES IN SERVER NAMESPACE ==="
# Service A: HTTP on Port 80
ip netns exec "${NS_SRV}" python3 -m http.server 80 >/dev/null 2>&1 &
PID_HTTP=$!

# Service B: Mock Admin on Port 2222
ip netns exec "${NS_SRV}" nc -l -k -p 2222 >/dev/null 2>&1 &
PID_ADMIN=$!
sleep 1

echo "=== [STEP 5] TESTING AND VALIDATING POLICY BEHAVIOR ==="
echo "[Test 1] Validating standard ICMP (ping):"
ip netns exec "${NS_CLI}" ping -c 2 192.168.50.2

echo "[Test 2] Querying permitted Web Service (Port 80):"
ip netns exec "${NS_CLI}" python3 -c "
import urllib.request
res = urllib.request.urlopen('http://192.168.50.2:80', timeout=2)
print(f'HTTP Connection: SUCCESS (Status: {res.status})')
"

echo "[Test 3] Testing default DROP policy against unauthorized port (Port 9999):"
set +e
ip netns exec "${NS_CLI}" nc -z -w 1 192.168.50.2 9999
EXIT_CODE=$?
set -e
if [ ${EXIT_CODE} -ne 0 ]; then
    echo "Unauthorized Port 9999 BLOCKED (Policy Enforcement Confirmed)"
fi

echo "[Test 4] Simulating volumetric brute-force attack on Port 2222..."
ip netns exec "${NS_CLI}" python3 -c "
import socket, time
target = ('192.168.50.2', 2222)
for i in range(1, 6):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(0.5)
        s.connect(target)
        print(f'Connection {i}: ESTABLISHED')
        s.close()
    except Exception as e:
        print(f'Connection {i}: BLOCKED ({e})')
    time.sleep(0.1)
"

echo "[*] Inspecting active nftables sets inside server namespace:"
ip netns exec "${NS_SRV}" nft list set inet lab_firewall blocklist

echo "[Test 5] Validating that the blocklist now drops even authorized web traffic on Port 80:"
set +e
ip netns exec "${NS_CLI}" python3 -c "
import urllib.request
urllib.request.urlopen('http://192.168.50.2:80', timeout=1)
" 2>/dev/null
EXIT_CODE=$?
set -e
if [ ${EXIT_CODE} -ne 0 ]; then
    echo "Attacker IP successfully quarantined: Port 80 connection DROPPED"
fi

echo "=== [STEP 6] CLEANUP TEARDOWN WORKFLOW ==="
kill "${PID_HTTP}" "${PID_ADMIN}" 2>/dev/null || true
ip netns del "${NS_CLI}"
ip netns del "${NS_SRV}"

echo "[SUCCESS] Dynamic nftables isolation and rate-limiting lab completed cleanly."
```

---

### Lab 3: Forensic Authentication Audit and Connection Triage Script

#### Objective
Develop an automated bash forensics script capable of triaging an active Linux node. The tool audits open sockets, parses historical `btmp` and `wtmp` accounting records, queries the systemd journal for privilege transitions, and flags active network sessions linked to processes running out of volatile or suspect directories (e.g., `/tmp`, `/dev/shm`).

#### Implementation Script
Save this script as `network_auth_triage.sh` and make it executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "======================================================================"
echo "          LINUX REMOTE ACCESS & AUTHENTICATION TRIAGE REPORT          "
echo "  Timestamp: $(date -u '+%Y-%m-%d %H:%M:%S UTC') | Host: $(hostname)"
echo "======================================================================"

echo -e "\n[1] ACTIVE LOGGED-IN USERS (/run/utmp)"
echo "----------------------------------------------------------------------"
if [ -s /run/utmp ] || [ -s /var/run/utmp ]; then
    w -h | awk '{printf "User: %-10s TTY: %-8s Remote IP: %-16s Idle: %-6s CMD: %s\n", $1, $2, $3, $4, $5}'
else
    echo "No active sessions found in utmp."
fi

echo -e "\n[2] TOP 10 RECENT FAILED AUTHENTICATION ATTEMPTS (/var/log/btmp)"
echo "----------------------------------------------------------------------"
if [ -f /var/log/btmp ] && [ -r /var/log/btmp ]; then
    lastb -n 10 -F -i | head -n -2 | awk '{printf "Failed User: %-12s Source: %-18s Time: %s %s %s %s\n", $1, $3, $4, $5, $6, $7}'
else
    echo "btmp log unreadable or absent. Check root permissions."
fi

echo -e "\n[3] RECENT SUCCESSFUL SYSTEM LOGINS (/var/log/wtmp)"
echo "----------------------------------------------------------------------"
last -n 5 -F -i | head -n -2 | awk '{printf "User: %-10s Terminal: %-10s Source: %-18s Login: %s %s %s %s\n", $1, $2, $3, $4, $5, $6, $7}'

echo -e "\n[4] ESTABLISHED INBOUND NETWORK CONNECTIONS"
echo "----------------------------------------------------------------------"
# Extract established non-loopback connections with associated process binaries
ss -tanp state established '( sport != :127.0.0.1 and dport != :127.0.0.1 )' | awk '
NR>1 {
    printf "Local: %-22s Remote: %-22s Info: %s\n", $4, $5, $6
}'

echo -e "\n[5] PROCESS SUSPICION AUDIT (BINARIES RUNNING OUT OF VOLATILE STORAGE)"
echo "----------------------------------------------------------------------"
# Inspect processes with active network sockets whose executable binary is in /tmp, /dev/shm, or /var/tmp
SUSPICION_COUNT=0
while read -r pid; do
    if [ -d "/proc/${pid}" ]; then
        EXE_PATH=$(readlink -f "/proc/${pid}/exe" 2>/dev/null || echo "[UNREADABLE]")
        if [[ "${EXE_PATH}" =~ ^/tmp || "${EXE_PATH}" =~ ^/dev/shm || "${EXE_PATH}" =~ ^/var/tmp ]]; then
            echo "[ALERT] Suspicious Binary Detected!"
            echo "  PID: ${pid} | Path: ${EXE_PATH}"
            echo "  Command Line: $(cat "/proc/${pid}/cmdline" | tr '\0' ' ')"
            echo "  Open Sockets: $(ls -l "/proc/${pid}/fd" 2>/dev/null | grep socket || true)"
            SUSPICION_COUNT=$((SUSPICION_COUNT + 1))
        fi
    fi
done < <(ss -tupn | awk -F'pid=' '{print $2}' | awk -F',' '{print $1}' | sort -u | grep -v '^$')

if [ "${SUSPICION_COUNT}" -eq 0 ]; then
    echo "No suspicious network-bound executables executing from /tmp, /dev/shm, or /var/tmp."
fi

echo -e "\n[6] RECENT SUDO PRIVILEGE ELEVATIONS (LAST 1 HOUR)"
echo "----------------------------------------------------------------------"
if command -v journalctl >/dev/null 2>&1; then
    journalctl _COMM=sudo --since "1 hour ago" --no-pager -q -o cat | tail -n 5 || echo "No sudo records found."
fi

echo -e "\n======================================================================"
echo "                         END OF FORENSIC REPORT                       "
echo "======================================================================"
```

---

## 6. Comprehensive Reference Matrices

### 1. `~/.ssh/config` Directive Reference Matrix

| Directive Keyword | Argument / Values | Operational Scope | Impact / Recommended Tuning |
| :--- | :--- | :--- | :--- |
| **`HostName`** | FQDN or IP Address | Connection Target | Overrides the literal host alias with the routable target address. |
| **`User`** | Username String | Identity Mapping | Sets the remote username to authenticate against. |
| **`Port`** | Integer ($1\text{--}65535$) | Transport Layer | Overrides the default TCP destination port (default: `22`). |
| **`IdentityFile`** | Absolute / Relative Path | Authentication | Specifies the exact private key to present. Can be declared multiple times. |
| **`IdentitiesOnly`** | `yes` / `no` | Authentication Hardening | `yes`: Disallows using unconfigured keys held in `ssh-agent`. Prevents auth lockout. |
| **`ProxyJump`** | `host[:port],...` | Bastion Traversal | Sets intermediate jump hosts. Direct end-to-end channel creation. |
| **`ControlMaster`** | `yes` / `no` / `auto` / `ask` | Multiplexing | Enables sharing multiple concurrent sessions over a single network pipe. |
| **`ControlPath`** | Filesystem Path String | Multiplexing Socket | Path to UNIX domain socket. Use `%C` hash tokens to avoid length exhaustion. |
| **`ControlPersist`** | `yes` / `no` / Time Duration | Transport Persistence | Keeps master daemon socket alive in background after last shell closes (e.g., `10m`). |
| **`ForwardAgent`** | `yes` / `no` | Security / Credentials | Should be `no` globally. Use `ProxyJump` instead to avoid credential hijacking. |
| **`ServerAliveInterval`**| Integer (Seconds) | Link Stability | Seconds between heartbeat probes sent to remote daemon. Mitigates NAT timeouts. |
| **`ServerAliveCountMax`** | Integer (Count) | Fault Detection | Consecutive unacknowledged keep-alive probes before client closes dead pipe. |
| **`StrictHostKeyChecking`**| `yes` / `no` / `accept-new` | MitM Mitigation | `yes`: Refuses to connect if host key changed; `accept-new`: Auto-adds new keys. |

---

### 2. `iptables` vs. `nftables` Syntax Translation Matrix

| Functional Target | Legacy `iptables` Command Syntax | Modern `nftables` Native Syntax |
| :--- | :--- | :--- |
| **Flush All Rules** | `iptables -F && iptables -t nat -F` | `nft flush ruleset` |
| **Default Chain Policy**| `iptables -P INPUT DROP` | `nft add chain inet filter input { policy drop \; }` |
| **Loopback Interface** | `iptables -A INPUT -i lo -j ACCEPT` | `nft add rule inet filter input iif "lo" accept` |
| **Conntrack Established**| `iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT` | `nft add rule inet filter input ct state established,related accept` |
| **Multiport TCP Match** | `iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT` | `nft add rule inet filter input tcp dport { 80, 443 } accept` |
| **Match Source Subnet** | `iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 22 -j ACCEPT` | `nft add rule inet filter input ip saddr 10.0.0.0/8 tcp dport 22 accept` |
| **Logging with Prefix** | `iptables -A INPUT -j LOG --log-prefix "FW_DROP: "` | `nft add rule inet filter input log prefix \"FW_DROP: \"` |
| **Rate-Limiting Rule** | `iptables -A INPUT -p icmp -m limit --limit 5/s -j ACCEPT` | `nft add rule inet filter input ip protocol icmp limit rate 5/second accept` |
| **Port Forward (DNAT)** | `iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 10.0.0.5:80` | `nft add rule ip nat prerouting tcp dport 8080 dnat to 10.0.0.5:80` |
| **Source NAT (Masquerade)**| `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` | `nft add rule ip nat postrouting oifname "eth0" masquerade` |

---

### 3. UTMP / WTMP / BTMP System Accounting Matrix

| Accounting Target | Storage Filesystem Path | Record Struct | Governing Writer | Inspection Tools | Security Auditing Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Active Sessions** | `/run/utmp` | `struct utmp` | `login`, `sshd`, `getty` | `who`, `w`, `users` | Inspect live terminal users, PTY lines, and idle times. |
| **Historical Logins**| `/var/log/wtmp` | `struct utmp` | `init`, `systemd`, `sshd` | `last` | Track historic user access, session duration, and system reboots. |
| **Failed Logins** | `/var/log/btmp` | `struct utmp` | `login`, `sshd`, PAM | `lastb` | Detect brute-force password spraying and unauthorized access attempts. |
| **Last Login Record**| `/var/log/lastlog` | `struct lastlog`| `pam_lastlog.so` | `lastlog` | Identify dormant user accounts that have never authenticated. |

---

### 4. Network Security Diagnostic CLI Matrix

| Command & Flags | Subsystem | Diagnostic Objective |
| :--- | :--- | :--- |
| **`ss -tulnp`** | Sockets | Displays listening TCP/UDP sockets with numeric ports, PIDs, and process names. |
| **`ss -tanpe`** | Sockets | Displays all established connections with socket buffer memory, UID, and inode metrics. |
| **`conntrack -L`** | Netfilter | Dumps the active in-kernel connection tracking state table. |
| **`conntrack -E`** | Netfilter | Continuously monitors real-time Netfilter connection state transitions. |
| **`nft list ruleset`**| Firewall | Prints the entire active `nftables` ruleset, including dynamic sets and meters. |
| **`nft monitor`** | Firewall | Live stream of ruleset updates, Netlink notifications, and packet traces. |
| **`last -i -F`** | Auth Accounting | Displays `wtmp` historical sessions showing full dates, times, and numeric IP addresses. |
| **`lastb -i -n 20`** | Auth Accounting | Dumps the 20 most recent failed login events from `btmp`. |
| **`ausearch -m USER_AUTH`** | Audit Subsystem | Queries auditd logs for user authentication and PAM verification events. |
| **`aureport --auth`** | Audit Subsystem | Generates a consolidated summary of authentication attempts from the audit trail. |