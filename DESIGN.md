# Dillo design

## Threat model

### THREAT-001 — Treat the host and guest as mutually adversarial

Dillo is intentionally designed for a mutually adversarial host and guest.
Neither party is assumed to trust the other. The goal is to make it easy to
protect the host from a malicious guest and the guest from a malicious host.

Host protection requires treating guest inputs and device activity as
untrusted, limiting component authority and resource use, isolating device
execution, and failing safely. Guest protection must not depend on the host
running an honest Dillo implementation.

### THREAT-002 — Require confidential computing for guest protection

Protecting the guest from a malicious host requires a confidential-computing
target, such as SEV-SNP, and its memory-protection and attestation mechanisms.
An ordinary `vm` target does not provide that protection. Attestation must be
appraised against the expected image, measured inputs, and security policy;
measurement alone does not establish that supplied content is trustworthy.

Unmeasured host inputs remain untrusted and must be constrained and validated
as required by PMI. A conformant-host requirement cannot replace guest or
verifier enforcement against a malicious host. The host can still deny service
by refusing to run or stopping the guest.

### THREAT-003 — Use PMI as the explicit launch and platform contract

PMI exists to make the guest's launch requirements and platform definition
explicit and portable across targets, with defined measurement and validation
rules. The image specifies the machine; the host realizes it or refuses the
launch. Host choices are limited to the authority PMI expressly delegates.

Dillo has no general host-controlled firmware configuration channel analogous
to `fw_cfg`. Platform and boot configuration must follow the PMI contract,
including its rules for measured inputs and validated host-supplied data.
Device I/O remains untrusted input; it must not become an implicit source of
trusted platform configuration.

## Design goal

Dillo realizes exactly the machine specified by the PMI: nothing more and
nothing less. Its central implementation choice is one concrete `Machine`
instance, whose trait interface and associated types define the contract that
the composed system conforms to. Startup selects that machine, applies the PMI
model, and boots. A variance from this design is a bug.

This document defines the design goal. Implementation findings and work plans
belong in issues; review procedures belong in [CONTRIBUTING.md](CONTRIBUTING.md).

## PMI launch contract

### PMI-001 — Apply the selected target faithfully

The PMI specification owns image interpretation, target semantics, action
ordering, memory contents, CPU requirements, and architecture-defined boot
state. Dillo must implement those semantics for the selected target and refuse
unsupported requirements before guest execution. It must not infer a different
launch contract from a particular image producer or bootloader.

The contract includes the selected target and its extensions, including the
base devicetree and explicitly delegated host choices. Host configuration may
exercise those choices only within their specified bounds. Where PMI expressly
permits variation, such as additional CPU features on the `vm` target, that
permission is part of the contract.

## Machine selection and realization

A **host backend** realizes the machine through a hypervisor API and host OS
services. The **machine model** is the guest-visible machine prescribed by PMI.
Devices and transports implement that model using the selected machine's
services.

### MODEL-001 — Select one concrete machine

One concrete `Machine` instance defines the interface contract for each VM's
realization and execution. Its host-backend implementation family is selected
once; individual components must conform to that family and its service
semantics. Devices belong to that machine, though the instance need not
construct them or directly hold every host resource.

### MODEL-002 — Supply host-specific primitives through associated types

`Machine` defines associated concrete types and service interfaces for
host-specific primitives, including CPU and memory access, interrupts, notifications,
execution, and confinement. These form the implementation family used by
shared devices and transports.

Associated types capture actual host differences and enable reuse of shared
behavior. The trait and its associated interfaces may evolve while preserving
this responsibility; device and transport protocols remain shared.

### MODEL-003 — Select, model, then boot

Startup selects a concrete machine instance, composes a faithful realization
of the PMI model conforming to its interface, and starts guest execution only
after realization succeeds.

This is a composition and conformance requirement, not a universal constructor
order. Devices and host resources may be constructed independently, and device
processes may be forked early, including before the machine instance exists.
Their attachment, services, execution, and lifecycle must conform to the
selected Machine interface and associated types. The design does not require
all constructors or process creation to be methods on a live Machine instance.

Architecture and target support are expressed through this same composition.
Adding a host platform or PMI target preserves the sequence and the ownership
of model semantics by PMI.

### MODEL-004 — Bind devices and resources to one machine instance

Each device belongs to one concrete machine instance and obtains guest-memory
access only through that relationship. Memory borrows, interrupts,
notifications, and attachments are bound to their owning instance. Backend
type compatibility does not authorize using one machine's resources in another.
Independently constructed devices and early-forked processes must establish
this binding before obtaining guest-memory access.

