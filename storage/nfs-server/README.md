# nfs-server

> **NOTE: Security Consideration**
>
> This extension exports filesystems over the network. `rpc.mountd` (port 20048 by default)
> and the kernel NFS server (port 2049) listen on all interfaces. Only install it on nodes
> that are meant to export filesystems, and use host-level firewalling or Kubernetes network
> policies to restrict access.

This extension provides the `nfs-utils` server-side daemons, turning a Talos node into an
NFS server. The client-side tools live in the [`nfs-utils`](../nfs-utils) extension.

## What's Included

- **exportfs**: Syncs the kernel export table (`/var/lib/nfs/etab`) from `/etc/exports`
- **rpc.mountd**: Services NFSv3 `MOUNT` requests and the kernel's export upcalls
- **nfsdcld**: NFSv4 client tracking daemon, required for state recovery across restarts
- **rpc.nfsd**: Starts the in-kernel NFS server threads

## Requirements

- The **`nfsd`** extension, for the `nfsd` kernel module. Every service here waits on
  `/proc/fs/nfsd`, which only appears once that module is loaded.
- The **`nfs-utils`** extension, for `rpcbind` (RPC portmapper) and `rpc.statd` (NFSv3 lock
  state monitoring). `rpc.mountd` and `rpc.nfsd` depend on the `ext-rpcbind` service.

## Configuration

`/etc/exports` is supplied through an `ExtensionServiceConfig`, for both `exportfs` and
`rpc-mountd`:

```yaml
apiVersion: v1alpha1
kind: ExtensionServiceConfig
name: exportfs
configFiles:
  - content: |
      /var/mnt/nfs 10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash,fsid=0)
    mountPath: /etc/exports
---
apiVersion: v1alpha1
kind: ExtensionServiceConfig
name: rpc-mountd
configFiles:
  - content: |
      /var/mnt/nfs 10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash,fsid=0)
    mountPath: /etc/exports
```

Only paths under `/var/mnt` can be exported — that is the only host directory bind-mounted
into the `exportfs` and `rpc.mountd` containers. Use a Talos user volume mounted there.

## How It Works

Each tool runs as its own containerized extension service, started in this order:

1. **rpc.mountd**: Answers NFSv3 mount requests and services the kernel export upcalls
   (needed for NFSv4 too)
2. **exportfs**: One-shot, populates the kernel export table from the configured `/etc/exports`
3. **nfsdcld**: Attaches to the kernel's `nfsd/cld` pipe for NFSv4 client tracking
4. **rpc.nfsd**: One-shot, starts the kernel NFS server threads

That ordering mirrors upstream's systemd units, where `nfs-server.service` is ordered
`After=nfs-mountd.service nfsdcld.service` and runs `exportfs -r` as its `ExecStartPre`.
`rpc.nfsd` must come last: `nfsd4_client_tracking_init()` runs once, when the server
starts, so `nfsdcld` has to be attached by then.

Nothing on Talos mounts the `nfsd` or `rpc_pipefs` filesystems, and the daemons cannot
mount them themselves (`rpc.nfsd` only tries by shelling out to `/bin/mount`, which these
scratch rootfs images do not ship). Both are therefore declared as filesystem mounts in the
service definitions instead of host bind mounts.

State persists across reboots in `/var/lib/nfs`.

## Protocol versions

`nfs-utils` is built with `--enable-nfsv4 --enable-nfsv41`, and `rpc.nfsd` is started
without any `--no-nfs-version` flag, so NFSv3, v4.0, v4.1 and v4.2 are all served.

For NFSv4 the export table needs a pseudo-root — give the top-level export `fsid=0`, as in
the example above.

**Client tracking** is handled by `nfsdcld`. Talos kernels are built with
`CONFIG_NFSD_LEGACY_CLIENT_TRACKING` unset, so neither the usermode-helper nor the
`/var/lib/nfs/v4recovery` fallback is compiled in — `nfsdcld` is the only client-tracking
method the kernel can use. Without it `nfsd` still starts (its return value is discarded in
`nfs4_state_start_net()`) and logs `NFSD: Unable to initialize client recovery tracking!`,
but `nfsd4_client_record_check()` then returns `-EOPNOTSUPP` and NFSv4 clients cannot
reclaim opens or locks after a server restart.

Restarting `nfsdcld` on its own tears down the `rpc_pipefs` mount and the `nfsd/cld` pipe
with it, while `nfsd` bound its tracking ops at start — so a bare `nfsdcld` restart needs an
`rpc-nfsd` restart too. Upstream systemd behaves the same way.

**ID mapping** needs nothing: `rpc.idmapd` is not shipped, and the kernel server defaults
to `nfsd.nfs4_disable_idmapping=1`, so AUTH_SYS clients exchange numeric UIDs/GIDs.
Kerberos (`--disable-gss`) and NFS junctions (`--disable-junction`) are not built.

## References

- [exportfs man page](https://linux.die.net/man/8/exportfs)
- [rpc.mountd man page](https://linux.die.net/man/8/rpc.mountd)
- [rpc.nfsd man page](https://linux.die.net/man/8/rpc.nfsd)
- [nfsdcld man page](https://linux.die.net/man/8/nfsdcld)
- [exports(5)](https://linux.die.net/man/5/exports)
