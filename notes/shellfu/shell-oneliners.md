---
id: shell-oneliners
title: Shell One-Liners
aliases:
  - shell-recipes
  - cli-tips
tags:
  - shell
  - bash
  - oneliners
  - productivity
description: Collection of useful shell one-liners for everyday tasks
---

# Shell One-Liners

A collection of powerful shell one-liners for common tasks. Each command is explained for clarity.

## File Operations

### File Finding

```bash
# Find files larger than 100MB
find . -type f -size +100M

# Find files modified in last 7 days
find . -type f -mtime -7

# Find empty files
find . -type f -empty

# Find files by extension
find . -name "*.log"

# Find files by multiple extensions
find . \( -name "*.log" -o -name "*.txt" \)

# Find and delete files older than 30 days
find . -type f -mtime +30 -delete

# Find executable files
find . -type f -executable

# Find files with specific permissions
find . -type f -perm 777

# Find files owned by specific user
find . -type f -user username

# Find files by content
grep -r "search_term" /path/to/search

# Find files containing text, showing filenames only
grep -rl "search_term" /path/to/search

# Find files NOT containing text
grep -RL "search_term" /path/to/search
```

### File Manipulation

```bash
# Remove all .tmp files
rm *.tmp

# Remove files in subdirectories
find . -name "*.tmp" -delete

# Rename multiple files (replace spaces with underscores)
rename 's/ /_/g' *

# Batch rename with pattern
rename 's/^prefix_//' prefix_*

# Convert all files to lowercase
for f in *; do mv "$f" "${f,,}"; done

# Convert all files to uppercase
for f in *; do mv "$f" "${f^^}"; done

# Add prefix to all files
for f in *; do mv "$f" "prefix_$f"; done

# Add suffix to all files (before extension)
for f in *; do mv "$f" "${f%.*}_suffix.${f##*.}"; done

# Remove extension from all files
for f in *; do mv "$f" "${f%.*}"; done

# Create backup of all files
for f in *; do cp "$f" "$f.bak"; done

# Touch files with names from list
cat file_list.txt | xargs touch
```

### File Content

```bash
# Show first N lines
head -n 10 file.txt

# Show last N lines
tail -n 10 file.txt

# Show lines N to M
sed -n '10,20p' file.txt

# Remove duplicate lines
sort file.txt | uniq

# Count occurrences of each line
sort file.txt | uniq -c | sort -rn

# Remove empty lines
grep . file.txt > output.txt
# or
sed '/^$/d' file.txt > output.txt

# Remove lines containing pattern
grep -v "pattern" file.txt > output.txt

# Replace text in file
sed 's/old/new/g' file.txt > output.txt

# Show line numbers
cat -n file.txt

# Count total lines
wc -l file.txt

# Count words
wc -w file.txt

# Show random lines
shuf -n 5 file.txt

# Reverse file content
tac file.txt

# Show file in hex
xxd file.txt
```

## Text Processing

### Grep Patterns

```bash
# Case-insensitive search
grep -i "pattern" file.txt

# Show line numbers with matches
grep -n "pattern" file.txt

# Show only matching part (color)
grep --color=always "pattern" file.txt

# Show context (2 lines before and after)
grep -C 2 "pattern" file.txt

# Show context (2 lines before)
grep -B 2 "pattern" file.txt

# Show context (2 lines after)
grep -A 2 "pattern" file.txt

# Count matching lines
grep -c "pattern" file.txt

# Invert match (show non-matching lines)
grep -v "pattern" file.txt

# Match whole word only
grep -w "pattern" file.txt

# Match exact line
grep -x "exact line" file.txt

# Use extended regex
grep -E "pattern1|pattern2" file.txt

# Match multiple files
grep "pattern" file1.txt file2.txt

# Recursive search
grep -r "pattern" /path/to/search

# Search for multiple patterns (OR)
grep -e "pattern1" -e "pattern2" file.txt

# Search for pattern NOT followed by another
grep -v "pattern1" file.txt | grep "pattern2"

# Match IP addresses
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' file.txt

# Match email addresses
grep -E '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt

# Match URLs
grep -E 'https?://[^ ]+' file.txt
```

### Sed Operations