## Devicetree hardware model

### DT-001 — Construct the declared machine

The PMI devicetree contract governs guest-visible hardware and resource
relationships. Dillo constructs that model using the selected machine's
primitives. Backend defaults must not independently define the hardware.
Passing the tree to the guest is an additional use of the contract.

### DT-002 — Realize declarations faithfully

Required declarations must be faithfully realized or explicitly refused.
Honor declared addresses, extents, topology, controllers, and semantics,
including nonstandard values wherever the backend permits them. A backend
limitation must not silently change the model.

### DT-003 — Introduce no undeclared hardware resources

Every guest-visible device, mapping, interrupt relationship, and address-space
exclusion must have authority in the contract, including implicit effects of
backend APIs. A reserved range alone does not authorize arbitrary VMM use.

### DT-004 — Preserve contract ownership

Host allocations, overlays, and derived child resources must preserve the
contract's bounds and relationships. Address containment alone does not
authorize arbitrary child hardware.

Each attached device specifies its bus. Dillo supplies no global bus default
and does not substitute another transport when a bus is absent or full. An
optional numeric slot pins the exact position; otherwise allocation uses the
lowest free slot on that bus, reserving explicit slots first. Unavailable
buses, exhausted capacity, and conflicting assignments are errors.

### DT-005 — Preserve authority in both directions

Every required declaration must lead to its faithful realization, and every
realized resource must trace back to its authority. Parsing, claim coverage,
and provenance labels alone do not establish conformance.

### DT-006 — Discover virtio-console through its transport

Virtio-console is discovered through virtio-mmio or PCI. The devicetree may
declare those transports; it does not describe virtio-console devices or their
ports. Platform UART and early serial devices remain devicetree-governed.

The [console design](deps/dillo-virtio-console/DESIGN-named-ports.md) defines
its external behavior.

### DT-007 — Drain the setup structure completely and exactly once

Machine setup uses a draining representation of the DTB. Each node is claimed
by exactly one responsible handler, which consumes its properties and children
or explicitly delegates their handling. Already-consumed entries cannot be
claimed again. Every entry must be accounted for, including structural and
informational entries whose handling does not create hardware.

Setup succeeds only when the entire structure is drained. Unsupported or
unhandled entries cause an explicit failure; removing an entry merely to make
the structure empty is not handling its semantics. Claimed declarations must
remain connected to their realized resources. Exactly-once handling establishes
coverage, while DT-002 and DT-005 require faithful realization.

## Independent devices and transports

### COMPOSE-001 — Share device and transport behavior

Devices and transports are independent, composable, host-backend-agnostic
implementations. The PMI model governs their composition. Both use primitives
conforming to the selected Machine interfaces; a host backend does not
substitute its own implementation of a shared protocol.

### COMPOSE-002 — Preserve responsibility boundaries

Keep machine realization, attachment, transport behavior, and device behavior
separate, including nested PCI and virtio composition. Host differences belong
in the primitives these components use. The [network design](deps/dillo-virtio-net/DESIGN.md)
applies these boundaries to host connectivity.

### COMPOSE-003 — Separate requirements from authority

A component's resource requirements do not authorize their allocation. The
owner admitting the component must enforce the machine contract, including
resource identity, bounds, relationships, and conflicts on every admission
path. Dispatchers route admitted accesses without independently defining the
hardware model.

### COMPOSE-004 — Expose narrow component services

Devices and transports receive only the services and authority required for
their work, rather than general control of the machine. This applies to guest
memory, interrupts, notifications, execution controls, and host resources.
Nested composition may preserve or narrow authority; it must not widen it or
grant access to another component's resources.

## Guest address-space authority

### MEMORY-001 — Keep full address-space access inside the machine

The machine owns and allocates all guest memory and alone manages the full
guest address space. This allocation and mapping authority does not override
the selected target's memory protections or permit host access to confidential
guest-private contents. Every other component, including devices, transports,
and auxiliary workers, only borrows access appropriate to its work. A borrow
constrains the permitted ranges, operations, and lifetime; it does not transfer
ownership or confer allocation authority. Combining or forwarding borrows must
not expand the recipient's authority.

Devices may request additional regions as their work requires. The machine
validates each request under the PMI and device-protocol contracts and supplies
a bounded borrow when permitted. Regions need not be fixed at construction or
assigned as a permanent partition per device. Guest-supplied addresses and
component requests are inputs to validation, not self-authorization. Checking
that an address falls inside mapped RAM alone does not establish permission.

