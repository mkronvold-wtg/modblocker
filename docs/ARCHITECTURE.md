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

### Pause container

The main container is `registry.k8s.io/pause`, which keeps the pod alive after the one-time host mutation succeeds. This keeps the DaemonSet converged without re-running the mutation loop continuously.

## Data flow

1. Kubernetes schedules the DaemonSet pod on a node.
2. The init container mounts `/` from the host and updates `/host/etc/modprobe.d/disable-copyfail.conf`.
3. The init container invokes the host `modprobe -r` for `algif_aead`, `esp4`, `esp6`, and `rxrpc` when those modules are loaded.
4. If Dirty Frag rules were newly added or Dirty Frag modules were removed, the init container runs `sync` and writes `3` to `/proc/sys/vm/drop_caches`.
5. The init container exits successfully.
6. The pause container stays running until the pod is rescheduled or replaced.

## Failure model

- If the host shell or module tools are missing, the init container fails loudly and Kubernetes retries the pod.
- If one of the targeted modules is already absent, that unload step becomes a no-op.
- If `esp4`, `esp6`, or `rxrpc` are actively in use and cannot be removed, the pod fails so the node remains visibly non-compliant.
- If a new node joins the cluster, the DaemonSet repeats the process on that node automatically.
