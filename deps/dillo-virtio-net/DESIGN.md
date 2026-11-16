# Virtio-net design

Networking follows [Dillo's machine and composition rules](../../DESIGN.md).
The virtio-net device implements shared guest protocol behavior, independently
of its virtio transport and host connectivity mode. Each configured device
uses an explicitly selected bus and optional slot under DT-004. Its MAC address
is device configuration, independent of host connectivity.

## Host connectivity

A common frame interface separates the device from host connectivity. The
selected Machine's interfaces and associated types govern host-specific
resources, socket and interface access, notifications, execution, and
confinement. Shared network protocol behavior uses conforming services.
Resources and device processes may be established independently or early;
construction need not pass through a live Machine instance. Auxiliary
networking and DNS work obey the same lifecycle and authority contracts as
other device work.

| Mode | Required behavior |
| --- | --- |
| `user` | Default mode on Linux, macOS, and Windows; outbound TCP/UDP connectivity through ordinary host sockets, host-loopback access through the gateway, and explicit inbound forwarding, without privileged host network setup. |
| `bridge` | Join the selected existing host L2 network through host-specific services; require `iface`. |
| `macvtap` | Attach to an existing Linux macvtap endpoint; require `iface`. |

A mode unavailable on the selected host must fail explicitly. Do not silently
switch modes, create another guest device, or change the PMI transport to work
around host limitations. Host interface management stays within the authority
granted for the configured endpoint.

## User-mode behavior

The guest uses a private network. Outbound flows are re-originated through
host sockets. Gateway aliases permit guest-to-host communication; inbound
connections require configured forwarding. Preserve TCP closure, half-close,
backpressure, and error semantics, and bounded UDP flow lifetimes.

Address assignment, local DNS, and IPv6 configuration are shared protocol
behavior. Host resolution and networking facilities come through machine
services. Usage documentation defines the exposed addressing defaults.

## Inbound forwarding

CLI and JSON share one forwarding schema. Each rule accepts a structured form
or shorthand `[proto:]port[:guest]`. Protocol is `tcp` by default or `udp`;
`port` is required; `guest` defaults to `port`. The host bind address defaults
to `127.0.0.1`; an explicit `ip` in the structured form selects another address.
The guest destination address belongs to the private network configuration.

Forwarding controls host-to-guest initiation; established traffic flows in both
directions. Outbound and gateway access do not require forwarding rules.
Reject malformed rules, forwards on non-user modes, missing required
interfaces, and unavailable host capabilities with actionable errors.

## Resource and lifecycle behavior

Guest-controlled traffic must not create unbounded flows, pending resolution
work, buffers, or host resources. Validate frames before allocating resources.
Report endpoint establishment failures and maintain bounded teardown of
networking and resolution work. Reset and reactivation preserve coherent
transport/device state under LIFECYCLE-001. These requirements apply with both
local services and out-of-process proxies.

See [README.md](README.md) for usage and validation information.
