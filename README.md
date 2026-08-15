# Homelab

A three-node homelab built around a deliberate split of responsibilities: an
always-on Dell OptiPlex 7050 Micro running stateful services, a Windows gaming
desktop used as burst GPU capacity, and a Raspberry Pi acting as an independent
watchdog. All inter-node traffic runs over Tailscale. The interesting part is
not the hardware — it is the constraint that each node is chosen for what it is
actually good at, and that the monitoring is designed around the assumption
that any given node will fail.

---

## Architecture

|  | Pi (Node C) | OptiPlex (Node B) | Desktop (Node A) |
|---|---|---|---|
| **Defining trait** | ~5W, never off | i7, 16GB, always on | 8GB VRAM, 64GB RAM, **unreliable** |
| **Failure mode** | SD card death | OOM, disk fill | You start a game |
| **Good at** | Watchdog, DNS | Stateful services | Burst compute |
| **Bad at** | Anything heavy | Sustained CPU/GPU | Being available |

The desktop being unreliable is a design input, not a defect. It is expected to
disappear whenever it gets used for gaming, so nothing stateful lives there and
nothing alerts on its absence.

---

## Selected decisions

Architecture Decision Records are kept with the configuration. They will be
published here as they are cleared for public release.

**ADR-0001 — Uptime Kuma runs on the Pi, not the OptiPlex.** Uptime Kuma was
originally installed on the OptiPlex simply because that is where everything
else already lived, which made the OptiPlex a single point of failure for
knowing whether the OptiPlex was up. Alerting that shares a failure domain with
the thing it watches goes silent at exactly the moment it is needed — the
outage takes down the notification path along with everything else. Moving Kuma
to a separate ~5W node with its own power draw, storage and failure modes turns
a silent outage into an alert.

---

## Status

Honest accounting of what is actually running today.

| | |
|---|---|
| Three-node hardware plan and role split | **Built** |
| OptiPlex node — board fix and RAM upgrade | **Built** |
| Tailscale mesh across the two active nodes | **Built** |
| Hybrid local + hosted LLM inference routing | **Built** |
| Infrastructure-as-config repo and ADR practice | **Built** |
| Prometheus / Grafana / Homepage monitoring stack | **In progress** — configs written and committed, not yet deployed |
| Host and GPU metrics exporters | **In progress** — configs written, not yet deployed |
| API spend dashboard widget | **In progress** |
| Raspberry Pi node | **Planned** — hardware not yet deployed |
| Uptime Kuma on the Pi (ADR-0001) | **Planned** — depends on the Pi |
| Pi-hole and DNS | **Planned** |
| Workflow orchestration and document management | **Planned** |
| Single sign-on | **Planned** |
| Rack enclosure and environmental sensors | **Planned** |
| Dedicated local AI node | **Planned** — deferred, possibly indefinitely |

Nothing above is marked Built unless it is running now. Items with written
configuration but no deployment are marked In progress, not Built.

---

## Note on scope

This is the public view. Configuration, runbooks, network addressing and the
full service inventory live in a separate private repository.
