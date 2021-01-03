# Introduction

Out of the box (that is, assuming the default config which ships with most distros) podman won't work with a read-only root (`/`) filesystem. That is because the graphroot needs to be writeable even when performing what one would expect to be read-only operations such as `podman ps` as they do need to mutate the state, as explained [here](https://github.com/containers/podman/issues/8253#issuecomment-723585257).

However, this doesn't stop us from running podman when the host rootfs is read-only. This guide will describe some ways to achieve that, each with its own set of pros/cons.

## Move the graphroot to a rw mount

This is the least exotic setup, not much changes.

Inside your `storage.conf` file, under the `[storage]` section, change `graphroot`. For example:

`/etc/containers/storage.conf`:

```
[storage]
graphroot = "/data/containers/storage"
```

## Move the graphroot to an ephemeral mount

This completely isolates podman state from the rest of the system. In the following example I'll be using a default tmpfs mount, `/run` but the same principle also applies to, for instance, a btrs or zfs dataset that is rolled back to am empty snapshot on boot. This is useful when you're launching fresh containers from systemd units and as such don't need to persist podman state.

Inside your `storage.conf` file, under the `[storage]` section, change `graphroot`. For example:

`/etc/containers/storage.conf`:

```
[storage]
graphroot = "/run/containers/storage"
```

Simply doing that means you are going to pull images in memory every time the host is rebooted. If this is not the desired behavior you can use the [additional image stores](https://www.redhat.com/sysadmin/image-stores-podman) feature, which works with either ro or rw mounts. 

This is very useful when you have control over the rootfs before it goes read-only, i.e. you're rolling A/B rootfs updates which boot read-only or simply because ext4 went read-only because of some unexpected io issue. 

Still in `storage.conf` you could do something like this:

```
[storage.options]
additionalimagestores = [
  "/images/containers/storage"
]
```

If `/images` is writeable, you can still manipulate by running `podman` with `--root` (there's an [issue](https://github.com/containers/podman/issues/7309) fixed in master but still not released when this was written which requires the use of `--storage-opt`): `podman --root=/var/lib/containers/storage --storage-opt=overlay.mountopt=nodev image pull alpine`.

Depending on your setup, there might still be a missing piece to the puzzle: volumes. This time you're going to change `containers.conf`:

`/etc/containers/containers.conf`:

```
[engine]
volume_path = "/data/containers/storage/volumes"
```

There's one caveat to this approach. Since we made podman state ephemeral, on the next boot `podman volume ls` wont show any volumes until the containers which created them are started, and manually created ones won't show up at all. There's also another [issue](https://github.com/containers/podman/pull/8254), fixed, waiting for release, which prevents re-creating the volume on subsequent boots.
 
Given that, you may want to use host mounts instead of volumes, at least for now.

## TODO

- Remove workarounds once v2.2.0 is released