# Btrfs Home Snapshot Tools

Two small Python scripts for creating and listing **writable Btrfs snapshots** of user home directories:

- `btrfs-snapshot-manager` creates and lists snapshots.
- `btrfs-snapshot-daemon` checks for eligible accounts and requests daily snapshots.

Snapshots are stored under `/home/.snapshots/<username>/`.

## Access control

The snapshot directory for each account is owned by root. A POSIX ACL grants that account read and traverse access to its directory. Other users are not granted access; root can access all snapshots.

The ACL is set with `setfacl`, so the system must have the ACL utilities installed. If the account's home directory is a Btrfs subvolume, the snapshot itself is writable, but its path is protected by the per-user snapshot directory.

## Requirements

- Linux with Btrfs.
- Python 3.
- Btrfs command-line tools, usually provided by the `btrfs-progs` package.
- ACL tools providing `setfacl`.
- `/home/<username>` must be a Btrfs subvolume.
- `/home/.snapshots` must be on the same Btrfs filesystem as the home subvolumes.
- Root privileges to create snapshots or run the daemon.

## Installation

Install both scripts in the same directory:

```text
/usr/local/sbin/btrfs-snapshot-manager
/usr/local/sbin/btrfs-snapshot-daemon
```

Make them executable:

```bash
sudo chmod 755 \
  /usr/local/sbin/btrfs-snapshot-manager \
  /usr/local/sbin/btrfs-snapshot-daemon
```

The scripts use `/home/<username>` as the source path. They do not use the home-directory path recorded in the account database.

## Usage

### Create a manual snapshot

Create a snapshot for the current account:

```bash
sudo btrfs-snapshot-manager create
```

Create one for a named account:

```bash
sudo btrfs-snapshot-manager create alice
```

Manual snapshots use a timestamp in their name, for example:

```text
/home/.snapshots/alice/manual-2026-10-29_12-30-00-123456
```

### List snapshots

A non-root user can list only their own snapshots:

```bash
btrfs-snapshot-manager list
```

Root can list one account's snapshots:

```bash
sudo btrfs-snapshot-manager list alice
```

Root can list snapshots for all accounts:

```bash
sudo btrfs-snapshot-manager list
```

### Help and version

```bash
btrfs-snapshot-manager --help
btrfs-snapshot-manager --version

btrfs-snapshot-daemon --help
btrfs-snapshot-daemon --version
```

Both scripts also support `-h` and `-v`.

## Run the daily daemon with systemd

Create `/etc/systemd/system/btrfs-snapshot-daemon.service`:

```ini
[Unit]
Description=Daily Btrfs snapshots of home directories
After=local-fs.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/btrfs-snapshot-daemon --interval 5
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now btrfs-snapshot-daemon.service
```

Check the service and its logs:

```bash
sudo systemctl status btrfs-snapshot-daemon.service
sudo journalctl -u btrfs-snapshot-daemon.service
```

Stop and disable it:

```bash
sudo systemctl disable --now btrfs-snapshot-daemon.service
```

The daemon scans accounts every five seconds. It skips accounts with a UID below `1000`, skips the `nobody` account, and processes accounts with a matching directory under `/home`. It creates no more than one daily snapshot per account per calendar day.

## Snapshot behavior and location

Snapshots use this layout:

```text
/home/.snapshots/<username>/daily-YYYY-MM-DD
/home/.snapshots/<username>/manual-YYYY-MM-DD_HH-MM-SS-microseconds
```

Snapshots are writable: the manager does not pass Btrfs's read-only option. Changes made inside a snapshot are separate from changes to the original subvolume.

The snapshots are accessible through the existing `/home` mount. The scripts do not separately mount snapshots.

## Important notes

- These scripts do **not** modify or reload `/etc/fstab`. Snapshots stored under `/home` do not require separate `fstab` entries.
- Snapshots are on the same filesystem as the original data. They are not a substitute for backups on another disk or system.
- The scripts assume each home subvolume is mounted at `/home/<username>`. If your Btrfs layout mounts home subvolumes from another path, update `HOME_ROOT` in both scripts.
- The daemon does not delete old snapshots. Add a retention policy separately if you need automatic cleanup.
