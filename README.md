# kopiatomic

Make atomic `kopia` or `restic` backups using zfs snapshots.

## Description

These scripts create `kopia` or `restic` backups that capture all your files at (nearly) the same moment in time.
To do so, they take one or more zfs datasets, snapshot each one, `nullfs`-mount them together at a temporary mount point, and run `kopia` or `restic` on that filesystem.
Backing up from a zfs snapshot ensures that files don't change during the backup.

Because the scripts use zfs to take atomic snapshots and `nullfs` to assemble filesystems, they support only zfs datasets on FreeBSD.
They're written in POSIX shell script and have no dependencies apart from `kopia` and `restic`, respectively.

## Quick Start

To back up zfs datasets, connect to a `kopia` repository and run `kopiatomic`, specifying the datasets you want to back up:

```sh
kopia repository connect from-config --file="repository.config"
kopiatomic zroot/usr/home zroot/usr/src
```

Or, make sure environment variables for a `restic` repository are set and run `restomic`, specifying datasets:

```sh
restomic zroot/usr/home zroot/usr/src
```

Note that `restomic` uses the `device-id-for-hardlinks` feature flag, available starting in `restic` 0.17.
Without this option set, `restic` registers all directories as modified every time when backing up from a mounted ZFS snapshot.

You can optionally process child datasets recursively with `-r` or process all mounted datasets with `-a`.

For detailed usage information and a list of options, refer to the script help text available with `kopiatomic -h` or `restomic -h`.

## Recommended Setup

By default, `kopiatomic` runs `kopia snapshot <your datasets>` and `restomic` runs `restic backup <your datasets>` (with the `device-id-for-hardlinks` flag set).
To handle the initial connection or environment variables, you can make a wrapper script around `kopia` or `restic` and tell the script to run your wrapper instead of running the command directly.
The following examples show how to do so for each backup client.

### Wrapper Script for `kopia`
Create a script called `kopia-myrepo` as follows:

```sh
#!/bin/sh
kopia repository connect from-config --file="myrepo.config"
exec kopia "$@"
```

Store it somewhere on your `$PATH`, make it executable, and run `kopiatomic -c "kopia-myrepo snapshot" zroot/usr/home zroot/usr/src`

### Wrapper Script for `restic`

Create a script called `restic-myrepo` as follows (example for an S3 backend).

```sh
#!/bin/sh
export AWS_ACCESS_KEY_ID='12345'
export AWS_SECRET_ACCESS_KEY='12345'
export RESTIC_REPOSITORY='s3:example.com/my-repo'
export RESTIC_PASSWORD_FILE='/path/to/password/file'
export RESTIC_FEATURES='device-id-for-hardlinks'
exec restic "$@"
```

Store it somewhere on your `$PATH`, make it executable, and run `restomic -c "restic-myrepo backup" zroot/usr/home zroot/usr/src`.

After running restomic, run the wrapper again to manage and prune snapshots, such as `restic-myrepo forget --keep-last 10 --prune`.

## Do I Need Atomic Backups?

Maybe.
Depending on how long `kopia` or `restic` takes to run and what else the machine is doing, the filesystem might change a lot between the first and the last file that the application backs up.
Related files like sidecars and multipart archives could be out of sync and appear corrupt when you restore from the backup.

That said, this drift might not be a problem.
Unless a file changes very frequently, it's likely that it gets backed up correctly the next time `kopia` or `restic` runs.
In that case, backup integrity is OK, but the backup might provide a recovery point objective one snapshot further back than it appears.

For general, non-database use, `kopiatomic` and `restomic` can make your `kopia` and `restic` backups more consistent at the cost of a little complexity.
Note that these scripts are not a robust solution for data where atomic backups are critical.

One limitation to be aware of: `kopiatomic` and `restomic` don't make precisely atomic snapshots across datasets.
Files within each dataset are captured atomically, but datasets might be captured at slightly different times, because the `zfs snapshot` command can take a few milliseconds to run.

## Common Issues

### My `restic` backup registers some directories as modified in every snapshot.

This issue happens because `restomic` recreates and then removes the directory tree above the specified datasets each time.
For example, if you run `restomic tank/usr/home/ben`, you might see this:

```console
$ restic diff --metadata aaaaaaaa bbbbbbbb
repository xxxxxxxx opened (version 2, compression level auto)
comparing snapshot aaaaaaaa to bbbbbbbb:

U    /restic/
U    /restic/tank
U    /restic/tank/usr
U    /restic/tank/usr/home

Files:           0 new,     0 removed,     0 changed
Dirs:            0 new,     0 removed
Others:          0 new,     0 removed
Data Blobs:      0 new,     0 removed
Tree Blobs:      4 new,     4 removed
  Added:   1.288 KiB
  Removed: 1.288 KiB
```

These directories are created to mount datasets into so that the backup mirrors the mounted filesystem.
`restic` sees the that the modify time is today and updates the modify time in the backup accordingly.
This metadata change adds a trivial amount to the backup, but it might add up if you back up very frequently.

You can avoid the change by setting `LEAVE_DIRS_IN_PLACE=1` at the top of the script.
With this flag set, `restomic` doesn't delete the working directories when it's done.
Note that you then need to remove them manually before backing up datasets at a higher level (such as `tank/usr/home` instead of `tank/usr/home/ben/`).

## Acknowledgements

I built on some of the work in this blog post by Arsen Arsenović: [Unattended backups with ZFS, restic, Backblaze B2 and systemd](https://www.aarsen.me/posts/2022-02-15-sweet-unattended-backups.html).
He uses a patched version of `restic` to solve the issue where it incorrectly registers all directories as changed when running from a zfs snapshot.
A similar patch for this issue is now available in `restic` with the `device-id-for-hardlinks` feature flag.
