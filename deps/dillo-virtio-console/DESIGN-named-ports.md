# Virtio-console design

The console follows [Dillo's machine and composition rules](../../DESIGN.md).
Its behavior is shared across transports and host platforms. Host endpoint
adoption, I/O, notifications, execution, and confinement conform to the selected
Machine's service interfaces and associated types. Endpoints and device
processes may be established independently or early; their construction need
not pass through a live Machine instance.

## Device and discovery

A configured console is one virtio-console device (device ID 3), with port 0
and optional additional named ports. It is discovered through its specified
virtio-mmio or PCI transport. The devicetree does not describe the console or
its ports (DT-006).

Port 0 is always present and unnamed, connected to stdio. Named ports receive
IDs 1 through N in configuration order; they do not replace port 0.

Offer `VIRTIO_CONSOLE_F_MULTIPORT` when named ports are nonempty or a resize
endpoint is configured. Otherwise use the single-port, two-queue behavior.
When MULTIPORT is negotiated, queues 0/1 serve port 0, queues 2/3 serve control,
and queues `2i+2`/`2i+3` serve named port i. `max_nr_ports` is `1+N`.
Transport interrupt capacity must cover the device's queues and configuration.

## Configuration contract

Console configuration is singular: `Layout.console`, with a nested `ports`
list. CLI and JSON express the same schema. Ports are not separate devices or
separate console arguments.

```text
--console endpoint=stdio,bus=mmio,ports=[[name=build,rx=pipe:3,tx=pipe:4],[name=log,tx=pipe:5]]
```

```json
{
  "console": {
    "endpoint": "stdio",
    "bus": "mmio",
    "ports": [
      {"name": "build", "rx": "pipe:3", "tx": "pipe:4"},
      {"name": "log", "tx": "pipe:5"}
    ]
  }
}
```

| Field | Requirement |
| --- | --- |
| `endpoint` | `stdio`; explicit when named ports are nonempty, otherwise defaults to `stdio`. |
| `bus` | Required; selects an available declared bus, with no global default. |
| `slot` | Optional exact placement under DT-004. |
| `ports` | Optional list, empty by default; at most 31 named ports. |

Thus `--console bus=mmio` configures a single-port console unless resize is
also requested. Examples using `mmio` require that transport in the PMI.

| Named-port field | Requirement |
| --- | --- |
| `name` | Unique nonempty free-form name, at most 255 bytes, with no NUL. No reserved consumer names. |
| `rx` | Optional `pipe:N`; Dillo reads host-to-guest bytes. |
| `tx` | Optional `pipe:N`; Dillo writes guest-to-host bytes. |

At least one direction is required. An omitted direction is closed; it does
not default to stdio. Each inherited pipe number may appear only once across
all named ports and directions. Numbers 0, 1, and 2 are reserved for stdio.

## Host endpoints

For `pipe:N`, the parent creates an anonymous pipe and inherits its device-side
end into Dillo. N denotes the inherited host descriptor or handle, represented
as an unsigned 64-bit number and checked against host limits at adoption.
A bidirectional port uses two pipes. The same layout syntax applies on all
hosts; inheritance and ownership mechanisms belong to machine services.

Endpoint validation failures identify the affected port and direction before
guest execution. EOF or a broken pipe closes that direction without spinning,
aborting the VM, or disrupting other ports. Shutdown is bounded even when an
input endpoint has no data. Each owned resource has one clear teardown owner.

## Guest control plane

When MULTIPORT is negotiated, use the virtio console control protocol. Wait
for `DEVICE_READY` before announcing ports. Port 0 receives `PORT_ADD`, its
console designation, and open-state messages, but no name. Named ports receive
`PORT_ADD`, `PORT_NAME`, and open-state messages in protocol order, respecting
per-port readiness. Names are NUL-terminated on the control wire. Closed
directions carry no usable data traffic.

The guest-visible behavior and lifecycle are independent of whether the
machine supplies local services or process proxies.

## Terminal resize

`--console-resize ENDPOINT` supplies a private Unix socket on Linux/macOS or a
named pipe on Windows through host services conforming to the Machine
interface. Each connection carries one four-byte update: nonzero columns followed by nonzero rows, both
little-endian u16 values. The receiver closes after consumption. The sender
retains the connection until EOF, subject to a bounded deadline. There is no
reply payload or history. Closing this connection does not close console
input; Madrillo does not interpret or relay the resize contract.

Resize enables `VIRTIO_CONSOLE_F_SIZE` and `VIRTIO_CONSOLE_F_MULTIPORT`, even
without named ports. Deliver standard `VIRTIO_CONSOLE_RESIZE` messages for
port 0 after guest readiness. Retain only the latest unsent size. A partial
client occupies one four-byte buffer for at most one second. Teardown removes
the listener. Dimensions follow the protocol without guest-kernel heuristics.
