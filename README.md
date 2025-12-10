# ansible-role-plex

## Overview

This role installs and configures **Plex Media Server** on a RHEL-based homelab system.
It creates a dedicated service user, mounts NAS-backed media and backup directories, configures the official Plex repository, installs Plex, and deploys systemd overrides to ensure Plex starts under the correct user and only after network storage is available.

The result is a stable, predictable Plex deployment that cleanly integrates with NAS storage and supports long-term homelab uptime.

---

## Required Variables

This role **requires** definition of at least one Plex NFS mount.
These mounts represent the media libraries Plex will index and any backup or metadata storage the system relies on.

| Variable          | Description                                               |
| ----------------- | --------------------------------------------------------- |
| `plex_nfs_mounts` | A list of NFS mount definitions (`path` + `src`) for Plex |

Each list item **must** contain:

* `path` — the local mount point on the Plex server
* `src` — the NAS NFS export that should be mounted

These mounts will be configured persistently (via `/etc/fstab`) and must exist for Plex to function correctly.

You may define these variables:

* in the calling playbook
* in `group_vars` / `host_vars`
* or passed inline when including the role

**Example:**

```yaml
- name: Deploy Plex Media Server
  hosts: plex
  vars:
    plex_nfs_mounts:
      - path: "/opt/media"
        src: "truenas.int.snyderfamily.co:/mnt/splish-splash/media"
      - path: "/opt/backup"
        src: "backup.int.snyderfamily.co:/mnt/puddle-party/plex-backup"

  roles:
    - plex
```

The above structure mirrors how mounts are defined in `defaults/main.yml`, allowing you to override or extend them depending on your environment.

---

## Task-by-Task Breakdown

### Creating the Media Group

A system group named **media** (GID 3000) is created.
This group owns the Plex-related directories, ensuring consistent permissions across local and NFS-mounted paths.

### Creating the Media User

A dedicated **media** user (UID 3000) is created and added to the media group.
This user replaces the default Plex service account so that NAS permissions remain consistent.

### Creating the Media User’s Home Directory

The role creates `/home/media` with correct ownership and permissions.
While Plex does not use this directory directly, maintaining proper home directory structure prevents OS inconsistencies.

### Mounting NFS Media Share(s)

Using the list provided in **plex_nfs_mounts**, each mount is:

* created as a directory
* mounted via NFS
* written to `/etc/fstab` for persistence

This allows Plex to read large media libraries without requiring local storage.

### Mounting NFS Backup Share(s)

Backup or metadata storage mounts are handled the same way, ensuring Plex has a safe location for exports and snapshots.

### Refreshing DNF Cache After Repository Changes

DNF metadata is expired to avoid installation failures when repositories change.

### Adding the Official Plex Repository

The official Plex RPM repository is added, enabling continuous updates and verified package installation.

### Forcing Another DNF Cache Refresh

A second refresh ensures the repository is detected immediately before installation.

### Installing Plex Media Server

The `plexmediaserver` package is installed, providing the primary service binary, supporting libraries, and web UI.

### Stopping and Disabling the Default Plex Service

To reconfigure systemd startup behavior, the default service is temporarily stopped and disabled.

### Creating the Systemd Override Directory

The folder `/etc/systemd/system/plexmediaserver.service.d/` is created to hold the override configuration.

### Deploying the Plex Systemd Override

The override ensures Plex:

* runs as the **media** user
* starts only after NFS media paths are ready
* depends on `/opt/media` to prevent early startup

This eliminates common issues like empty libraries or failed scans at boot.

### Reloading Systemd to Apply Service Changes

Systemd is reloaded so the override takes effect.

---

## Summary

This role fully configures a production-grade Plex Media Server backed by NFS storage.
It ensures correct user identity, reliable mounting of media and backup directories, correct repository configuration, and robust service startup sequencing.

By defining the **required variable** `plex_nfs_mounts`, you guarantee Plex launches in a consistent, storage-aware environment suitable for homelab use.
