# Docker IPv6 Connection Reset Problem - Solution for Home Servers

## Problem Overview

### Symptoms
When pulling Docker images, you encounter intermittent failures with errors like:
```
failed to copy: read tcp [2a02:...]:port->[2606:4700::...]:443: read: connection reset by peer
```

**Key characteristics:**
- Works sometimes, fails randomly (non-deterministic)
- Error contains IPv6 addresses (indicated by colons in IP)
- Cloudflare CDN address in error: `2606:4700::...`
- Particularly affects Docker Hub images
- More frequent since late 2023

### When Does This Occur?

This issue appears on systems where:
- IPv6 is enabled on the host
- ISP provides IPv6 connectivity (even if unstable)
- Docker is configured to use IPv6 (default behavior)
- Pulling images from Docker Hub (which uses Cloudflare CDN)

## Root Cause Analysis

### Why IPv6 Causes Problems

1. **Docker Hub Infrastructure**
   - Docker Hub uses Cloudflare CDN for image distribution
   - Cloudflare serves content over both IPv4 and IPv6
   - Docker prefers IPv6 when available (RFC 6724 behavior)

2. **Common IPv6 Issues on Home Networks**
   - **Inconsistent Routing**: ISP IPv6 routes are unstable
   - **MTU Problems**: IPv6 path MTU discovery failures
   - **Firewall Issues**: Router/ISP firewalls partially block IPv6
   - **NAT64/DNS64**: Translation layers causing instability
   - **Tunnel Brokers**: 6to4/Teredo tunnels with poor reliability

3. **Cloudflare CDN Specifics**
   - Reports of intermittent TLS handshake failures over IPv6
   - Some users report 50% failure rate on IPv6 connections
   - Same requests work consistently over IPv4

### Is This Cloudflare's Fault?

**Partially, but not entirely:**

- **Cloudflare's Role**: Their IPv6 CDN infrastructure has known intermittent issues
- **ISP's Role**: Many home ISPs have poorly configured IPv6 routing
- **Docker's Role**: Docker doesn't gracefully fall back from IPv6 to IPv4 on failures

**The combination creates a perfect storm for home server users.**

## The Solution

### Quick Fix (Immediate)

Disable IPv6 for Docker while keeping system IPv6 available:

```bash
# 1. Edit Docker daemon configuration
sudo nano /etc/docker/daemon.json

# 2. Add these settings:
{
  "ipv6": false,
  "ip6tables": false
}

# 3. Restart Docker
sudo systemctl restart docker
```

### Complete Fix (Permanent & System-Wide)

This disables IPv6 at the system level, forcing all Docker traffic over IPv4:

```bash
# 1. Stop Docker services
sudo systemctl stop docker
sudo systemctl stop docker.socket

# 2. Disable IPv6 at kernel level
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1

# 3. Make it persistent (survives reboots)
echo "net.ipv6.conf.all.disable_ipv6=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6=1" | sudo tee -a /etc/sysctl.conf

# 4. Configure Docker daemon
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "ipv6": false,
  "ip6tables": false
}
EOF

# 5. Start Docker
sudo systemctl start docker

# 6. Verify the fix
docker pull alpine:latest
```

### Verification

After applying the fix, you should see:

```bash
$ docker pull alpine:latest
latest: Pulling from library/alpine
644afed44dca: Pull complete
Digest: sha256:...
Status: Downloaded newer image for alpine:latest
```

**No IPv6 addresses should appear in any output.**

## Impact on Other Services

### What Still Works?

✅ **Cloudflare Tunnel** (cloudflared)
- Uses IPv4 for tunnel connections by default
- Will continue working normally

✅ **Most web services**
- Modern services support both IPv4 and IPv6
- IPv4 is universal and always works

✅ **Container networking**
- Containers communicate internally via Docker networks
- No impact on inter-container communication

### What Breaks?

❌ **IPv6-only services** (extremely rare)
- Services that ONLY support IPv6 (virtually non-existent in 2025)

❌ **IPv6-specific testing**
- If you need to test IPv6 connectivity, re-enable temporarily

## Alternative Solutions

If you need to keep IPv6 enabled system-wide:

### Option 1: Use Docker Registry Mirrors

Configure Docker to use mirrors that have better IPv6:

```json
{
  "registry-mirrors": ["https://mirror.gcr.io"],
  "ipv6": false
}
```

### Option 2: Prefer IPv4 in Docker Only

Add to `/etc/docker/daemon.json`:

```json
{
  "dns": ["8.8.8.8", "8.8.4.4"],
  "ipv6": false,
  "fixed-cidr-v6": "",
  "experimental": true,
  "ip6tables": false
}
```

### Option 3: Use Private Registry

Set up a local Docker registry to cache images:

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Then pull through your local registry
docker pull localhost:5000/alpine:latest
```

## Troubleshooting

### Fix Not Working?

1. **Verify IPv6 is disabled:**
```bash
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
# Should output: 1
```

2. **Check Docker daemon config:**
```bash
docker info | grep -i ipv6
# Should show: IPv6: false
```

3. **Restart everything:**
```bash
sudo systemctl restart docker
sudo systemctl restart docker.socket
```

4. **Check for other network issues:**
```bash
# Test basic connectivity
ping -c 3 8.8.8.8

# Test DNS
nslookg docker.io

# Test Docker Hub connectivity
curl -I https://registry-1.docker.io/v2/
```

### Still Having Issues?

- Check firewall rules: `sudo iptables -L -n -v`
- Check DNS resolution: `systemd-resolve --status`
- Review Docker logs: `journalctl -u docker -n 50`

## Background Information

### Why This Happens Now

This problem became prevalent in 2023-2024 because:

1. **Docker Hub migrated to Cloudflare CDN** for better global distribution
2. **More ISPs rolled out IPv6** without proper configuration
3. **Docker's IPv6 preference** combined with unstable ISP IPv6 routing
4. **Cloudflare's IPv6 CDN** has intermittent issues in some regions

### Long-term Expectations

- **IPv6 will improve** as ISPs mature their implementations
- **Cloudflare may fix** their CDN IPv6 reliability issues
- **Docker might add** better IPv4 fallback mechanisms

**Until then, disabling IPv6 for Docker is the most reliable solution for home servers.**

## References

- [Docker Forums: Connection Reset by Peer](https://forums.docker.com/t/connection-reset-by-peer/144317)
- [GitHub Issue: cloudflared IPv6 problems](https://github.com/cloudflare/cloudflared/issues/811)
- [Cloudflare Community: IPv6 TLS Errors](https://community.cloudflare.com/t/intermittent-tls-errors-when-accessing-websites-behind-cloudflare-via-ipv6/650353)


---

**Last Updated:** January 2026  
**Tested On:** Ubuntu 24.04, Debian 12, Rocky Linux 9  
**Docker Versions:** 20.10+, 24.0+, 25.0+