```bash
# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences
sed 's/old/new/g' file.txt

# Replace on specific line numbers
sed '10,20s/old/new/' file.txt

# Delete specific line
sed '5d' file.txt

# Delete range of lines
sed '10,20d' file.txt

# Delete lines matching pattern
sed '/pattern/d' file.txt

# Delete empty lines
sed '/^$/d' file.txt

# Delete last line
sed '$d' file.txt

# Insert line at beginning
sed '1i\First line' file.txt

# Append line at end
sed '$a\Last line' file.txt

# Add line number to each line
sed '=' file.txt | sed 'N;s/\n/ /'

# Print specific lines
sed -n '10,20p' file.txt

# Convert tabs to spaces (4 spaces)
sed 's/\t/    /g' file.txt

# Remove leading whitespace
sed 's/^[ \t]*//' file.txt

# Remove trailing whitespace
sed 's/[ \t]*$//' file.txt

# Remove all whitespace
sed 's/ //g' file.txt

# Replace multiple spaces with single space
sed 's/  */ /g' file.txt

# Convert DOS to Unix line endings
sed 's/\r$//' file.txt

# Convert Unix to DOS line endings
sed 's/$/\r/' file.txt
```

### Awk Operations

```bash
# Print specific column (column 2)
awk '{print $2}' file.txt

# Print multiple columns
awk '{print $1, $3}' file.txt

# Print columns with separator
awk -F: '{print $1, $3}' /etc/passwd

# Sum values in column 3
awk '{sum += $3} END {print sum}' file.txt

# Average of column 3
awk '{sum += $3; count++} END {print sum/count}' file.txt

# Count lines matching pattern
awk '/pattern/ {count++} END {print count}' file.txt

# Print lines longer than 80 characters
awk 'length > 80' file.txt

# Print first line
awk 'NR==1' file.txt

# Print last line
awk 'END {print}' file.txt

# Print line numbers
awk '{print NR, $0}' file.txt

# Filter by condition (column 2 > 100)
awk '$2 > 100' file.txt

# Print rows where column matches pattern
awk '$1 ~ /pattern/' file.txt

# Print total number of fields per line
awk '{print NF}' file.txt

# Reformat output with custom separator
awk '{OFS="|"; print $1, $2, $3}' file.txt

# Remove duplicate lines (preserving order)
awk '!seen[$0]++' file.txt

# Calculate frequency of each value in column 1
awk '{count[$1]++} END {for (word in count) print word, count[word]}' file.txt

# Process CSV properly
awk -F, '{print $1, $3}' file.csv
```

## System Monitoring

### Process Management

```bash
# Find processes by name
ps aux | grep process_name

# Kill process by name
pkill process_name

# Kill process by PID
kill 1234

# Force kill process
kill -9 1234

# List all processes with full command
ps auxf

# List processes sorted by CPU usage
ps aux --sort=-%cpu | head -10

# List processes sorted by memory usage
ps aux --sort=-%mem | head -10

# Find process ID by name
pgrep process_name

# Kill all processes by user
pkill -u username

# Show process tree
pstree

# Show parent process IDs
ps -eo pid,ppid,cmd

# Find zombie processes
ps aux | awk '{print $8}' | grep Z

# Kill all processes matching pattern
ps aux | grep pattern | awk '{print $2}' | xargs kill

# Check if process is running
pgrep -x process_name > /dev/null && echo "Running" || echo "Not running"

# Monitor process in real time
watch -n 1 'ps aux | grep process_name'
```

### Resource Monitoring

```bash
# Show system load
uptime

# Show CPU usage
top -bn1 | grep "Cpu(s)"

# Show memory usage
free -h

# Show disk usage
df -h

# Show disk usage for specific directory
du -sh /path/to/directory

# Find largest files in directory
du -ah /path | sort -rh | head -20

# Find largest directories
du -h --max-depth=1 | sort -rh | head -20

# Show inode usage
df -i

# Monitor disk I/O
iostat -x 1

# Monitor network connections
netstat -tuln

# Show network statistics
ss -s

# Show open files by process
lsof -p PID

# Find what process is using a file
lsof /path/to/file

# Find what process is using a port
lsof -i :8080

# Show system information
uname -a

# Show kernel version
uname -r

# Show distribution information
cat /etc/os-release

# Show hardware info
lshw

# Show CPU info
lscpu

# Show memory info
dmidecode -t memory

# Show disk info
lsblk

# Show mounted filesystems
mount | column -t

# Show running services
systemctl list-units --type=service --state=running

# Show listening ports
ss -tulpn | grep LISTEN
```

### Log Monitoring

