# Project goal

This repository manages Kubernetes-hosted mitigations for CopyFail and Dirty Frag on immutable VKS worker nodes.

# Technical constraints

- Worker nodes are rebuilt from a base image, so manual host changes do not survive node replacement, upgrades, remediation, or cluster recreation.
- The mitigation must be reapplied automatically when a node joins the cluster.
- The current target modules are `algif_aead`, `esp4`, `esp6`, and `rxrpc`.

# Implementation requirements

- Keep deployable Kubernetes manifests under `manifests/`.
- Use a privileged DaemonSet to write a host modprobe configuration file, unload the target modules, apply the Dirty Frag page-cache cleanup step during remediation, and continuously report compliance health from the steady-state container.
- Make host changes idempotent so repeated pod starts do not duplicate configuration.
- Keep documentation under `docs/`.
- Minimize scope to the module-disable mitigation and the documentation needed to operate it.
