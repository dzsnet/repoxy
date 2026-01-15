# Repoxy - Repository Caching Proxy

A high-performance NGINX-based caching proxy for package repositories (APT, RPM, etc.). Reduces bandwidth usage and speeds up package installations by caching repository metadata and packages locally.

The main purpose was to provide a dynamic repo mirror for Grafana Alloy without having the whole Grafana repo mirrored, but sky is the limit. (huarpvar - the only line here what is not AI generated ;))

## Features

- 🚀 **High-performance caching** - Up to 365-day cache retention for packages
- 🔒 **Whitelist-based security** - Only configured repositories are accessible
- 📦 **Multi-format support** - APT (Debian/Ubuntu), RPM (RedHat/CentOS), and more
- 💾 **5GB default cache** - Configurable storage limit
- 📊 **Monitoring** - Built-in health check and detailed logging
- 🔄 **Intelligent cache** - Ignores upstream cache-control headers to force local caching
- ⚡ **Differentiated caching** - Metadata cached for 5 minutes, packages for 365 days

## Quick Start

### 1. Start the Proxy

```bash
docker-compose up -d
```

### 2. Check Status

```bash
curl http://localhost/proxy-status
# Should return: Repo Cache Proxy - OK
```

### 3. Test Caching

```bash
# First request (MISS)
curl -I http://localhost/apt.grafana.com/gpg-full.key

# Second request (HIT)
curl -I http://localhost/apt.grafana.com/gpg-full.key

# Check cache status in response headers
# X-Cache-Status: HIT/MISS
```

## Architecture

```
Client → Repoxy (Port 80) → Upstream Repository
                ↓
         Cache (5GB max as default)
```

## Currently Whitelisted Repositories

- **Grafana APT**: `apt.grafana.com`
- **Grafana RPM**: `rpm.grafana.com`

## Client Configuration

### APT (Debian/Ubuntu)

Create `/etc/apt/sources.list.d/grafana.list`:

```bash
deb [arch=amd64] http://your-proxy-server/apt.grafana.com/ stable main
```

Then update:

```bash
sudo apt update
sudo apt install grafana
```

### RPM (RHEL/CentOS/Fedora)

Create `/etc/yum.repos.d/grafana.repo`:

```ini
[grafana]
name=Grafana Repository
baseurl=http://your-proxy-server/rpm.grafana.com/
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
```

Then install:

```bash
sudo yum install grafana
# or
sudo dnf install grafana
```

### Testing Client Configuration

```bash
# Test repository access
curl -I http://your-proxy-server/apt.grafana.com/dists/stable/InRelease
curl -I http://your-proxy-server/rpm.grafana.com/repodata/repomd.xml

# Both should return HTTP 200 OK
```

## Adding New Repositories

To whitelist a new repository, add a location block to `nginx.conf`:

### Example: Docker APT Repository

```nginx
# Docker APT Repository
location ~ ^/download\.docker\.com/linux/ubuntu/(.*)$ {
    set $upstream_domain "download.docker.com";
    set $upstream_path "/linux/ubuntu/$1";
    
    proxy_pass https://$upstream_domain$upstream_path$is_args$args;
    proxy_cache_key "$upstream_domain$upstream_path$is_args$args";
    proxy_set_header Host $upstream_domain;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    proxy_ssl_server_name on;
    proxy_ssl_protocols TLSv1.2 TLSv1.3;
}
```

**Steps to add a new repository:**

1. Edit `nginx.conf` and add a new location block (see example above)
2. Update the blocked request message to include the new domain
3. Restart the container: `docker-compose restart`
4. Test the new repository endpoint

### Location Block Components

For each repository, you need **two location blocks**:

**1. Metadata location** (5 minute cache):
```nginx
location ~ ^/domain\.com/(.*)$ {
    proxy_cache_valid 200 5m;  # Short cache for metadata
    # ... other settings
}
```

**2. Package location** (365 day cache):
```nginx
location ~ ^/domain\.com/.*\.(deb|rpm|tar\.gz)$ {
    proxy_cache_valid 200 365d;  # Long cache for packages
    # ... other settings
}
```

**Key components:**
- **`location ~ ^/domain\.com/(.*)$`** - Regex pattern matching the repository path
- **`set $upstream_domain`** - The actual upstream server
- **`set $upstream_path`** - The path to proxy to
- **`proxy_cache_key`** - Unique cache key for each resource
- **`proxy_cache_valid`** - Cache duration (5m for metadata, 365d for packages)
- **`proxy_ssl_server_name on`** - Required for SNI support

## Configuration

### Cache Settings

Default cache configuration in `nginx.conf`:

```nginx
proxy_cache_path /var/cache/nginx/repos 
                 levels=1:2 
                 keys_zone=repo_cache:200m 
                 max_size=5g 
                 inactive=30d 
                 use_temp_path=off;
```

- **max_size**: 5GB maximum cache size
- **inactive**: Remove cache entries not accessed for 30 days
- **keys_zone**: 100MB memory for cache keys

### Cache Duration

The proxy uses **differentiated caching** to balance freshness with bandwidth savings:

**Metadata Files** (5 minute cache):
- `InRelease`, `Release`, `Packages`, `Packages.gz/bz2/xz`
- `repomd.xml`, `primary.xml`, `filelists.xml`, `other.xml`
- Repository index files that change frequently

