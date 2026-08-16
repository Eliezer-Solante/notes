![[Pasted image 20260731152324.png]]


![[Pasted image 20260731153528.png]]


![[Pasted image 20260731153809.png]]

create a persistent volume, so whenever the container is deleted, the data will persist inside the persistent volume
![[Pasted image 20260731153938.png]]

if you have already a external data then you want the database to write there, then use bind mount
Volume mount - from /var/lib/docker/volumes
Bind mount - from any host path

instead of using `-v` tag, use `--mount`
![[Pasted image 20260731154444.png]]
In the context of `--mount` (and the equivalent in `-v`), they describe **the two ends of the mapping** — where the data physically lives, and where the container sees it.

**`source`** — the location on the **host** (outside the container). This is where the actual files physically live on your machine's disk. For a bind mount, it's a real path like `/opt/data`. For a named volume, it's the volume's name (Docker resolves that internally to somewhere under `/var/lib/docker/volumes/`, but you just refer to it by name).

**`target`** — the path **inside the container** where that host storage gets attached. This is the path your application (MySQL, in this case) actually reads and writes to, from its own point of view.

**The mental model:** think of it like plugging in an external hard drive.

- `source` = the actual drive, sitting on your desk (the host)
- `target` = the folder path where that drive gets mounted/attached inside the container's filesystem

So in your command:

bash

```bash
--mount type=bind,source=/opt/data,target=/var/lib/mysql
```

- `source=/opt/data` → "here's where the real files live on the host"
- `target=/var/lib/mysql` → "and inside the container, make them appear at the exact path MySQL expects to find its data"




![[Pasted image 20260731154617.png]]
![[Pasted image 20260731154741.png]]

![[Pasted image 20260731154902.png]]

### Where Docker stores files

By default, everything Docker manages lives under `/var/lib/docker/` on Linux (classic storage) — images, containers, volumes, networks, build cache, all of it, organized in subdirectories per component. You can check where it's pointed with:

bash

```bash
docker info | grep "Docker Root Dir"
```

You can relocate this (e.g. to a bigger disk) by setting `"data-root"` in `/etc/docker/daemon.json`:

json

```json
{ "data-root": "/mnt/bigdisk/docker" }
```

**Important shift with the containerd image store (more on this below):** image and layer data moves to `/var/lib/containerd/` instead, since containerd manages its own content store separately from the legacy Docker directory structure.

### Layer reuse

