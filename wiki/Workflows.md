# Workflows

This document contains swimlane diagrams for key Netclient workflows.

## Table of Contents

- [Join Network Flow](#join-network-flow)
- [Daemon Startup Flow](#daemon-startup-flow)
- [Node Update Flow](#node-update-flow)
- [DNS Synchronization Flow](#dns-synchronization-flow)
- [Peer Connection Flow](#peer-connection-flow)
- [Leave Network Flow](#leave-network-flow)
- [Auto-Relay Selection Flow](#auto-relay-selection-flow)

---

## Join Network Flow

This diagram shows the complete flow when a user joins a new network.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant CLI as CLI (cmd/join.go)
    participant Register as Register (functions/)
    participant Config as Config Manager
    participant API as Netmaker API
    participant Daemon as Daemon Process

    User->>CLI: netclient join -t <token>

    CLI->>CLI: validateArgs()
    CLI->>CLI: setHostFields()

    CLI->>Register: Register(token)

    Register->>Register: Decode base64 token envelope
    Note over Register: Extract server URL,<br/>network, credentials

    Register->>Register: GetLocalIfaces()
    Register->>Register: GetDefaultInterface()

    Register->>API: POST /api/v1/host/register/{token}
    Note over API: Validate token<br/>Create host record<br/>Assign IP address

    API-->>Register: HostRegisterResponse
    Note over Register: Contains:<br/>- Server config<br/>- Node config<br/>- Peer list

    Register->>Config: UpdateServerConfig()
    Register->>Config: SaveServer()
    Register->>Config: UpdateHost()
    Register->>Config: WriteNodeConfig()

    Config->>Config: Atomic JSON write

    Register->>Daemon: daemon.Restart()

    Daemon->>Daemon: Load new config
    Daemon->>Daemon: Create WireGuard interface
    Daemon->>Daemon: Connect to MQTT

    Daemon-->>User: Network joined successfully
```

**Detailed Steps:**

| Step | Component | Action |
|------|-----------|--------|
| 1 | CLI | Parse command arguments, validate token format |
| 2 | CLI | Set host fields (name, interfaces, endpoints) |
| 3-4 | Register | Decode base64 envelope containing server URL and credentials |
| 5-6 | Register | Discover local network interfaces |
| 7-8 | API | Register host with Netmaker server |
| 9-12 | Config | Persist server, node, and host configuration |
| 13-16 | Daemon | Restart daemon to apply new configuration |

---

## Daemon Startup Flow

This diagram shows the daemon initialization sequence.

```mermaid
sequenceDiagram
    autonumber
    participant OS as Operating System
    participant Main as main.go
    participant Cmd as cmd/root.go
    participant Init as InitConfig()
    participant Daemon as functions/daemon.go
    participant WG as WireGuard
    participant MQTT as MQTT Client
    participant DNS as DNS Server

    OS->>Main: Start netclient daemon

    Main->>Cmd: Execute()

    Cmd->>Init: InitConfig()

    Init->>Init: CheckUID() - verify root

    Init->>Init: RemoveAllLockFiles()

    Init->>Init: Migrate YAML → JSON
    Note over Init: Legacy config migration

    Init->>Init: ReadNetclientConfig()
    Init->>Init: ReadNodeConfig()
    Init->>Init: ReadServerConf()

    Init->>Init: SetServerCtx()
    Note over Init: Set current server context

    Init->>Init: checkConfig()
    Note over Init: Validate/generate keys

    Init->>WG: Create test interface
    WG-->>Init: Verify WG working

    Init-->>Cmd: Config ready

    Cmd->>Daemon: Daemon()

    par Background Goroutines
        Daemon->>MQTT: Connect to broker
        MQTT->>MQTT: Subscribe to topics
        Note over MQTT: host/{id}/update/*<br/>host/{id}/dns/*<br/>host/{id}/acl/*

        Daemon->>DNS: Start DNS server
        Note over DNS: Listen on :53

        Daemon->>Daemon: Start metrics loop
        Daemon->>Daemon: Start ping loop
        Daemon->>Daemon: Start relay checker
    end

    Daemon->>Daemon: Setup signal handlers
    Note over Daemon: SIGTERM → shutdown<br/>SIGHUP → reload

    loop Main Event Loop
        Daemon->>Daemon: Wait for signals/events
    end
```

**Goroutine Overview:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           DAEMON GOROUTINES                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                         MAIN GOROUTINE                                   │    │
│  │                                                                          │    │
│  │   • Configuration initialization                                         │    │
│  │   • Signal handling (SIGTERM, SIGHUP)                                   │    │
│  │   • Graceful shutdown coordination                                       │    │
│  │                                                                          │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐        │
│  │ MQTT Handler  │ │ Metrics Loop  │ │  Ping Loop    │ │ Relay Checker │        │
│  │               │ │               │ │               │ │               │        │
│  │ • Message rx  │ │ • Collect     │ │ • ICMP ping   │ │ • Test relays │        │
│  │ • Dispatch    │ │ • Report      │ │ • TCP ping    │ │ • Select best │        │
│  │ • Update cfg  │ │ • Every 5min  │ │ • Every 30s   │ │ • Update peer │        │
│  └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘        │
│                                                                                  │
│  ┌───────────────┐                                                               │
│  │  DNS Server   │                                                               │
│  │               │                                                               │
│  │ • UDP :53     │                                                               │
│  │ • Resolve     │                                                               │
│  │ • Forward     │                                                               │
│  └───────────────┘                                                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Node Update Flow

This diagram shows how configuration updates are processed via MQTT.

```mermaid
sequenceDiagram
    autonumber
    participant Server as Netmaker Server
    participant Broker as MQTT Broker
    participant MQ as MQ Handler
    participant Decrypt as Decryption
    participant Cache as Message Cache
    participant WG as WireGuard
    participant FW as Firewall
    participant Pub as MQ Publisher

    Server->>Broker: Publish node update

    Broker->>MQ: Message on host/{id}/update/{net}

    MQ->>Decrypt: DecryptMessage(payload)

    alt AES-GCM Encryption
        Decrypt->>Decrypt: AES-GCM decrypt
    else RSA Encryption (legacy)
        Decrypt->>Decrypt: RSA decrypt
    end

    Decrypt->>Decrypt: GZIP decompress

    Decrypt-->>MQ: Decrypted JSON

    MQ->>MQ: Parse Node struct

    MQ->>Cache: Check message cache
    Note over Cache: Prevent duplicate<br/>processing

    alt Already processed
        Cache-->>MQ: Skip (cached)
    else New message
        Cache->>Cache: Store in cache

        MQ->>MQ: IfaceDelta()
        Note over MQ: Compare with<br/>current config

        alt Configuration changed
            MQ->>WG: Create NCIface
            MQ->>WG: Configure(peers)

            WG->>WG: Apply WireGuard config
            WG->>WG: Update peer list
            WG->>WG: Set allowed IPs

            MQ->>FW: Update firewall rules
            FW->>FW: Apply ACL changes
        end

        MQ->>Pub: PublishSignal("DONE")
        Pub->>Broker: Signal completion
    end
```

**Message Flow Detail:**

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          NODE UPDATE MESSAGE FLOW                               │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   INCOMING MESSAGE                                                              │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                         │  │
│   │   Topic: host/{host-id}/update/{network}                                │  │
│   │                                                                         │  │
│   │   Payload (encrypted + compressed):                                     │  │
│   │   ┌─────────────────────────────────────────────────────────────────┐  │  │
│   │   │  AES-GCM(GZIP(JSON))                                            │  │  │
│   │   │                                                                  │  │  │
│   │   │  Decrypted JSON:                                                 │  │  │
│   │   │  {                                                               │  │  │
│   │   │    "network": "mynet",                                           │  │  │
│   │   │    "address": "10.10.10.5/24",                                   │  │  │
│   │   │    "peers": [...],                                               │  │  │
│   │   │    "dns_on": true,                                               │  │  │
│   │   │    "is_egress": false,                                           │  │  │
│   │   │    ...                                                           │  │  │
│   │   │  }                                                               │  │  │
│   │   └─────────────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│   PROCESSING PIPELINE                                                           │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐         │
│   │ Receive │──►│ Decrypt │──►│Decomprss│──►│  Parse  │──►│  Apply  │         │
│   │         │   │ AES-GCM │   │  GZIP   │   │  JSON   │   │  Config │         │
│   └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘         │
│                                                                                 │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## DNS Synchronization Flow

```mermaid
sequenceDiagram
    autonumber
    participant Server as Netmaker Server
    participant Broker as MQTT Broker
    participant Handler as DNS Handler
    participant Config as DNS Config
    participant Resolver as DNS Resolver
    participant System as System DNS

    Server->>Broker: Publish DNS update

    Broker->>Handler: Message on host/dns/sync/{server}

    Handler->>Handler: Parse DNS payload
    Note over Handler: Extract:<br/>- DNS entries<br/>- Nameservers<br/>- Search domains

    Handler->>Config: UpdateDNSConfig()

    Config->>Config: Write dns.json

    Config->>Resolver: Reload configuration

    Resolver->>Resolver: Update A/AAAA records
    Resolver->>Resolver: Update PTR records
    Resolver->>Resolver: Set upstream servers

    alt Linux with systemd-resolved
        Config->>System: Update via D-Bus
    else Linux with resolvconf
        Config->>System: Update /etc/resolv.conf
    else Windows
        Config->>System: Update Registry
    end

    Handler->>Handler: Restart DNS listener
    Note over Handler: Rebind to :53
```

---

## Peer Connection Flow

```mermaid
sequenceDiagram
    autonumber
    participant NodeA as Node A (Initiator)
    participant STUN as STUN Server
    participant Server as Netmaker Server
    participant NodeB as Node B (Responder)
    participant WGA as WireGuard A
    participant WGB as WireGuard B

    Note over NodeA,NodeB: Initial Discovery Phase

    NodeA->>STUN: Request external IP
    STUN-->>NodeA: Public IP + Port

    NodeA->>Server: Report endpoint info
    Server->>Server: Store peer data

    NodeB->>STUN: Request external IP
    STUN-->>NodeB: Public IP + Port

    NodeB->>Server: Report endpoint info

    Note over NodeA,NodeB: Configuration Distribution

    Server->>NodeA: Push NodeB peer config
    Note over NodeA: Contains:<br/>- Public key<br/>- Endpoint<br/>- Allowed IPs

    Server->>NodeB: Push NodeA peer config

    Note over NodeA,NodeB: WireGuard Handshake

    NodeA->>WGA: Configure peer

    WGA->>WGB: Initiation (encrypted)
    Note over WGA,WGB: Noise Protocol<br/>IK Handshake

    WGB->>WGA: Response (encrypted)

    WGA->>WGB: Confirmation

    Note over WGA,WGB: Tunnel Established

    loop Keepalive
        WGA->>WGB: Keepalive (25s)
        WGB->>WGA: Keepalive (25s)
    end
```

**NAT Traversal Decision Tree:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          NAT TRAVERSAL DECISION                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                          ┌───────────────┐                                       │
│                          │  Both peers   │                                       │
│                          │  have public  │                                       │
│                          │     IPs?      │                                       │
│                          └───────┬───────┘                                       │
│                                  │                                               │
│                    ┌─────────────┴─────────────┐                                │
│                    │                           │                                │
│                   YES                          NO                               │
│                    │                           │                                │
│                    ▼                           ▼                                │
│           ┌───────────────┐          ┌───────────────┐                          │
│           │ Direct P2P    │          │  NAT type     │                          │
│           │ connection    │          │  compatible?  │                          │
│           └───────────────┘          └───────┬───────┘                          │
│                                              │                                   │
│                                ┌─────────────┴─────────────┐                    │
│                                │                           │                    │
│                               YES                          NO                   │
│                                │                           │                    │
│                                ▼                           ▼                    │
│                       ┌───────────────┐          ┌───────────────┐              │
│                       │  UDP hole     │          │   Use relay   │              │
│                       │  punching     │          │     node      │              │
│                       └───────────────┘          └───────┬───────┘              │
│                                                          │                      │
│                                                          ▼                      │
│                                                 ┌───────────────┐               │
│                                                 │ Select best   │               │
│                                                 │ relay by      │               │
│                                                 │ latency       │               │
│                                                 └───────────────┘               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Leave Network Flow

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant CLI as CLI (cmd/leave.go)
    participant Functions as Functions Layer
    participant Config as Config Manager
    participant WG as WireGuard
    participant FW as Firewall
    participant DNS as DNS
    participant API as Netmaker API
    participant Daemon as Daemon

    User->>CLI: netclient leave -n <network>

    CLI->>Functions: LeaveNetwork(network)

    Functions->>Config: GetNode(network)

    Functions->>WG: RemovePeers(network)
    WG->>WG: Clear peer config

    Functions->>WG: RemoveInterface()
    WG->>WG: Delete nm-* interface

    Functions->>FW: CleanupRules(network)
    FW->>FW: Remove ACL rules
    FW->>FW: Remove NAT rules

    Functions->>DNS: RemoveEntries(network)
    DNS->>DNS: Clear DNS records
    DNS->>DNS: Restore system DNS

    Functions->>API: DELETE /api/nodes/{network}/{id}
    API-->>Functions: Confirm deletion

    Functions->>Config: DeleteNode(network)
    Config->>Config: Remove from nodes.json

    alt Last network on server
        Functions->>Config: DeleteServer()
        Config->>Config: Remove from servers.json
    end

    Functions->>Daemon: Restart()

    Daemon-->>User: Left network successfully
```

---

## Auto-Relay Selection Flow

```mermaid
sequenceDiagram
    autonumber
    participant Checker as Relay Checker
    participant Cache as Relay Cache
    participant Peers as Peer List
    participant Ping as Ping Module
    participant Config as Config Manager
    participant WG as WireGuard

    loop Every check interval

        Checker->>Cache: GetCachedRelays(network)

        alt Cache empty or stale
            Checker->>Peers: GetAvailableRelays()
            Peers-->>Checker: List of relay nodes
            Checker->>Cache: UpdateCache(relays)
        end

        Checker->>Checker: FilterEligibleRelays()
        Note over Checker: Must be:<br/>- Online<br/>- Has relay flag<br/>- Reachable

        loop For each relay candidate
            Checker->>Ping: MeasureLatency(relay)

            alt TCP Ping (internal)
                Ping->>Ping: TCP connect test
            else ICMP Ping (external)
                Ping->>Ping: ICMP echo request
            end

            Ping-->>Checker: Latency (ms)
        end

        Checker->>Checker: SortByLatency()
        Checker->>Checker: SelectOptimal()
        Note over Checker: Choose lowest<br/>latency relay

        alt Relay changed
            Checker->>Config: UpdateRelayConfig()
            Checker->>WG: UpdatePeerEndpoint()
            WG->>WG: Route through new relay
        end

    end
```

**Relay Selection Criteria:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          RELAY SELECTION CRITERIA                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ELIGIBILITY REQUIREMENTS                                                       │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │   1. Node has relay capability flag set                                 │   │
│   │   2. Node is currently online (recent heartbeat)                        │   │
│   │   3. Node has public IP or reachable endpoint                          │   │
│   │   4. Node is on same network                                           │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   SELECTION ALGORITHM                                                            │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │   for each eligible_relay:                                              │   │
│   │       latency = measure_tcp_ping(relay.endpoint)                        │   │
│   │       if latency < best_latency:                                        │   │
│   │           best_relay = relay                                            │   │
│   │           best_latency = latency                                        │   │
│   │                                                                         │   │
│   │   if best_relay != current_relay:                                       │   │
│   │       switch_to_relay(best_relay)                                       │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   LATENCY THRESHOLDS                                                             │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │   Excellent:  < 50ms   │  Good:  50-100ms  │  Acceptable:  100-200ms   │   │
│   │   Poor:      200-500ms │  Bad:    > 500ms  │  Timeout:     > 2000ms    │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Configuration Pull Flow

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant CLI as CLI (cmd/pull.go)
    participant Functions as Functions Layer
    participant Auth as Auth Module
    participant API as Netmaker API
    participant Config as Config Manager
    participant WG as WireGuard

    User->>CLI: netclient pull -n <network>

    CLI->>Functions: Pull(network)

    Functions->>Auth: GetAuthToken()

    alt Token expired or missing
        Auth->>API: POST /api/hosts/adm/authenticate
        API-->>Auth: JWT token
        Auth->>Auth: Cache token
    end

    Auth-->>Functions: Valid token

    Functions->>API: GET /api/nodes/{network}/{id}
    Note over API: Include Auth header

    API-->>Functions: Node configuration

    Functions->>Config: UpdateNodeConfig()
    Config->>Config: Write nodes.json

    Functions->>WG: ApplyConfiguration()
    WG->>WG: Update peers
    WG->>WG: Update allowed IPs

    Functions-->>User: Configuration updated
```

---

## Related Documentation

- [Architecture](Architecture) - Component architecture diagrams
- [System Integration](System-Integration) - OS-level integration
- [Components](Components) - Detailed package breakdown
- [Configuration](Configuration) - Config file formats
