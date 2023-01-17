# kopiatomic

Make atomic `kopia` backups using zfs snapshots.

## Description

This script creates `kopia` backups that capture all your files at (nearly) the same moment in time.
To do so, it takes one or more zfs datasets, snapshots each one, `nullfs`-mounts them together at a temporary mount point, and runs `kopia` on that filesystem.
Backing up from a zfs snapshot ensures that files don't change during the backup.

Because it uses zfs to take atomic snapshots and `nullfs` to assemble filesystems, the script supports only zfs datasets on FreeBSD.
It's written in POSIX shell script and has no dependencies apart from `kopia`.

## Quick Start

To back up zfs datasets, connect to a `kopia` repository and run `kopiatomic`, specifying the datasets you want to back up:

```sh
kopia repository connect from-config --file="repository.config"
kopiatomic zroot/usr/home zroot/usr/src
```

You can optionally process child datasets recursively with `-r` or process all mounted datasets with `-a`.

For detailed usage information and a list of options, refer to the script help text available with `kopiatomic -h`.

## Do I Need Atomic Backups?

Maybe.
Depending on how long `kopia` takes to run and what else the machine is doing, the filesystem might change a lot between the first and the last file that `kopia` backs up.
Related files like sidecars and multipart archives could be out of sync and appear corrupt when you restore from the backup.

That said, this drift might not be a problem.
Unless a file changes very frequently, it's likely that it gets backed up correctly the next time `kopia` runs.
In that case, backup integrity is OK, but the backup might provide a recovery point objective one snapshot further back than it appears.

For general, non-database use, `kopiatomic` can make your `kopia` backups more consistent at the cost of a little complexity.
Note that it's not a robust solution for data where atomic backups are critical.

One limitation to be aware of: `kopiatomic` doesn't make precisely atomic snapshots across datasets.
Files within each dataset are captured atomically, but datasets might be captured at slightly different times, because `zfs snapshot` command can take a few milliseconds to run.

## Acknowledgements

I built on some of the work in this blog post by Arsen Arsenović: [Unattended backups with ZFS, restic, Backblaze B2 and systemd](https://www.aarsen.me/posts/2022-02-15-sweet-unattended-backups.html). Note that he uses a patched version of restic. I tried this approach with `restic`, but I'm not aware of a way to use `restic` with zfs snapshots.