This is the caching mechanism we touched on earlier, worth being precise about. Every Dockerfile instruction produces a layer identified by a **content hash** of its inputs (the instruction plus the preceding layer's hash). If you build two different images that happen to share the same base layers — say, both `FROM node:20` — Docker doesn't duplicate that data on disk. It stores the layer once and both images just reference the same hash.

This is also why `docker build` cache works the way it does: if an early instruction (and everything it depends on) hasn't changed since the last build, Docker reuses the cached layer instead of re-executing it. This is a strong argument for structuring your Dockerfile so rarely-changing instructions (installing OS packages, dependencies) come before frequently-changing ones (your app source).

### Copy-on-write (CoW)

This is the mechanism that makes layer reuse _safe_ even though layers are shared. When a container modifies a file that exists in one of the read-only image layers below it, the storage driver doesn't touch the original — it **copies the file up into the container's writable layer first, then applies the change there**. The underlying image layer, and every other container sharing it, is completely unaffected.

This is efficient because:

- Nothing is copied until you actually write to it (lazy).
- Only the modified files get copied, not the whole layer.
- Deleting a file works via a special marker (a "whiteout" file) that tells the union filesystem "hide this file from the layers below" — the original bytes in the read-only layer are untouched.

This is the same principle as copy-on-write in memory management or ZFS/Btrfs snapshots — share until someone writes, then fork just that piece.

### Persistent volumes: `-v` vs `--mount`

Both attach external storage to a container, but they differ in syntax and behavior:

**`-v` / `--volume`** — older, terser, colon-separated:

bash

```bash
docker run -v mydata:/app/data myimage        # named volume
docker run -v /host/path:/app/data myimage    # bind mount
docker run -v /app/cache myimage              # anonymous volume
```

Quirk to know: if the source (`mydata` or `/host/path`) doesn't exist yet, `-v` will **silently create it** — which is convenient but can also mask typos (mistype a host path and you silently get a fresh empty directory instead of an error).

**`--mount`** — newer, more explicit, comma-separated key=value:

bash

```bash
docker run --mount type=volume,source=mydata,target=/app/data myimage
docker run --mount type=bind,source=/host/path,target=/app/data myimage
docker run --mount type=tmpfs,target=/app/cache myimage
```

`--mount` will **error out** if the source doesn't exist (for bind mounts), which is safer for scripts/production. It also supports options `-v` can't express as cleanly, like `readonly`, `bind-propagation`, or `volume-subpath`.

**Rule of thumb:** Docker's own docs recommend `--mount` for anything beyond quick local testing — it's more explicit about intent and fails loudly instead of quietly doing the wrong thing.

### Storage drivers

These are what actually implement the copy-on-write layering we just discussed, on the classic (pre-containerd) storage path. Currently:

|Driver|Status|
|---|---|
|`overlay2`|Default and recommended for virtually all current Linux distros — best overall stability/compatibility|
|`fuse-overlayfs`|Only needed for rootless Docker on kernels older than 5.11; modern kernels use `overlay2` in rootless mode too|
|`btrfs`|Available if your backing filesystem is Btrfs — supports snapshots, more setup/maintenance|
|`zfs`|Available if backing filesystem is ZFS — good for high-density workloads (PaaS-style), heavier memory use|
|`vfs`|Debugging/testing only — no copy-on-write at all, very poor performance, not for production|

**Removed:** `aufs`, the legacy `overlay` driver, and `devicemapper` have all been dropped from current Docker documentation and are no longer viable choices going forward — `overlay2` fully superseded them once modern kernels made it universally available.

Check what you're currently running:

bash

```bash
docker info | grep "Storage Driver"
```

### The containerd image store

This is the newest structural change, and it's worth separating clearly into two eras:

**Classic (pre-v29)** — the architecture we've been describing: Docker's own "graph driver" layer (the `overlay2`/`btrfs`/`zfs` drivers above) manages images and layers directly, storing everything under `/var/lib/docker`. This is what all the storage-driver discussion above applies to.

**Engine v29.0+ fresh installs** — Docker now defaults to the **containerd image store** instead. Rather than Docker's own graph driver code managing layers, it hands that job off to containerd's native content store and **snapshotter** framework — the same subsystem Kubernetes has used for years. Practically:

- Data moves from `/var/lib/docker` to `/var/lib/containerd`.
- `docker info` now shows something like `driver-type: io.containerd.snapshotter.v1` instead of a classic driver name.
- It enables features the classic path couldn't: proper multi-platform image support, build **attestations** (SBOM/provenance), and generally tighter alignment with the wider containerd/OCI ecosystem.
- Known trade-off: it typically uses **more disk space**, since it can store both compressed and uncompressed layer data.
- Known limitation: **incompatible with `userns-remap`** (user namespace remapping) as of now — if your security setup depends on that, you can't use it yet.

**If you upgrade an existing host to v29 rather than installing fresh, you are not force-migrated** — your daemon keeps using whatever classic storage driver it already had. The containerd image store is only the _default for brand-new installs_. You can also explicitly opt out on a fresh install:

json

```json
{ "features": { "containerd-image-store": "disable" } }
```

Worth knowing for planning purposes: Docker has stated the classic graph driver backend **will be removed in a future release** — so the classic drivers table above is describing something on a deprecation path, not a permanent fixture. If you're setting up new infrastructure today, it's worth at least testing on the containerd image store rather than assuming `overlay2` will be the default indefinitely.





<mark style="background: #D2B3FFA6;">Additional commands :</mark>

to run `.sh` files use `bash file.sh`

