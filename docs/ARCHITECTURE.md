# Architecture

## Repository layout

- `.github/copilot-instructions.md`: normalized project guidance derived from the original input.
- `manifests/daemonset.yaml`: the node-mitigation workload.
- `manifests/kustomization.yaml`: the entry point for applying the manifest set.
- `docs/DESIGN.md`: design rationale and constraints.
- `docs/ARCHITECTURE.md`: component overview and operating model.

## Runtime components

### DaemonSet

The DaemonSet schedules one pod per schedulable Linux node in `kube-system`.

### Init container

The init container runs as privileged and mounts the host root filesystem at `/host`. It performs two actions:

1. Write the module-disable configuration into the host's `modprobe.d` directory.
2. Use `chroot /host` to invoke the host shell and unload `algif_aead`, `esp4`, `esp6`, and `rxrpc` when they are active.
3. Drop the host page cache after Dirty Frag remediation is newly applied.

### Monitor sidecar

The second container is a lightweight BusyBox monitor. It mounts the host root read-only and an internal `emptyDir` status volume. On a loop, it validates the required lines in `/host/etc/modprobe.d/modblocker.conf`, confirms the target modules are absent from `/proc/modules`, writes a compliance flag when everything is still correct, and records a heartbeat timestamp for liveness.

### Health probes

- The readiness probe checks for the shared healthy flag, so a node that drifts out of compliance becomes visibly unready.
- The liveness probe checks that the heartbeat file is still fresh, so a stalled monitor process gets restarted without conflating loop failure with node non-compliance.

### Security posture

- The pod disables automatic service account token mounting and Kubernetes service link injection.
- The monitor sidecar runs as an explicit non-root user, drops all Linux capabilities, disallows privilege escalation, uses a read-only root filesystem, and uses the runtime-default seccomp profile.
- The init container remains privileged and unconfined only because it must write to the host filesystem, unload host kernel modules, and run the cache-drop remediation step.

## Data flow

1. Kubernetes schedules the DaemonSet pod on a node.
2. The init container mounts `/` from the host and updates `/host/etc/modprobe.d/modblocker.conf`.
3. The init container invokes the host `modprobe -r` for `algif_aead`, `esp4`, `esp6`, and `rxrpc` when those modules are loaded.
4. If Dirty Frag rules were newly added or Dirty Frag modules were removed, the init container runs `sync` and writes `3` to `/proc/sys/vm/drop_caches`.
5. The init container exits successfully.
6. The monitor sidecar starts its validation loop, writes readiness state into `/status/healthy`, and updates `/status/heartbeat`.
7. Kubernetes uses those files for ongoing readiness and liveness checks.

## Failure model

- If the host shell or module tools are missing, the init container fails loudly and Kubernetes retries the pod.
- If one of the targeted modules is already absent, that unload step becomes a no-op.
- If `esp4`, `esp6`, or `rxrpc` are actively in use and cannot be removed, the pod fails so the node remains visibly non-compliant.
- If the managed config is removed or one of the target modules gets reloaded later, the monitor sidecar makes the pod unready.
- If the monitor loop wedges or exits unexpectedly, the liveness probe fails and Kubernetes restarts that container.
- If a cluster policy forbids privileged init containers, hostPath mounts, or unconfined seccomp, the DaemonSet will be rejected instead of silently weakening the mitigation.
- If a new node joins the cluster, the DaemonSet repeats the process on that node automatically.