These limits apply to local services and process proxies alike. Early forks
must not leave device processes with unrestricted inherited guest mappings or
handles. Reset, teardown, mapping changes, and reclamation must account for
outstanding borrows and prevent access beyond their valid lifetime. The machine
retains ownership throughout. Borrowing is an access and lifecycle contract;
its implementation may use local handles or IPC rather than Rust references.
Neither process separation nor an opaque handle alone proves that access is
limited.

### MEMORY-002 — Let the machine place device-requested allocations

For new guest-memory allocations, devices express their requirements and the
machine chooses placement within the PMI contract. The machine retains
ownership and supplies the device with bounded access. A request must not
silently enlarge guest RAM or introduce an unauthorized guest-visible region.

Allocation requests and access to existing guest memory are distinct. Where a
device protocol lets the guest supply addresses, the machine validates and
mediates borrows of those existing regions. In particular, virtio drivers
allocate queue memory and supply queue and descriptor-buffer addresses;
Machine placement must preserve those protocol semantics. The allocation and
borrowing interfaces must support both cases without changing guest behavior.

### MEMORY-003 — Treat borrowed guest contents as concurrently mutable

A borrow guarantees permitted access to valid backing for its lifetime; it
does not imply exclusive access or stable contents. Read-only access limits
the borrower, not the guest's ability to write. Devices must treat guest-owned
contents as untrusted and potentially changing during an operation, and must
not rely on validation of a value that they subsequently reread unchecked.
Synchronization and memory-ordering behavior must respect the device protocol.

## Process model and confinement

### PROCESS-001 — Separate device execution where supported

The device model uses out-of-process execution wherever the host backend
supports the required separation. In-process execution is a fallback for host
limitations. It is not a convenience alternative to supported isolation.

### PROCESS-002 — Conform execution to machine interfaces

`Machine` and its associated host-specific types define the interfaces for
process establishment, communication, resource transfer, notification, and
lifecycle control. Backend-specific setup may occur before instance construction.
Shared devices and transports use these services without choosing OS-specific
mechanisms or assuming a shared address space. Auxiliary device work is
subject to the same execution contract. Early process creation must preserve
these contracts when components are composed with the machine.

### PROCESS-003 — Preserve semantics across process boundaries

Common interfaces accommodate in-process implementations and out-of-process
proxies with the same logical composition and service semantics. Local Rust
objects may stay local behind a proxy; shared address-space objects alone do
not establish process portability.

### PROCESS-004 — Constrain component authority

Each component receives only the resources and operations needed for its work.
Process separation includes confinement. Linux device processes use strict
seccomp filters established by the Linux machine family. Other host backends
provide their own confinement mechanisms. Shared interfaces express service
requirements rather than OS-specific sandbox configuration.

### LIFECYCLE-001 — Keep reset and teardown coherent

Machine-compatible lifecycle interfaces coordinate shutdown, reset, and failure
handling across components, including independently constructed ones. Stop or
finish old device work before changing memory or controller state that it can
access. Device, transport, and execution state must agree across activation,
reset, and reactivation. Reboot establishes the target's required launch state
without stale work from the previous execution. Teardown releases owned host
resources and reports failures.

### LIFECYCLE-002 — Fail the VM on unexpected device execution failure

Unexpected device-process death, worker panic, or unrecoverable execution
failure must reach the VM supervisor, stop guest execution, and end the VM
with a reported failure. Dillo must not automatically respawn the device,
reset it for another attempt, or restart the VM to recover from that failure.
Containment must not give a malicious guest an unattended retry loop.

Expected protocol outcomes, such as a request error, normal endpoint EOF, or
an orderly shutdown, are distinct from execution failure. Their handling must
be explicit. Teardown must preserve memory and resource authority while
stopping remaining work; errors must not be hidden as successful completion.

## Platform and project scope

### PLATFORM-001 — Support Linux, macOS, and Windows hosts

Host-specific mechanisms remain behind the machine boundary. Supported
architecture/target combinations and native validation results are documented
separately from the design goal.

### GUEST-001 — Support Linux guests

Dillo targets Linux guests. Other guest operating systems are outside its
compatibility requirements.

### PLATFORM-002 — Target modern virtualization

Dillo targets modern virtualization, including Intel TDX, AMD SEV-SNP, and Arm
CCA where available, with shared behavior across targets. General-purpose
legacy compatibility is outside its scope. Ordinary Intel KVM execution must
not depend on the restricted-guest scratch-TSS workaround.

Dillo owns VM execution and the device model. Pichi owns image management and
orchestration. PMI owns the image-format and launch specification. Changes to
shared contracts require coordination with their owners; conformance work
must not invent new cross-project contracts.
