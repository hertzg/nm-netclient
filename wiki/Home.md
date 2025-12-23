# Netclient Wiki

Welcome to the Netclient documentation. Netclient is an automated WireGuard management client for the [Netmaker](https://github.com/gravitl/netmaker) platform.

## Overview

Netclient is a daemon and CLI tool that enables clients to join, manage, and maintain connections to WireGuard VPN networks managed by a Netmaker server. It handles:

- **Peer Discovery** - Automatic peer configuration and updates
- **Configuration Sync** - Real-time synchronization with Netmaker server via MQTT
- **DNS Management** - Network-specific DNS resolution
- **Firewall Rules** - Access control list enforcement
- **Network Routing** - Egress/ingress gateway functionality

## Quick Links

| Documentation | Description |
|--------------|-------------|
| [Architecture](Architecture) | System architecture and component overview |
| [Workflows](Workflows) | Swimlane diagrams for key processes |
| [System Integration](System-Integration) | OS integration (systemd, firewall, DNS) |
| [Components](Components) | Detailed component breakdown |
| [Configuration](Configuration) | Configuration management and storage |

## Supported Platforms

| Platform | Init System | Firewall | DNS Management |
|----------|-------------|----------|----------------|
| Linux | systemd, sysvinit, openrc, runit | iptables, nftables | systemd-resolved, resolvconf |
| macOS | launchd | - | scutil |
| FreeBSD | rc.d | pf | resolvconf |
| Windows | Windows Service | Windows Firewall | Registry |

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              NETMAKER SERVER                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                      │
│  │   REST API  │    │ MQTT Broker │    │  Database   │                      │
│  └──────┬──────┘    └──────┬──────┘    └─────────────┘                      │
└─────────┼──────────────────┼────────────────────────────────────────────────┘
          │                  │
          │ HTTPS            │ MQTT (TLS)
          │                  │
┌─────────┼──────────────────┼────────────────────────────────────────────────┐
│         ▼                  ▼                        NETCLIENT HOST          │
│  ┌─────────────┐    ┌─────────────┐                                         │
│  │  Auth/API   │    │ MQ Handlers │◄─── Node Updates, DNS Sync, ACLs        │
│  │   Client    │    │             │                                         │
│  └──────┬──────┘    └──────┬──────┘                                         │
│         │                  │                                                 │
│         ▼                  ▼                                                 │
│  ┌─────────────────────────────────┐                                        │
│  │         DAEMON CORE             │                                        │
│  │  ┌─────────┐  ┌──────────────┐  │                                        │
│  │  │ Config  │  │   Functions  │  │                                        │
│  │  │ Manager │  │    Layer     │  │                                        │
│  │  └────┬────┘  └───────┬──────┘  │                                        │
│  └───────┼───────────────┼─────────┘                                        │
│          │               │                                                   │
│          ▼               ▼                                                   │
│  ┌───────────────────────────────────────────────────────────────┐          │
│  │                    SYSTEM INTEGRATION                          │          │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │          │
│  │  │WireGuard │  │ Firewall │  │   DNS    │  │ Routing  │       │          │
│  │  │Interface │  │(iptables)│  │ Resolver │  │  Table   │       │          │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │          │
│  └───────────────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Communication Flow

```
                    ┌──────────────┐
                    │   Netmaker   │
                    │    Server    │
                    └───────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │Netclient │  │Netclient │  │Netclient │
        │  Host A  │  │  Host B  │  │  Host C  │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │             │             │
             └─────────────┼─────────────┘
                           │
                    WireGuard Mesh
                    (Direct P2P or
                     via Relay)
```

## Version

Current Version: **v1.4.0**