**Package Files** (365 day cache):
- `.deb`, `.rpm`, `.tar.gz`, `.tar.xz`, `.tar.bz2`
- Actual packages that never change once published

This ensures:
- ✅ Clients see new packages within 5 minutes of release
- ✅ Large package downloads are cached for a year (huge bandwidth savings)
- ✅ No stale package installations

```nginx
# Metadata locations - 5 minute cache
proxy_cache_valid 200 5m;

# Package file locations - 365 day cache  
proxy_cache_valid 200 365d;

# All files - 404 errors cached for 1 hour
proxy_cache_valid 404 1h;
```

### DNS Resolvers

Configure in `nginx.conf`:


```nginx
# nginx.conf
# add your preferred dns servers
resolver 8.8.4.4 8.8.8.8 valid=300s;
resolver_timeout 10s;
```

## Monitoring & Logging

### Log Files

- **access.log** - All successful requests with cache status
- **error.log** - Errors and warnings
- **blocked.log** - Rejected requests (non-whitelisted domains)

### View Logs

```bash
# Real-time access log
docker exec repoxy tail -f /var/log/nginx/access.log

# Real-time error log
docker exec repoxy tail -f /var/log/nginx/error.log

# Blocked requests
docker exec repoxy tail -f /var/log/nginx/blocked.log

# Or from host
tail -f logs/access.log
```

### Cache Statistics

```bash
# Number of cached files
docker exec repoxy find /var/cache/nginx/repos -type f | wc -l

# Cache disk usage
docker exec repoxy du -sh /var/cache/nginx/repos
```

### Health Check

```bash
curl http://localhost/proxy-status
# Returns: Repo Cache Proxy - OK
```

## Troubleshooting

### Issue: 502 Bad Gateway Errors

### Issue: Stale Metadata (Not Seeing New Packages)

**Symptom**: New packages released upstream don't appear in `apt update`

**Solution**: Use differentiated caching with separate location blocks:
- Metadata files (InRelease, Packages, repomd.xml): **5 minute cache**
- Package files (.deb, .rpm): **365 day cache**

This ensures clients check for new packages every 5 minutes while still caching actual package downloads for a year.

**Symptom**: Requests return 502, logs show "no resolver defined"

**Solution**: Ensure `resolver` directive is configured in `nginx.conf`:

```nginx
resolver 8.8.4.4 8.8.8.8 valid=300s;
```

### Issue: Cache Not Working (Always MISS)

**Symptom**: All requests show `cache: MISS` in access logs

**Solution**: Add `proxy_ignore_headers` to override upstream cache-control:

```nginx
proxy_ignore_headers Cache-Control Expires Set-Cookie;
```

Many upstream repositories send `Cache-Control: private` which prevents caching.

### Issue: Permission Denied on Cache Directory

**Symptom**: Errors writing to `/var/cache/nginx`

**Solution**: Ensure cache directories have correct ownership:

```bash
docker exec repoxy chown -R nginx:nginx /var/cache/nginx
```

### Issue: Blocked Requests

**Symptom**: Getting 403 Forbidden

**Solution**: The domain is not whitelisted. Add a location block for the repository or check the URL format matches the regex pattern.

## Directory Structure

```
repoxy/
├── docker-compose.yml    # Docker configuration
├── nginx.conf           # NGINX proxy configuration
├── README.md           # This file
├── testing             # Test commands
├── cache/              # Cache storage (5GB max default)
│   └── repos/         # Cached packages and metadata
└── logs/              # Log files
    ├── access.log     # Request logs
    ├── error.log      # Error logs
    └── blocked.log    # Rejected requests
```

## Security Considerations

- ✅ Only whitelisted repositories are accessible
- ✅ All non-whitelisted requests return 403 Forbidden
- ✅ TLS/SSL enabled for upstream connections (TLSv1.2+)
- ✅ Requests are logged for audit purposes
- ⚠️  GPG key verification happens on the client side
- ⚠️  This proxy does NOT inspect or validate package contents

## Performance Tips

1. **Place on fast storage**: Cache directory benefits from SSD storage
2. **Adjust cache size**: Increase `max_size` based on your needs
3. **Monitor cache hit ratio**: Check logs regularly
4. **Pre-warm cache**: Download commonly used packages after setup

```bash
# Example: Pre-warm Grafana packages
curl -s http://localhost/apt.grafana.com/dists/stable/main/binary-amd64/Packages.gz -o /dev/null
```

## Maintenance

### Clear Cache

```bash
# Stop container
docker-compose down

# Clear cache
rm -rf cache/repos/*

# Start container
docker-compose up -d
```

### Restart Service

```bash
docker-compose restart
```

### Update Configuration

```bash
# Edit nginx.conf
nano nginx.conf

# Validate configuration
docker exec repoxy nginx -t

# Reload without downtime
docker exec repoxy nginx -s reload

# Or restart container
docker-compose restart
```

## License

This project is provided as-is for internal use.

## Support

For issues or questions:
- Check logs: `docker-compose logs -f`
- Verify configuration: `docker exec repoxy nginx -t`
- Review this README for common troubleshooting steps
