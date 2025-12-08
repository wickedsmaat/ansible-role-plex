# ansible-role-plex
Overview

This role installs and configures Plex Media Server on a RHEL-based homelab system.
It creates a dedicated service user, mounts media directories from a NAS, configures the official Plex repository, installs Plex, and adds service overrides to ensure Plex starts under the correct user and only after media storage is available.

The end result is a stable, predictable, NAS-backed Plex deployment suitable for long-term homelab usage.

## Task-by-Task Breakdown
Creating the Media Group

This task creates a system group named media (GID 3000).
The group ownership is used for Plex and for all mounted media directories.
This ensures consistent permissions across local and network-mounted storage.

Creating the Media User

A media user (UID 3000) is created and assigned to the media group.
This user will run the Plex service instead of the default plex user.
Running Plex as a custom media user ensures that permissions align naturally with NAS-mounted storage.

Creating the Media User’s Home Directory

This task explicitly creates /home/media with correct owner and permissions.
Some future tasks rely on this path existing, especially for defaults and user environment consistency.
Even if Plex doesn’t use the home directory directly, this maintains proper OS hygiene.

Mounting NFS Media Share

This task mounts an NFS export (e.g., from a TrueNAS server) to /opt/media.
It also writes a persistent entry to /etc/fstab.
By mounting through NFS, the Plex host can index large media libraries without storing them locally.

Mounting NFS Backup Share

Similarly, this mounts a backup location used for Plex metadata, exports, or snapshots.
It is stored separately to avoid filling the main filesystem.
This keeps Plex’s metadata safe and future-proofs the server.

Refreshing DNF Cache After Repository Changes

Whenever new repositories are added, stale metadata can cause installation failures.
This task forces DNF to expire cached data so the system fetches the newest metadata.
It ensures the Plex repository will be recognized immediately.

Adding the Official Plex Repository

This task adds Plex’s official RPM repository to the system’s repo list.
Plex ships its own repository to ensure users always receive the latest stable server builds.
Using the official repo also ensures proper GPG verification and update management.

Forcing Another DNF Cache Refresh

Because a new repo has been added, the cache is refreshed again.
This guarantees that the plexmediaserver package can be located during installation.
It reduces installation errors caused by stale repository metadata.

Installing Plex Media Server

This task installs the plexmediaserver package from the Plex repo.
Installation includes the service, binaries, web interface, and supporting libraries.
After installation, the service exists but does not yet run under the media user.

Stopping and Disabling the Default Plex Service

By default, Plex is enabled to start at boot and runs as the plex user.
This task disables the service temporarily so we can override its configuration.
Stopping it ensures no lingering processes interfere with the new settings.

Creating the Systemd Override Directory for Plex

This task creates /etc/systemd/system/plexmediaserver.service.d/.
Systemd uses this folder for custom configuration overrides.
Without it, we could not modify how Plex runs or what user it runs as.

Deploying the Plex Systemd Override File

This override file instructs systemd to:

Run Plex as the media user

Ensure Plex starts only after media mounts are online

Enforce dependency on /opt/media

This avoids Plex starting too early, which can result in empty libraries or scanning failures.

Reloading Systemd to Apply Service Changes

After placing the override file, systemd must reload its configuration.
This ensures that the next time Plex starts, it uses the updated user identity and dependency rules.
Without this reload, systemd would ignore the new override.

## Summary

This role fully configures a production-ready Plex Media Server for homelab use.
It handles storage mounts, user identity, repository setup, and service overrides to ensure a stable, predictable Plex deployment backed by network storage.