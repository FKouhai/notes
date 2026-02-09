---
id: yq
title: yq - YAML Processor
aliases:
  - yq-tutorial
  - yaml-processing
tags:
  - shell
  - yaml
  - yq
  - data-processing
description: Comprehensive guide to yq command-line YAML processor with practical examples
---

# yq - YAML Processor

`yq` is a portable command-line YAML processor. It's like `jq` but for YAML - you can slice, filter, map and transform YAML data with ease.

## Installation

```bash
# Using snap (Linux)
sudo snap install yq

# Using brew (macOS)
brew install yq

# Using chocolatey (Windows)
choco install yq

# Download binary directly
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq
chmod +x /usr/bin/yq

# Go install
go install github.com/mikefarah/yq/v4@latest
```

## Basic Syntax

```bash
# Read YAML file
yq '.key' file.yaml

# Write to file
yq '.key = "value"' file.yaml -i

# Output as JSON
yq -o json '.key' file.yaml

# Output as YAML (default)
yq -o yaml '.key' file.yaml
```

## Input Examples

All examples use this sample YAML:

```yaml
name: myapp
version: 1.0.0
metadata:
  author: John Doe
  created: 2024-01-15
services:
  - name: web
    port: 8080
    enabled: true
    replicas: 3
  - name: db
    port: 3306
    enabled: true
    replicas: 1
  - name: cache
    port: 6379
    enabled: false
    replicas: 0
environments:
  production:
    url: prod.example.com
    database:
      host: db.prod.example.com
      port: 5432
  development:
    url: dev.example.com
    database:
      host: db.dev.example.com
      port: 5432
```

## Basic Operations

### Read Values

```bash
# Simple key access
yq '.name' config.yaml

# Nested key access
yq '.metadata.author' config.yaml

# Array index access
yq '.services[0].name' config.yaml

# Negative index (from end)
yq '.services[-1].name' config.yaml

# Iterate over array
yq '.services[]' config.yaml

# Iterate with index
yq '.services[] | select(.name == "web")' config.yaml
```

### Filter Arrays

```bash
# Select by value
yq '.services[] | select(.name == "web")' config.yaml

# Select enabled services
yq '.services[] | select(.enabled == true)' config.yaml

# Multiple conditions
yq '.services[] | select(.enabled == true and .replicas > 1)' config.yaml

# Negate condition
yq '.services[] | select(.enabled != true)' config.yaml

# String contains
yq '.services[] | select(.name | contains("e"))' config.yaml

# Regex match
yq '.services[] | select(.name | test("^w"))' config.yaml
```

### Modify Values

```bash
# Set simple value
yq '.name = "newapp"' config.yaml -i

# Set nested value
yq '.metadata.author = "Jane Doe"' config.yaml -i

# Set array element
yq '.services[0].replicas = 5' config.yaml -i

# Set multiple values
yq '.name = "newapp" | .version = "2.0.0"' config.yaml -i

# Set based on condition
yq '.services[] |= select(.name == "web") | .replicas = 10' config.yaml -i
```

### Add Fields

```bash
# Add new key
yq '.newkey = "newvalue"' config.yaml -i

# Add nested key
yq '.metadata.newfield = "value"' config.yaml -i

# Add field to array elements
yq '.services[] += {status: "running"}' config.yaml -i

# Add field conditionally
yq '.services[] |= select(.name == "web") |= .+ {status: "active"}' config.yaml -i
```

### Delete Fields

```bash
# Delete simple key
yq 'del(.metadata)' config.yaml -i

# Delete specific key
yq 'del(.metadata.created)' config.yaml -i

# Delete from array elements
yq '.services[] | del(.enabled)' config.yaml -i

# Delete array element by index
yq 'del(.services[0])' config.yaml -i

# Delete array element by condition
yq 'del(.services[] | select(.name == "cache"))' config.yaml -i
```

## Array Operations

### Map and Transform

```bash
# Extract specific field from array
yq '.services[].name' config.yaml

# Extract multiple fields (creates new objects)
yq '.services[] | {name, port, enabled}' config.yaml

# Transform array elements
yq '.services[] | .replicas *= 2' config.yaml

# Add field to all array elements
yq '.services[] |= .+ {type: "service"}' config.yaml

# Chain operations
yq '.services | map(select(.enabled)) | map(.name)' config.yaml
```

