# Adaptive Communication Network with Failure Recovery

**Project 30 – Computer Networks**

### Team Members
- Saankhya Srikanth
- Revant H
- Rishith S

## Overview

This project implements a UDP-based communication network that can automatically recover from network failures.

An SDN controller using **Ryu** monitors the network and reroutes traffic through an alternate path when a link fails.

## Network Topology

```text
              s2
             /  \
h1 ─┐       /    \
h2 ─┼── s1          s4 ── h4
h3 ─┘       \    /
             \  /
              s3
```

**Primary path:** `s1 → s2 → s4`  
**Backup path:** `s1 → s3 → s4`

## How It Works

1. UDP clients send data to the server.
2. Ryu monitors the network topology.
3. If the primary link fails, the controller detects it.
4. The controller calculates an alternate path and updates the switches.
5. The UDP application detects lost packets and retransmits them.

## Technologies Used

- Python
- UDP Sockets
- Mininet
- Open vSwitch
- OpenFlow 1.3
- Ryu SDN Controller

## Failure Demonstration

A link failure can be simulated in Mininet:

```text
mininet> link s1 s2 down
```

Traffic switches from:

```text
s1 → s2 → s4
```

to:

```text
s1 → s3 → s4
```

The link can then be restored using:

```text
mininet> link s1 s2 up
```

## Testing

The project is tested for:

- Normal UDP communication
- Link failure and recovery
- Automatic rerouting
- Packet loss and retransmission
- Multiple clients

## Expected Outcome

The network should continue communication even when the primary path fails, with the SDN controller automatically switching traffic to the backup path.
