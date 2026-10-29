# Btrfs Home Snapshots

A small Python script for creating and listing **writable Btrfs snapshots** of user home directories under `/home`.

The script can create snapshots manually or run as a systemd service. The service checks for eligible user accounts every five seconds and creates at most one daily snapshot per user.

## Features

- Creates writable Btrfs snapshots of `/home/<username>`.
- Stores snapshots under `/home/.snapshots/<username>/`.
- Creates manual snapshots on request.
- Creates one daily snapshot per eligible user.
- Detects newly created eligible accounts by checking the system account list.
- Lets non-root users list only their own snapshots.
- Lets root create and list snapshots for any user.
- Provides `--help` / `-h` and `--version` / `-v` options.

## Requirements

- Linux with Btrfs.
- Python 3.
- The Btrfs command-line tools, usually provided by the `btrfs-progs` package.
- `/home/<username>` must be a Btrfs subvolume.
- `/home/.snapshots` must be on the same Btrfs filesystem as the home subvolumes.
- Root privileges to create snapshots or run the service.

## Installation

Save the script as:

```text
/usr/local/sbin/btrfs-home-snapshots
```

Make it executable:

```bash
sudo chmod 755 /usr/local/sbin/btrfs-home-snapshots
```

The script uses `/home/<username>` as the source path. It does not use the home directory path recorded in the system account database.

## Usage

### Create a manual snapshot

Create a snapshot for the current account:

```bash
btrfs-home-snapshots create
```

Create a snapshot for a named account:

```bash
sudo btrfs-home-snapshots create alice
```

Manual snapshots are named with a timestamp, for example:

```text
/home/.snapshots/alice/manual-2026-10-29_12-30-00
```

### List snapshots

List the current user's snapshots:

```bash
btrfs-home-snapshots list
```

A non-root user can list only their own snapshots. Root can list one user's snapshots:

```bash
sudo btrfs-home-snapshots list alice
```

Root can list snapshots for all users:

```bash
sudo btrfs-home-snapshots list
```

### Help and version

```bash
btrfs-home-snapshots --help
btrfs-home-snapshots --version
```

The `-h` and `-v` short options are also available.

## Run the daily snapshot service

Create `/etc/systemd/system/btrfs-home-snapshots.service`:

```ini
[Unit]
Description=Daily Btrfs snapshots of home directories
After=local-fs.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/btrfs-home-snapshots daemon --interval 5
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reload systemd's service definitions and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now btrfs-home-snapshots.service
```

Check the service status and logs:

```bash
sudo systemctl status btrfs-home-snapshots.service
sudo journalctl -u btrfs-home-snapshots.service
```

The daemon checks accounts every five seconds. It skips accounts with a UID below `1000`, skips the `nobody` account, and only processes accounts that have a matching directory under `/home`. It creates a snapshot when the daemon first runs, then creates no more than one daily snapshot per user for each calendar day.

To stop and disable the service:

```bash
sudo systemctl disable --now btrfs-home-snapshots.service
```

## Snapshot location and behavior

Snapshots are stored in this layout:

```text
/home/.snapshots/<username>/daily-YYYY-MM-DD
/home/.snapshots/<username>/manual-YYYY-MM-DD_HH-MM-SS
```

Snapshots are **writable**: the Btrfs snapshot command does not use the read-only option. Changes made inside a snapshot do not automatically change the original home subvolume, and changes to the original do not automatically update the snapshot.

Snapshots are accessible at the paths above through the existing `/home` mount. The script does not separately mount snapshots.

## Important notes

- This script does **not** modify or reload `/etc/fstab`. Snapshots under the existing `/home` mount do not require separate `fstab` entries.
- Snapshots are stored on the same filesystem as `/home`; they are not a substitute for backups on another disk or system.
- The script assumes home subvolumes are mounted as `/home/<username>`. If your Btrfs layout uses a subvolume such as `@` mounted elsewhere, change the `source` path in the script to match your actual mounted layout.
- The script does not delete old snapshots. Add a separate retention policy only after deciding how long snapshots should be kept.