### Array Filtering and Sorting

```bash
# Filter and extract
yq '.services[] | select(.enabled) | .name' config.yaml

# Sort by field
yq '.services | sort_by(.port)' config.yaml

# Sort descending
yq '.services | sort_by(.port) | reverse' config.yaml

# Unique values
yq '[.services[].port] | unique' config.yaml

# Group by field
yq '.services | group_by(.enabled)' config.yaml
```

### Array Length and Counting

```bash
# Array length
yq '.services | length' config.yaml

# Count matching elements
yq '[.services[] | select(.enabled)] | length' config.yaml

# Count by field value
yq '[.services[].enabled] | group_by(.) | map({enabled: .[0], count: length})' config.yaml
```

## String Operations

```bash
# Uppercase
yq '.name | ascii_upcase' config.yaml

# Lowercase
yq '.name | ascii_downcase' config.yaml

# String length
yq '.name | length' config.yaml

# Split string
yq '.metadata.author | split(" ")' config.yaml

# Join array
yq '["hello", "world"] | join(" ")' config.yaml

# Substring
yq '.name[0:3]' config.yaml

# Trim
yq ' "  hello  " | ltrimstr("  ") | rtrimstr("  ")' config.yaml

# Replace
yq '.name | sub("app"; "service")' config.yaml

# Replace all
yq '.name | gsub("a"; "x")' config.yaml
```

## Number Operations

```bash
# Basic math
yq '1 + 2' <<< ''
yq '10 - 3' <<< ''
yq '5 * 4' <<< ''
yq '20 / 4' <<< ''
yq '10 % 3' <<< ''

# Math functions
yq 'sqrt(16)' <<< ''
yq 'pow(2, 3)' <<< ''
yq 'abs(-5)' <<< ''
yq 'round(3.7)' <<< ''
yq 'floor(3.9)' <<< ''
yq 'ceil(3.1)' <<< ''

# Array operations
yq '[1, 2, 3] | add' <<< ''
yq '[1, 2, 3] | length' <<< ''
yq '[1, 2, 3] | min' <<< ''
yq '[1, 2, 3] | max' <<< ''
yq '[1, 2, 3] | sort' <<< ''
yq '[1, 2, 3] | reverse' <<< ''

# Range
yq '[range(1; 5)]' <<< ''
```

## Advanced Operations

### Pipe and Chain

```bash
# Multiple operations
yq '.services | map(select(.enabled)) | map(.name) | join(", ")' config.yaml

# Complex transformation
yq '.services | map({name, port: .port + 1000, status}) | sort_by(.port)' config.yaml

# Nested operations
yq '.environments.production.database | {host, port}' config.yaml
```

### Variables

```bash
# Define variable
yq '.services as $s | $s | length' config.yaml

# Multiple variables
yq '.services as $s | $s | map(select(.enabled)) as $a | $a | length' config.yaml

# Variable with filter
yq '.services[] | select(.enabled) as $s | $s.name' config.yaml
```

### Conditional Logic

```bash
# If-then-else
yq 'if .name == "myapp" then "production" else "dev" end' config.yaml

# Ternary-like
yq '(.services[0].enabled) // "disabled"' config.yaml

# Multiple conditions
yq '.services[] | select(.enabled and (.replicas > 0)) | .name' config.yaml
```

### Functions

```bash
# Built-in functions
yq 'has("name")' config.yaml
yq 'keys' config.yaml
yq 'values' config.yaml
yq 'type' config.yaml

# Custom function
yq 'def double_replicas: .replicas *= 2; .services[] | double_replicas' config.yaml -i

# Function with parameter
yq 'def multiply($x): . * $x; .services[].replicas |= multiply(2)' config.yaml -i

# Multiple functions
yq 'def inc: . + 1; def dec: . - 1; .services[].replicas |= inc' config.yaml -i
```

## Real-World Examples

### Kubernetes Manifests

