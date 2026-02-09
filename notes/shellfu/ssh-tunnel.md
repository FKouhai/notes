---
id: ssh-tunnel
title: SSH Tunneling - Secure Port Forwarding
aliases:
  - ssh-forwarding
  - port-forwarding
  - ssh-tunneling
tags:
  - shell
  - ssh
  - networking
  - security
description: Comprehensive guide to SSH tunneling and port forwarding with practical examples
---

# SSH Tunneling

SSH tunneling (also known as SSH port forwarding) is a method of transporting arbitrary data over an encrypted SSH connection. It allows you to securely access network services running on remote or local machines.

## Types of SSH Tunneling

There are three main types of SSH tunneling:

1. **Local Port Forwarding** (-L) - Forward local port to remote server
2. **Remote Port Forwarding** (-R) - Forward remote port to local machine
3. **Dynamic Port Forwarding** (-D) - Create SOCKS proxy

## Basic Syntax

```bash
# Local port forwarding
ssh -L [local_port:]local_host:remote_port remote_user@remote_host

# Remote port forwarding
ssh -R [remote_port:]remote_host:local_port remote_user@remote_host

# Dynamic port forwarding (SOCKS proxy)
ssh -D local_port remote_user@remote_host
```

## Local Port Forwarding

### Basic Usage

```bash
# Forward local port 8080 to remote host's port 80
# Purpose: Access remote web server via localhost:8080
ssh -L 8080:localhost:80 user@remote.example.com

# Forward local port 3306 to remote database server's port 3306
# Purpose: Access remote MySQL from local machine
ssh -L 3306:localhost:3306 user@db.example.com

# Forward local port 5901 to remote VNC server
# Purpose: Access remote VNC session
ssh -L 5901:localhost:5901 user@remote.example.com
```

### Forward to Another Remote Host

```bash
# Access third host via jump server
# Purpose: Reach db.internal.com through jump.example.com
ssh -L 3306:db.internal.com:3306 user@jump.example.com

# Multiple port forwards
# Purpose: Access multiple services through single SSH connection
ssh -L 8080:web.internal.com:80 -L 3306:db.internal.com:3306 user@jump.example.com
```

### Real-World Examples

```bash
# Access Kubernetes API server securely
# Purpose: Connect to k8s API from localhost
ssh -L 6443:localhost:6443 k8s-admin@k8s-control-plane.example.com

# Access Grafana running on remote server
# Purpose: View Grafana dashboard without exposing it publicly
ssh -L 3000:localhost:3000 user@monitoring.example.com

# Access PostgreSQL in private VPC
# Purpose: Connect to DB from local PostgreSQL client
ssh -L 5432:postgres-private.internal:5432 user@bastion.example.com

# Access Redis cluster
# Purpose: Connect to Redis from local redis-cli
ssh -L 6379:redis.internal:6379 user@bastion.example.com
```

### Persistent Local Tunnels

```bash
# Keep tunnel alive with ServerAliveInterval
# Purpose: Prevent tunnel from timing out due to inactivity
ssh -L 8080:localhost:80 -o ServerAliveInterval=60 user@remote.example.com

# Use autossh for automatic reconnection
# Purpose: Maintain tunnel even after network disruption
autossh -M 0 -L 8080:localhost:80 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" user@remote.example.com

# Background tunnel with nohup
# Purpose: Run tunnel in background, survives terminal close
nohup ssh -L 8080:localhost:80 user@remote.example.com > /dev/null 2>&1 &

# Background tunnel with screen/tmux
# Purpose: Manage tunnel in detachable session
screen -S tunnel
ssh -L 8080:localhost:80 user@remote.example.com
# Press Ctrl+A+D to detach
```

## Remote Port Forwarding

### Basic Usage

```bash
# Forward remote port 8080 to local machine's port 80
# Purpose: Make local web server accessible from remote server
ssh -R 8080:localhost:80 user@remote.example.com

# Forward remote port 3306 to local database
# Purpose: Allow remote server to access local database
ssh -R 3306:localhost:3306 user@remote.example.com
```

### Real-World Examples

```bash
# Expose local development server to internet
# Purpose: Share localhost:3000 with remote server for testing
ssh -R 8080:localhost:3000 user@remote.example.com

# Allow remote server to access local API
# Purpose: Remote server calls your local development API
ssh -R 9000:localhost:9000 user@api.example.com

# Reverse tunnel for behind-NAT access
# Purpose: Access home computer from work server (home behind NAT)
# On home machine:
ssh -R 2222:localhost:22 user@work.example.com
# On work machine:
ssh -p 2222 localhost  # Connects back to home machine

# Webhook development
# Purpose: Receive webhooks on local machine from external service
ssh -R 8443:localhost:8080 user@public-server.example.com
# Point webhook URL to: https://public-server.example.com:8443
```

