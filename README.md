# Docker-Samba-Server-For-XP-Clients
Samba Server For Windows XP Clients [Docker]
<br><br><b>Dockerfile</b></br>
<pre>
# 1. Build
# docker build -t debian12:xp-samba .

# 2. Run (most common way – map host folder into /mnt)
# Note:  Docker bypasses firewall.  Make sure real path has read/write permission.
# docker run -d --name samba-xp -p 445:445 -v /path/on/your/host/xp-share:/mnt --restart unless-stopped debian12:xp-samba
# OR
# docker run -d --name samba-xp -p 445:445 -v /path/on/your/host/xp-share:/mnt --rm debian12:xp-samba

# =============================================================================
# Samba server compatible with Windows XP (requires SMB1/NT1)
# Debian 12 (bookworm) base image
# =============================================================================

FROM debian:bookworm-slim

# ─────────────────────────────────────────────────────────────────────────────
# Install Samba + basic utilities
# ─────────────────────────────────────────────────────────────────────────────
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
        samba \
        procps \
        iproute2 \
        && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/* && \
    mkdir -p /mnt

# ─────────────────────────────────────────────────────────────────────────────
# Create minimal /etc/samba/smb.conf with XP compatibility
# (you can also bind-mount your own file at runtime)
# ─────────────────────────────────────────────────────────────────────────────
COPY <<'EOF' /etc/samba/smb.conf
[global]
    workgroup = WORKGROUP
    # Enables SMB1 for XP
    server min protocol = NT1
    server max protocol = SMB3
    map to guest = bad user
    usershare allow guests = yes
    # Helps a bit with XP-era clients on fast links, but not mandatory
    #min receivefile size = 16384

    # Common settings that help with ancient clients
    dns proxy = no
    log file = /var/log/samba/log.%m
    max log size = 50
    logging = file
    panic action = /usr/share/samba/panic-action %d

    # Unless you have very specific, well-measured workload that actually benefits from 
    # fixed buffers (very rare nowadays), remove the line completely.
    # Performance / compatibility tweaks
    #socket options = TCP_NODELAY IPTOS_LOWDELAY SO_RCVBUF=65536 SO_SNDBUF=65536

[xp-share]
    # Make sure this path is read/write
    path = /mnt
    browseable = yes
    writable = yes
    # Easy, no password (or set valid users = yourusername and use password)
    guest ok = yes
    read only = no
    #force user = testuser
EOF

# ─────────────────────────────────────────────────────────────────────────────
# Create log directory (samba likes to write logs)
# ─────────────────────────────────────────────────────────────────────────────
RUN mkdir -p /var/log/samba && \
    chmod 1777 /var/log/samba

# ─────────────────────────────────────────────────────────────────────────────
# Expose Samba ports
# ─────────────────────────────────────────────────────────────────────────────
# 139  → NetBIOS session service (usually not needed with modern clients)
# 445  → SMB over TCP (the one XP will mostly use)
EXPOSE 139 445

# ─────────────────────────────────────────────────────────────────────────────
# Healthcheck (optional but recommended)
# ─────────────────────────────────────────────────────────────────────────────
HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD smbstatus -b || exit 1

# ─────────────────────────────────────────────────────────────────────────────
# Run smbd in foreground (required for Docker to consider the container running)
# In a standard Docker container, you should not run a process in "daemon" or
# background mode. If you use the -D flag (which tells Samba to daemonize/fork
# into the background), the primary process will start, spawn a child process, 
# and then immediately exit. Since Docker monitors the PID 1 process, as soon
# as that process exits, Docker thinks the container has finished its job and will
# shut it down.
#
# -F	Runs smbd in the foreground. This is the most important change for Docker.
# --no-process-group	Prevents Samba from creating a new process group, making it 
#       easier for Docker to manage and terminate the process.
# -s	Specifies the path to the configuration file (your current usage is correct).
# -S	(Optional) Logs to stdout instead of a file, which is the "Docker way" of handling logs.
# ─────────────────────────────────────────────────────────────────────────────
CMD ["/usr/sbin/smbd", "--foreground", "--no-process-group", "-s", "/etc/samba/smb.conf"] 
</pre>
