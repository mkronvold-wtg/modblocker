# Project goal

This repository manages a Kubernetes-hosted mitigation for CopyFail on immutable VKS worker nodes.

# Technical constraints

- Worker nodes are rebuilt from a base image, so manual host changes do not survive node replacement, upgrades, remediation, or cluster recreation.
- The mitigation must be reapplied automatically when a node joins the cluster.
- The current target module is `algif_aead`.

# Implementation requirements

- Keep deployable Kubernetes manifests under `manifests/`.
- Use a privileged DaemonSet to write a host modprobe configuration file and unload the target module.
- Make host changes idempotent so repeated pod starts do not duplicate configuration.
- Keep documentation under `docs/`.
- Minimize scope to the module-disable mitigation and the documentation needed to operate it.