### GatewayPorts for Public Access

```bash
# Allow remote tunnel to be accessible from any host (not just localhost)
# On remote server's sshd_config: GatewayPorts yes
sudo systemctl restart sshd

# Then connect
ssh -R 0.0.0.0:8080:localhost:80 user@remote.example.com
# Now anyone can connect to remote.example.com:8080
```

## Dynamic Port Forwarding (SOCKS Proxy)

### Basic Usage

```bash
# Create SOCKS proxy on localhost port 1080
# Purpose: Route traffic through remote server dynamically
ssh -D 1080 user@remote.example.com

# Create proxy with specific bind address
ssh -D 127.0.0.1:1080 user@remote.example.com
```

### Real-World Examples

```bash
# Browse internet through remote server
# Purpose: Appear as if browsing from remote server's IP
ssh -D 1080 user@remote.example.com
# Configure browser to use SOCKS5 proxy at localhost:1080

# Access internal network services
# Purpose: Browse internal web servers in private network
ssh -D 1080 user@bastion.example.com
# Configure browser proxy, then visit http://internal.service/

# SSH through SOCKS proxy
# Purpose: SSH to internal hosts via jump server
ssh -D 1080 user@bastion.example.com
# In another terminal:
ssh -o ProxyCommand="nc -X 5 -x 127.0.0.1:1080 %h %p" user@internal.example.com

# Git through SOCKS proxy
# Purpose: Clone git repositories in private network
ssh -D 1080 user@bastion.example.com
git config --global http.proxy socks5://127.0.0.1:1080
git clone http://git.internal.example.com/repo.git
```

### Applications with SOCKS Proxy

```bash
# cURL through SOCKS proxy
curl --socks5 127.0.0.1:1080 http://internal.example.com/api

# wget through SOCKS proxy
wget -e "use_proxy=yes" -e "http_proxy=socks5://127.0.0.1:1080" http://internal.example.com/file

# Docker with SOCKS proxy
export ALL_PROXY=socks5://127.0.0.1:1080
docker pull internal.registry.example.com/image:tag
```

## Advanced SSH Tunneling

### Multiple Tunnels in One Connection

```bash
# Multiple local forwards
ssh -L 8080:localhost:80 -L 3306:localhost:3306 -L 6379:localhost:6379 user@remote.example.com

# Mix local and remote forwards
ssh -L 8080:localhost:80 -R 9000:localhost:3000 user@remote.example.com

# Add dynamic port forwarding
ssh -L 8080:localhost:80 -D 1080 user@remote.example.com
```

### Bind Addresses

```bash
# Bind to specific local address (not just localhost)
ssh -L 127.0.0.1:8080:localhost:80 user@remote.example.com  # Only localhost
ssh -L 0.0.0.0:8080:localhost:80 user@remote.example.com     # All interfaces

# Forward to specific remote address
ssh -L 8080:192.168.1.100:80 user@remote.example.com
```

### SSH Config for Tunnels

```bash
# ~/.ssh/config
Host tunnel-web
    HostName remote.example.com
    User user
    LocalForward 8080 localhost:80

Host tunnel-db
    HostName db.example.com
    User dbuser
    LocalForward 3306 localhost:3306

Host tunnel-proxy
    HostName proxy.example.com
    User proxyuser
    DynamicForward 1080
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Usage
ssh tunnel-web     # Establishes web tunnel
ssh tunnel-db      # Establishes DB tunnel
ssh tunnel-proxy   # Establishes SOCKS proxy
```

### Jump Hosts

```bash
# Simple jump host
# Purpose: Access final server through intermediate server
ssh -J user@jump.example.com user@final.example.com

# Jump host with port forwarding
ssh -J user@jump.example.com -L 8080:localhost:80 user@final.example.com

# Multiple jump hosts
ssh -J user@jump1.example.com,user@jump2.example.com user@final.example.com

# In SSH config
Host final
    HostName final.example.com
    User user
    ProxyJump jump1,jump2
```

## Tunnel Management

### Listing Active Tunnels

```bash
# List SSH connections (including tunnels)
ps aux | grep ssh

# List listening ports
netstat -tlnp | grep ssh
# or
ss -tlnp | grep ssh

# Check if tunnel is working
curl -v http://localhost:8080
```

### Terminating Tunnels

```bash
# Kill specific SSH connection
pkill -f "ssh -L 8080"

# Kill all SSH tunnels
pkill -f "ssh.*-L"
pkill -f "ssh.*-R"
pkill -f "ssh.*-D"

# Kill by PID
ps aux | grep ssh
kill <PID>

# Exit SSH session normally
# Press Ctrl+C or type "exit" in SSH session
```

### Monitoring Tunnels