```bash
# Monitor log file in real time
tail -f /var/log/syslog

# Monitor with line numbers
tail -f -n 100 /var/log/syslog

# Monitor multiple logs
tail -f /var/log/*.log

# Show last 100 lines of log
tail -n 100 /var/log/syslog

# Show first 100 lines of log
head -n 100 /var/log/syslog

# Find error messages in logs
grep -i error /var/log/syslog

# Show error messages in real time
tail -f /var/log/syslog | grep -i error

# Count errors in log
grep -c ERROR /var/log/syslog

# Show logs from specific time period
grep "2024-01-15" /var/log/syslog

# Show logs from last hour
grep "$(date +'%b %e %H')" /var/log/syslog

# Find recent modifications
find /var/log -type f -mmin -60

# Archive old logs
find /var/log -type f -mtime +30 -exec gzip {} \;

# Clean old logs
find /var/log -type f -mtime +90 -delete

# Show log size
du -sh /var/log/*.log

# Compress all logs
gzip /var/log/*.log
```

## Network Operations

### Network Information

```bash
# Show IP address
ip addr show

# Show external IP
curl ifconfig.me

# Show DNS information
nslookup domain.com

# Show DNS records
dig domain.com

# Show route table
ip route show

# Show network interfaces
ip link show

# Show network statistics
ip -s link show

# Ping host
ping -c 4 host.com

# Trace route to host
traceroute host.com

# Check port connectivity
nc -zv host.com 80

# Check if port is open
nmap -p 80 host.com

# Show open ports
netstat -tuln

# Show network connections
netstat -an | grep ESTABLISHED

# Show process listening on port
lsof -i :80

# Download file
wget http://example.com/file.txt

# Download with resume
wget -c http://example.com/file.zip

# Download file with curl
curl -O http://example.com/file.txt

# Download with custom filename
curl -o newfile.txt http://example.com/file.txt

# Show HTTP headers
curl -I http://example.com

# Test website response time
curl -o /dev/null -s -w "%{time_total}\n" http://example.com

# Test website status code
curl -s -o /dev/null -w "%{http_code}" http://example.com
```

### Network Troubleshooting

```bash
# Test DNS resolution
nslookup domain.com

# Test connectivity to port
nc -zv host.com 80

# Show detailed ping statistics
ping -c 10 host.com | tail -2

# Trace route with no DNS resolution
traceroute -n host.com

# Show packet loss rate
ping -c 100 host.com | grep "packet loss"

# Test bandwidth between two hosts
iperf -c host.com

# Show bandwidth usage
iftop

# Monitor network traffic
tcpdump -i eth0

# Capture HTTP traffic
tcpdump -i eth0 -A -s 0 'tcp port 80'

# Show ARP table
arp -a

# Flush ARP cache
sudo ip neigh flush all

# Show routing table
ip route

# Add static route
sudo ip route add 192.168.1.0/24 via 192.168.1.1

# Show firewall rules
sudo iptables -L -n

# Check SSL certificate
openssl s_client -connect host.com:443 -showcerts

# Test SSL certificate expiry
echo | openssl s_client -servername host.com -connect host.com:443 2>/dev/null | openssl x509 -noout -dates
```

## File Compression and Archives

### Compression

```bash
# Compress file with gzip
gzip file.txt

# Decompress gzip file
gunzip file.txt.gz

# Compress with maximum compression
gzip -9 file.txt

# Compress directory with tar
tar -czf archive.tar.gz directory/

# Extract tar.gz archive
tar -xzf archive.tar.gz

# Extract to specific directory
tar -xzf archive.tar.gz -C /path/to/destination

# List archive contents
tar -tzf archive.tar.gz

# Compress with bzip2 (better compression)
bzip2 file.txt

# Decompress bzip2 file
bunzip2 file.txt.bz2

# Compress with xz (best compression)
xz file.txt

# Decompress xz file
unxz file.txt.xz

# Create tar.bz2 archive
tar -cjf archive.tar.bz2 directory/

# Extract tar.bz2 archive
tar -xjf archive.tar.bz2

# Create zip archive
zip -r archive.zip directory/

# Extract zip archive
unzip archive.zip

# List zip contents
unzip -l archive.zip

# Create encrypted zip
zip -e secure.zip file.txt

# Update existing archive
zip -u archive.zip newfile.txt

# Create split archive
tar -czf - directory/ | split -d -b 100M - archive.tar.gz.
```

## Data Conversion

### Format Conversion

```bash
# Convert Unix timestamp to date
date -d @1609459200

# Convert date to Unix timestamp
date -d "2021-01-01" +%s

# Convert CSV to JSON
# Using jq (assuming csv is space-separated)
cat file.csv | jq -R 'split(",") | {name: .[0], age: .[1] | tonumber}'

# Convert JSON to CSV
jq -r '.[] | [.name, .age] | @csv' file.json

# Convert tabs to spaces
expand -t 4 file.txt > output.txt

# Convert spaces to tabs
unexpand -a file.txt > output.txt

# Convert encoding
iconv -f ISO-8859-1 -t UTF-8 input.txt > output.txt

# Convert line endings (Unix to Windows)
unix2dos file.txt

# Convert line endings (Windows to Unix)
dos2unix file.txt

# Convert image format
convert input.png output.jpg

# Convert PDF to images
convert input.pdf page_%03d.png

# Convert images to PDF
convert *.images output.pdf

# Base64 encode
echo "text" | base64

# Base64 decode
echo "dGV4dA==" | base64 -d

# Encode URL
python3 -c "import urllib.parse; print(urllib.parse.quote('text'))"

# Decode URL
python3 -c "import urllib.parse; print(urllib.parse.unquote('text'))"
```

