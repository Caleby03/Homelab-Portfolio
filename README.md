# Homelab

A three-node homelab run as infrastructure-as-config: an always-on Dell OptiPlex
7050 Micro hosting the stateful container stack, a Windows gaming desktop
(Ryzen 7 3700X, RTX 2070 Super 8GB, 64GB RAM) used as burst GPU capacity, and a
Raspberry Pi as an independent watchdog. All inter-node traffic runs over
Tailscale; nothing is exposed to the LAN or the public internet.

The point of the project is not the hardware. It is that every node is assigned
work it is actually suited to, every container is capacity-bounded, and the
monitoring is designed around the assumption that any given node will fail —
including the node doing the monitoring.

---

## Architecture

|  | Pi (Node C) | OptiPlex (Node B) | Desktop (Node A) |
|---|---|---|---|
| **Defining trait** | ~5W, never off | i7, 16GB, always on | 8GB VRAM, 64GB RAM, **unreliable** |
| **Failure mode** | SD card death | OOM, disk fill | You start a game |
| **Good at** | Watchdog, DNS | Stateful services | Burst compute |
| **Bad at** | Anything heavy | Sustained CPU/GPU | Being available |

### Why the work is split this way

**The desktop is unreliable by design.** It is a gaming machine first, so it
disappears without warning. That is treated as an input, not a defect: nothing
stateful is placed on it, it owns no queue, and no alert fires when it vanishes.
Alerting on a node that is *expected* to go offline just trains alert-blindness.
What it is genuinely good at is burst GPU work, so that is all it gets —
inference and headless browser workers, both of which can fail and be retried.

**The OptiPlex holds anything that must survive a reboot.** Databases, the queue
broker, dashboards, bookkeeping. It is the only node with a real availability
guarantee, so it is the only node trusted with state.

**Headless browser work runs on the desktop, orchestrated from the OptiPlex.**
Each Chromium worker costs 300–500MB. Running a scraping pipeline on a 16GB box
alongside the databases would flatten it. Splitting orchestration from execution
means the small always-on node keeps the queue and the state, while the big
unreliable node does the expensive, interruptible part.

**The Pi exists to watch the others.** ~5W, separate power draw, separate
storage, separate failure modes — see ADR-0001.

---

## What is running

Containerized on the always-on node, managed through Portainer:

| Role | Component |
|---|---|
| Container management | Portainer |
| Uptime monitoring and alerting | Uptime Kuma |
| Service dashboard | Homepage |
| Relational database | PostgreSQL, with pgAdmin |
| Queue broker | Redis |
| Double-entry bookkeeping | Manager.io |

Metrics collection and time-series dashboards (Prometheus, Grafana, cAdvisor and
the host/GPU exporters) are deployed but not yet configured — see Build status.

---

## Operational practice

Conventions applied across every compose file in the private config repo:

- **Every container has a `mem_limit`.** A 16GB host with unbounded containers
  is an OOM waiting to happen. The monitoring stack is capped at ~1.1GB total,
  sized deliberately so it fits alongside the services it observes.
- **Image tags are pinned to explicit versions**, never `latest`. A rebuild
  reproduces the same stack rather than silently pulling a new major version.
- **Config is bind-mounted, state lives in named volumes.** Config is in git;
  state deliberately is not. The repo plus a Tailscale login is the rebuild path.
- **Reachability is a firewall property, not a bind-address property.** Docker
  continues to listen on all interfaces so containers can reach one another and
  the host; the packet filter is what decides who gets through. See Network
  security model.
- **Services address each other by Tailscale MagicDNS name, not IP.** This is
  what makes a service portable between nodes: moving it becomes a volume copy
  rather than a config rewrite. The one exception is Docker port bindings, which
  require a literal address at bind time.
- **Secrets never reach the repository.** `.env` is gitignored, `.env.example`
  documents every variable with placeholder values, and `gitleaks` runs against
  staged changes before every push.

---

## Network security model

The design goal is that nothing is reachable without authenticating first, and
that being on the same physical network grants nothing at all.

- **Tailscale is the only transport between nodes.** A WireGuard-based mesh
  carries all inter-node traffic. There is no port forwarding on the router, no
  reverse proxy exposed to the internet, and no public ingress of any kind.
- **Access is authenticated per device, not per network.** A machine reaches
  services because it has authenticated onto the tailnet — not because it
  happens to sit on the right LAN. Revoking a device removes its access
  everywhere at once, rather than requiring a firewall change.
- **The firewall is the enforcement boundary.** Docker deliberately still
  listens on all interfaces, so containers and the host can talk to each other
  without special-casing every service. External traffic is filtered in the
  `DOCKER-USER` chain — evaluated ahead of Docker's own accept rules — so only
  packets arriving over the Tailscale interface reach a published port. Host
  SSH is restricted to the same interface via UFW.
- **The desktop is firewalled to the tailnet range** for the exporter ports it
  publishes, so an exporter cannot be scraped from anywhere else.
- **Secrets stay out of version control.** `.env` is gitignored, `.env.example`
  documents every required variable with placeholder values, and `gitleaks` is
  run against staged changes before every push.

### Closing a real gap: Docker's firewall bypass

The tailnet-only property was originally enforced by convention — every
published port was expected to carry an explicit bind address. Auditing it
showed that convention was not holding, and published container ports were
reachable from the LAN.

The cause is a well-known and easily-missed Docker behaviour: publishing a port
inserts iptables rules that are evaluated *before* the chain UFW manages. A
`deny incoming` default policy therefore does not block a published container
port. UFW reports the host as locked down while Docker has already accepted the
traffic.