```bash
# Monitor SSH connection
# Keep alive ping to detect dropped connection
ssh -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 8080:localhost:80 user@remote.example.com

# Auto-reconnect with autossh
autossh -M 0 -L 8080:localhost:80 \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    -o "ExitOnForwardFailure yes" \
    user@remote.example.com
```

## Security Best Practices

### SSH Configuration

```bash
# /etc/ssh/sshd_config (on remote server)
GatewayPorts no              # Default: only allow remote port forwarding to localhost
AllowTcpForwarding yes       # Enable port forwarding
PermitTunnel yes             # Enable tunnel device forwarding
MaxSessions 10               # Limit concurrent sessions
PermitRootLogin no           # Disable root login
PasswordAuthentication no    # Disable password auth, use keys only

# /etc/ssh/ssh_config (local client)
ServerAliveInterval 60       # Keep connections alive
ServerAliveCountMax 3
TCPKeepAlive yes
Compression yes              # Compress tunnel traffic
```

### SSH Keys Management

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "user@example.com"

# Copy key to remote server
ssh-copy-id user@remote.example.com

# Use specific key for tunnel
ssh -i ~/.ssh/tunnel_key -L 8080:localhost:80 user@remote.example.com

# In SSH config
Host tunnel-web
    HostName remote.example.com
    User user
    IdentityFile ~/.ssh/tunnel_key
    LocalForward 8080 localhost:80
```

### Firewall Considerations

```bash
# Firewall on local machine (ufw example)
# Allow SSH to remote server
sudo ufw allow from remote.example.com to any port 22

# Firewall on remote server
# Only allow SSH from trusted IPs
sudo ufw allow from trusted.ip.address to any port 22

# For public remote port forwarding:
# Only allow specific port for tunnel traffic
sudo ufw allow from any to any port 8080 proto tcp
```

## Troubleshooting

### Common Issues

```bash
# Permission denied
# Check: SSH key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa

# Port already in use
# Find and kill process using port
lsof -ti:8080 | xargs kill -9

# Connection refused
# Check: SSH service running
sudo systemctl status ssh

# Check: Firewall blocking
sudo ufw status
sudo iptables -L -n

# Tunnel works once then dies
# Add keepalive options
ssh -o ServerAliveInterval=30 -L 8080:localhost:80 user@remote.example.com
```

### Debug Mode

```bash
# Verbose SSH output
ssh -v -L 8080:localhost:80 user@remote.example.com

# Very verbose
ssh -vvv -L 8080:localhost:80 user@remote.example.com

# Test connection without tunnel
ssh user@remote.example.com

# Test port connectivity
nc -zv localhost 8080
telnet localhost 8080
```

## Use Cases

### Development

```bash
# Access dev database from local IDE
ssh -L 3306:dev-db.internal:3306 dev@bastion.example.com

# Share local dev server with team
ssh -R 3000:localhost:3000 user@shared.example.com

# Access multiple microservices
ssh -L 8080:service1:8080 -L 8081:service2:8080 -L 8082:service3:8080 user@bastion.example.com
```

### Production Access

```bash
# Access production Grafana
ssh -L 3000:localhost:3000 admin@monitoring.example.com

# Access Kubernetes dashboard
ssh -L 8443:localhost:8443 k8s-admin@k8s.example.com

# Access Prometheus
ssh -L 9090:localhost:9090 admin@monitoring.example.com
```

### Database Access

```bash
# MySQL
ssh -L 3306:localhost:3306 dbuser@db.example.com
mysql -h 127.0.0.1 -u appuser -p

# PostgreSQL
ssh -L 5432:localhost:5432 dbuser@db.example.com
psql -h 127.0.0.1 -U appuser -d appdb

# MongoDB
ssh -L 27017:localhost:27017 mongouser@mongo.example.com
mongo --host 127.0.0.1

# Redis
ssh -L 6379:localhost:6379 redisuser@redis.example.com
redis-cli -h 127.0.0.1
```

### CI/CD Pipelines

```bash
# Access private registry from CI
ssh -R 5000:localhost:5000 ci-runner@deploy.example.com
# CI runner now uses localhost:5000 for internal registry

# Access deployment server from CI
ssh -L 2222:deploy.internal:22 ci-runner@bastion.example.com
ssh -p 2222 localhost deploy /path/to/script.sh
```

## Quick Reference

```bash
# Local forwarding
ssh -L local_port:remote_host:remote_port user@server

# Remote forwarding
ssh -R remote_port:local_host:local_port user@server

# Dynamic forwarding (SOCKS proxy)
ssh -D local_port user@server

# Keep alive
ssh -o ServerAliveInterval=60 -L port:host:port user@server

# Background tunnel
ssh -fN -L port:host:port user@server

# With compression
ssh -C -L port:host:port user@server

# Jump host
ssh -J jumpuser@jumphost user@desthost
```