```bash
# Update image tag in Deployment
yq '.spec.template.spec.containers[0].image = "myapp:v2.0"' deployment.yaml -i

# Update replicas based on environment
yq '.spec.replicas = 5' deployment.yaml -i

# Add environment variable
yq '.spec.template.spec.containers[0].env += {name: "NEW_VAR", value: "value"}' deployment.yaml -i

# Update ConfigMap
yq '.data["DB_HOST"] = "new-db.example.com"' configmap.yaml -i

# Merge multiple YAML files
yq '.resources[]' deployment.yaml service.yaml configmap.yaml

# Extract specific resources from multi-doc YAML
yq 'select(.kind == "Deployment")' k8s-manifest.yaml

# Extract all service names
yq '.metadata.name' k8s-manifest.yaml

# Find all ConfigMaps
yq 'select(.kind == "ConfigMap") | .metadata.name' k8s-manifest.yaml
```

### CI/CD Configuration

```bash
# Update GitLab CI image
yq '.image = "node:18-alpine"' .gitlab-ci.yml -i

# Add GitLab CI variable
yq '.variables += {NEW_VAR: "value"}' .gitlab-ci.yml -i

# Update GitHub Actions step
yq '.jobs.build.steps[0].uses = "actions/checkout@v4"' .github/workflows/build.yml -i

# Update Docker Compose image version
yq '.services.web.image = "myapp:v2.0"' docker-compose.yml -i

# Update Docker Compose ports
yq '.services.web.ports += ["8081:8080"]' docker-compose.yml -i

# Add environment to Docker Compose service
yq '.services.web.environment += {DEBUG: "true"}' docker-compose.yml -i
```

### Application Configuration

```bash
# Update database connection string
yq '.database.url = "postgresql://user:pass@new-host:5432/db"' config.yaml -i

# Toggle feature flags
yq '.features.newFeature = true' config.yaml -i

# Update log level
yq '.logging.level = "debug"' config.yaml -i

# Update server port
yq '.server.port = 8081' config.yaml -i

# Add new API endpoint
yq '.apis += {path: "/new", method: "GET"}' config.yaml -i

# Remove deprecated setting
yq 'del(.legacy.enabled)' config.yaml -i
```

### Helm Chart Values

```bash
# Update Helm values
yq '.image.tag = "v2.0"' values.yaml -i

# Update replica count
yq '.replicaCount = 5' values.yaml -i

# Enable/disable feature
yq '.enabled = true' values.yaml -i

# Add ingress host
yq '.ingress.hosts += ["new.example.com"]' values.yaml -i

# Update resource limits
yq '.resources.requests.cpu = "500m"' values.yaml -i

# Set environment-specific values
yq '.production.replicaCount = 10' values.yaml -i
yq '.development.replicaCount = 1' values.yaml -i
```

### Terraform Variables

```bash
# Update Terraform variable default
yq '.variable.instance_type.default = "t3.large"' variables.tf -i

# Update variable description
yq '.variable.instance_type.description = "New description"' variables.tf -i

# Add new variable
yq '.variable.new_var = {type: "string", default: "value"}' variables.tf -i

# Update module source
yq '.module.vpc.source = "git::https://github.com/example/vpc-module"' main.tf -i
```

### Multi-Document YAML

```bash
# Split multi-doc YAML into individual files
yq 'select(documentIndex == 0)' multi.yaml > doc0.yaml
yq 'select(documentIndex == 1)' multi.yaml > doc1.yaml

# Merge multiple YAML files
yq '.' file1.yaml file2.yaml > merged.yaml

# Concatenate arrays from multiple files
yq '.[]' file1.yaml file2.yaml > combined.yaml

# Select specific document
yq 'select(.kind == "Deployment")' k8s-resources.yaml

# Count documents
yq 'length' multi.yaml
```

## Data Conversion

```bash
# YAML to JSON
yq -o json '.' config.yaml > config.json

# JSON to YAML
yq -p=json -o=yaml '.' config.json > config.yaml

# YAML to CSV (flat structure)
yq -o=csv '.users[] | [.name, .age, .city]' config.yaml

# YAML to XML
yq -o=xml '.' config.yaml > config.xml

# Format output
yq -y '.' config.yaml  # YAML (default)
yq -j '.' config.yaml  # JSON
yq -p=props '.' config.yaml  # Properties
```

## Complex Queries

### Nested Access

```bash
# Deep nested access
yq '.environments.production.database.host' config.yaml

# Extract nested arrays
yq '.environments.*.database.host' config.yaml

# Multiple levels of nesting
yq '.metadata.author' config.yaml
```

### Merge and Combine