That is the worst class of security bug in a homelab — not an open port, but a
system that reports itself as closed while it is open, so everything built on
top of it inherits a false assumption. The fix was to stop treating bind
addresses as a security control and put enforcement in the packet filter
instead: rules in the `DOCKER-USER` chain, which is evaluated ahead of Docker's
own accept rules, admit only traffic arriving over the Tailscale interface.
Host SSH was locked to that same interface with UFW. Containers still listen
broadly so they can reach one another and the host — reachability is decided at
the filter, not at the socket.

The tradeoff is deliberate and worth stating: enforcement is now a single
control rather than layered. Because the listening sockets remain open on all
interfaces, the firewall rules are load-bearing, which makes their persistence
across reboots and Docker restarts a first-class operational concern rather
than a detail.

Recorded as ADR-0002, with a runbook covering verification — which has to be
done from a LAN host that is not on the tailnet, since checking from the host
itself passes trivially and proves nothing.


Planned hardening: single sign-on in front of the web interfaces, tailnet ACLs
restricting which node may reach which service, and network isolation for agent
runtimes so a runtime cannot reach the rest of the stack.

---

## Architecture decisions

Decisions are recorded as ADRs at the time they are made, rather than
reconstructed afterwards. The configs show *what* runs; the ADRs record *why*.

**ADR-0001 — monitoring moved off the box it monitors.** Uptime Kuma was
originally installed on the OptiPlex, simply because that is where everything
else already lived. That made the OptiPlex a single point of failure for knowing
whether the OptiPlex was up: if it OOMs, fills its disk, or reboots for updates,
monitoring stops at the same instant the thing being monitored stops — and the
failure is silent, because the alerting is part of the outage. Alerting must not
share a failure domain with the thing it watches. Uptime Kuma therefore moves to
the Pi: an independent observer with its own power, storage and failure modes.

Prometheus, Grafana and Homepage deliberately stay on the OptiPlex. They serve a
different purpose — diagnosis after the fact, not notification in the moment.
Only the second needs to survive the outage. A dashboard being unreachable
during an outage is accepted: during an outage there is nothing to look at.

**Alerting deliberately excludes the desktop.** Alerting on a node expected to
disappear produces noise, and noise produces ignored alerts.

**A dedicated local AI node is deferred, with a defined trigger.** The hybrid
local + hosted inference routing already covers the work the 2070 Super cannot.
A dedicated GPU build is deferred until a specific spend threshold is crossed,
rather than bought speculatively — the discipline being that hardware must fix a
fault that exists now, or remove a ceiling actually being hit.

---

## Migrations

- **Uptime Kuma → Pi.** Planned move off the monitored host, per ADR-0001.
  Deliberately built on the OptiPlex first: the rebuild is cheap, doing it twice
  is useful practice, and because every monitor targets a MagicDNS name rather
  than an IP, the second pass is a volume copy rather than a rewrite.
- **OptiPlex board replacement and 8→16GB RAM upgrade.** Fixed a real fault and
  lifted a ceiling that was actually being hit, which is what justified it.
- **Docker firewall lockdown.** Audited the tailnet-only invariant, found
  published container ports reachable from the LAN because Docker bypasses UFW,
  and moved enforcement into the `DOCKER-USER` chain with SSH restricted via
  UFW. See ADR-0002.
- **Monitoring stack rollout.** Prometheus recording rules pre-compute the
  values the dashboard displays, because the dashboard widget passes PromQL in a
  URL query string and long expressions become an encoding problem. Collapsing
  each to a single short metric name keeps the dashboard config readable.

---

## Planned — agent orchestration

**Nothing in this section is running today.** It is the next phase of the build,
recorded here as direction rather than achievement.

The intent is a managed cluster of AI agents handling day-to-day operational
work — monitoring triage, routine maintenance, bookkeeping and document intake —
orchestrated from the always-on node, with GPU-heavy steps dispatched to the
desktop and hosted models used where local capacity falls short.

The placement logic is the same as everywhere else in this homelab: orchestration
and state on the reliable node, expensive interruptible compute on the unreliable
one. Agent runtimes are intended to be network-isolated from the rest of the
stack, so that a compromised or misbehaving runtime cannot reach the databases or
the queue broker.

---

## Build status

Built in public. This is a live project, so the page separates what runs today
from what is next — nothing below is listed as running unless it is.

**Running now.** Three-node role split with workloads placed by node
reliability. Tailscale-only mesh with no public ingress and no LAN exposure, enforced in the
firewall after an audit found Docker bypassing UFW (ADR-0002).
Containerized always-on stack — container management, uptime monitoring and
alerting, service dashboard, relational database, queue broker and bookkeeping —
every container under an explicit memory limit and a pinned image tag. Hybrid
local + hosted inference routing. Infrastructure-as-config repository with ADR
practice and secret scanning on every push.

**In progress.** Metrics collection and time-series dashboards, deployed but not
yet configured. Host and GPU exporters. API spend dashboard widget.

**Next.** Raspberry Pi watchdog node, and with it the ADR-0001 migration of
uptime monitoring off the host it monitors. DNS filtering. Workflow
orchestration and document management. Single sign-on and tailnet ACLs. The
agent orchestration cluster described above.

---

## Note on scope

This is the public view. The configuration itself, runbooks, network addressing,
port bindings and firewall rules live in a separate private repository.