## System Management

### User Management

```bash
# Add new user
sudo adduser username

# Add user to group
sudo usermod -aG groupname username

# Delete user
sudo deluser username

# List all users
cut -d: -f1 /etc/passwd

# List all groups
cut -d: -f1 /etc/group

# Show current user
whoami

# Show user ID
id

# Show user groups
groups

# Show logged in users
who

# Show last logins
last

# Change user password
sudo passwd username

# Lock user account
sudo usermod -L username

# Unlock user account
sudo usermod -U username

# Show user details
finger username
```

### Package Management

```bash
# Debian/Ubuntu - List installed packages
dpkg -l

# Debian/Ubuntu - Search for package
apt-cache search package_name

# Debian/Ubuntu - Install package
sudo apt install package_name

# Debian/Ubuntu - Remove package
sudo apt remove package_name

# Debian/Ubuntu - Update package list
sudo apt update

# Debian/Ubuntu - Upgrade packages
sudo apt upgrade

# Debian/Ubuntu - Show package info
apt-cache show package_name

# Red Hat/CentOS - List installed packages
rpm -qa

# Red Hat/CentOS - Search for package
yum search package_name

# Red Hat/CentOS - Install package
sudo yum install package_name

# Red Hat/CentOS - Remove package
sudo yum remove package_name

# Red Hat/CentOS - Update packages
sudo yum update

# Red Hat/CentOS - Show package info
yum info package_name
```

## Docker Operations

```bash
# List all containers
docker ps -a

# List running containers
docker ps

# Stop all containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)

# Remove all stopped containers
docker container prune

# List all images
docker images

# Remove dangling images
docker image prune

# Remove all unused images
docker image prune -a

# Show container logs
docker logs container_id

# Follow container logs
docker logs -f container_id

# Execute command in container
docker exec -it container_id bash

# Show container stats
docker stats

# Show container resource usage
docker stats container_id

# Build image from Dockerfile
docker build -t image_name .

# Run container
docker run -d -p 80:80 image_name

# Remove all unused resources
docker system prune -a

# Export container to tar
docker export container_id > container.tar

# Import container from tar
docker import container.tar image_name

# Copy file from container
docker cp container_id:/path/to/file ./local_file

# Copy file to container
docker cp ./local_file container_id:/path/to/file
```

## Kubernetes Operations

```bash
# List all pods
kubectl get pods

# List pods in all namespaces
kubectl get pods --all-namespaces

# Describe pod
kubectl describe pod pod_name

# Show pod logs
kubectl logs pod_name

# Follow pod logs
kubectl logs -f pod_name

# Execute command in pod
kubectl exec -it pod_name -- bash

# List all deployments
kubectl get deployments

# Scale deployment
kubectl scale deployment deployment_name --replicas=3

# Update deployment image
kubectl set image deployment/deployment_name container_name=image:v2

# Rollback deployment
kubectl rollout undo deployment/deployment_name

# Get rollout status
kubectl rollout status deployment/deployment_name

# Get rollout history
kubectl rollout history deployment/deployment_name

# List all services
kubectl get services

# Port forward to service
kubectl port-forward service/service_name 8080:80

# Get service endpoint
kubectl get endpoints service_name

# Apply configuration
kubectl apply -f config.yaml

# Delete resource
kubectl delete -f config.yaml

# Get all resources
kubectl get all

# Get events
kubectl get events --sort-by='.lastTimestamp'
```

## Quick Reference

```bash
# File operations
find . -name "*.log" -mtime -7
grep -r "pattern" /path/to/search
sed 's/old/new/g' file.txt
awk '{print $1, $2}' file.txt

# System monitoring
ps aux | grep process
kill -9 1234
top -bn1
free -h
df -h

# Network
ping -c 4 host.com
curl ifconfig.me
netstat -tuln
lsof -i :8080

# Data processing
sort file.txt | uniq
wc -l file.txt
head -n 10 file.txt
tail -f /var/log/syslog

# Docker
docker ps
docker logs -f container_id
docker exec -it container_id bash
docker system prune -a
```