```bash
# Merge objects
yq '. * {"newKey": "newValue"}' config.yaml

# Merge arrays
yq '. + [.new.service]' config.yaml

# Update specific element in array
yq '.services[] |= select(.name == "web") |= .+ {replicas: 10}' config.yaml -i

# Remove duplicates from array
yq '.services | unique_by(.name)' config.yaml
```

### Path Operations

```bash
# Get all keys
yq 'keys' config.yaml

# Get all paths to values
yq '.. | select(type == "!!str")' config.yaml

# Get paths matching pattern
yq '.services[] | path(select(.enabled == true))' config.yaml

# Delete by path
yq 'del(.environments.production)' config.yaml -i
```

## Practical Use Cases

### Environment-Specific Configs

```bash
# Extract production config
yq '.environments.production' config.yaml > prod-config.yaml

# Extract development config
yq '.environments.development' config.yaml > dev-config.yaml

# Switch environment
yq '.current = "production"' config.yaml -i
```

### Secrets Management

```bash
# Remove sensitive data
yq 'del(.secrets)' config.yaml > config-public.yaml

# Extract secrets for vault
yq '.secrets' config.yaml > secrets.yaml

# Mask specific fields
yq '.password = "*****"' config.yaml -i
```

### Config Validation

```bash
# Check if required field exists
yq '.required_field' config.yaml

# Validate version format
yq '.version | test("^\\d+\\.\\d+\\.\\d+$")' config.yaml

# Check array length
yq '.services | length > 0' config.yaml
```

### Backup and Restore

```bash
# Backup config with timestamp
cp config.yaml config.yaml.backup.$(date +%Y%m%d_%H%M%S)

# Restore from backup
yq '.environments.production' config.yaml.backup.20240115_120000 > prod-env.yaml

# Compare configs
diff <(yq '.' config.yaml) <(yq '.' config.yaml.backup)
```

## Advanced Features

### Recursion

```bash
# Recursive search for key
yq '.. | .name? // empty' config.yaml

# Recursive modification
yq '.. | select(type == "!!int") |= . + 1' config.yaml

# Deep merge
yq '.deep * {nested: {value: "updated"}}' config.yaml
```

### Null Handling

```bash
# Default value for null
yq '.optional // "default"' config.yaml

# Check for null
yq '.optional == null' config.yaml

# Remove null values
yq 'del(.. | select(. == null))' config.yaml
```

### Error Handling

```bash
# Safe access (no error if key doesn't exist)
yq '.optional?' config.yaml

# Multiple fallbacks
yq '.primary // .secondary // "default"' config.yaml

# Try-catch pattern
yq '(.version // empty) // "unknown"' config.yaml
```

## Performance Tips

```bash
# Process large files efficiently
yq '.services[]' large-file.yaml | yq '.'

# Use eval for complex operations
yq eval '.services | map(select(.enabled))' config.yaml

# Read from stdin
cat config.yaml | yq '.name'

# Multiple expressions
yq '.name, .version' config.yaml
```

## Common Pitfalls

```bash
# Wrong: Dot notation for keys with special characters
yq '.user-name' <<< 'user-name: value'  # Error

# Right: Use bracket notation
yq '."user-name"' <<< 'user-name: value'  # Works

# Wrong: Not handling null
yq '.missing_key' <<< 'key: value'  # Returns null

# Right: Handle null
yq '.missing_key // "default"' <<< 'key: value'  # Returns "default"

# Wrong: In-place editing multiple times
yq '.key1 = "val1"' file.yaml -i
yq '.key2 = "val2"' file.yaml -i  # Creates backup each time

# Right: Combine operations
yq '.key1 = "val1" | .key2 = "val2"' file.yaml -i
```

## Quick Reference

```bash
# Read value
yq '.key' file.yaml

# Set value
yq '.key = "value"' file.yaml -i

# Delete key
yq 'del(.key)' file.yaml -i

# Select array element
yq '.array[0]' file.yaml

# Filter array
yq '.array[] | select(.enabled)' file.yaml

# Update array element
yq '.array[] |= select(.name == "x") | .value = "new"' file.yaml -i

# Add field
yq '. += {newkey: "value"}' file.yaml -i

# Convert formats
yq -o json '.' file.yaml
yq -p json -o yaml '.' file.json

# Multi-doc YAML
yq 'select(documentIndex == 0)' file.yaml
```
