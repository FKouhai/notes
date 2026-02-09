---
id: jq
title: jq - JSON Processor
aliases:
  - jq-tutorial
  - json-processing
tags:
  - shell
  - json
  - jq
  - data-processing
description: Comprehensive guide to jq command-line JSON processor with practical examples
---

# jq - JSON Processor

`jq` is a lightweight and flexible command-line JSON processor. It's like `sed` for JSON data - you can slice, filter, map and transform structured data.

## Installation

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq

# Fedora
sudo dnf install jq

# From source
sudo apt-get install -y automake
git clone https://github.com/stedolan/jq.git
cd jq
autoreconf -i
./configure --disable-maintainer-mode
make
sudo make install
```

## Basic Syntax

```bash
jq '.key' file.json          # Extract value by key
jq '.key1, .key2' file.json  # Extract multiple values
jq '.key.subkey' file.json   # Nested key access
jq '.[]' file.json           # Iterate over arrays
jq '.[0]' file.json          # Access array index
jq '.[-1]' file.json         # Last element
```

## Input Examples

All examples use this sample JSON:

```json
{
  "users": [
    {"name": "Alice", "age": 30, "city": "NYC", "active": true},
    {"name": "Bob", "age": 25, "city": "LA", "active": false},
    {"name": "Charlie", "age": 35, "city": "SF", "active": true}
  ],
  "metadata": {
    "version": "1.0",
    "created": "2024-01-15"
  }
}
```

## Common Operations

### Pretty Print

```bash
# Prettify JSON (default)
echo '{"name":"Alice","age":30}' | jq '.'

# Compact JSON (remove whitespace)
echo '{"name":"Alice","age":30}' | jq -c '.'
```

### Filter Objects

```bash
# Extract specific fields
jq '.metadata.version' data.json

# Select specific fields (creates new object)
jq '{name: .name, age: .age}' data.json

# Extract all keys
jq 'keys' data.json

# Extract all values
jq 'values' data.json
```

### Array Operations

```bash
# Extract all items in array
jq '.users[]' data.json

# Extract specific array element
jq '.users[0]' data.json

# Slice array (indices 0-2)
jq '.users[0:2]' data.json

# Length of array
jq '.users | length' data.json

# Reverse array
jq '.users | reverse' data.json

# Sort array
jq '.users | sort_by(.age)' data.json

# Sort array (descending)
jq '.users | sort_by(.age) | reverse' data.json

# Group by field
jq 'group_by(.city)' data.json

# Unique values
jq 'map(.city) | unique' data.json
```

### Filter with Conditions

```bash
# Filter array (active users only)
jq '.users[] | select(.active == true)' data.json

# Multiple conditions
jq '.users[] | select(.active == true and .age > 25)' data.json

# Negate condition
jq '.users[] | select(.active != true)' data.json

# Regex match
jq '.users[] | select(.name | test("^A"))' data.json

# Contains
jq '.users[] | select(.city | contains("N"))' data.json
```

### Map and Transform

```bash
# Transform array items (extract name)
jq '.users | map(.name)' data.json

# Add new field to each item
jq '.users | map(. + {status: "online"})' data.json

# Modify field values
jq '.users | map(.age *= 2)' data.json

# Extract specific fields from array
jq '.users[] | {name, age}' data.json

# Chain transformations
jq '.users | map(select(.active)) | map(.name)' data.json
```

### String Operations

```bash
# Uppercase
jq '.users[] | .name | ascii_upcase' data.json

# Lowercase
jq '.users[] | .name | ascii_downcase' data.json

# String length
jq '.users[] | .name | length' data.json

# Split string
jq '"hello world" | split(" ")' data.json

# Join array
jq '["hello", "world"] | join(" ")' data.json

# Substring
jq '".users[0].name" | .[0:3]' data.json

# Trim
jq '"  hello  " | ltrimstr("  ") | rtrimstr("  ")' data.json

# Replace
jq '"hello world" | sub("world"; "everyone")' data.json

# Replace all occurrences
jq '"hello world hello" | gsub("hello"; "hi")' data.json
```

### Number Operations

```bash
# Basic math
jq '1 + 2' <<< ''           # 3
jq '10 - 3' <<< ''          # 7
jq '5 * 4' <<< ''           # 20
jq '20 / 4' <<< ''          # 5
jq '10 % 3' <<< ''          # 1

# Math functions
jq 'sqrt(16)' <<< ''        # 4
jq 'pow(2, 3)' <<< ''       # 8
jq 'abs(-5)' <<< ''         # 5
jq 'round(3.7)' <<< ''      # 4
jq 'floor(3.9)' <<< ''      # 3
jq 'ceil(3.1)' <<< ''       # 4
jq 'min(1, 2, 3)' <<< ''    # 1
jq 'max(1, 2, 3)' <<< ''    # 3
jq 'min_by(.age)' <<< '[{"age":25},{"age":30}]'  # {"age":25}

# Range
jq '[range(1; 5)]' <<< ''   # [1,2,3,4]

# Sum array
jq '[1,2,3,4] | add' <<< ''  # 10

# Average
jq '[1,2,3,4] | add / length' <<< ''  # 2.5
```

### Pipe Chaining

```bash
# Multiple operations
jq '.users | map(select(.active)) | map(.name) | join(", ")' data.json

# Complex transformation
jq '.users | map({name, age: .age * 2, city}) | sort_by(.age)' data.json

# Count occurrences
jq '[.users[].city] | group_by(.) | map({city: .[0], count: length})' data.json
```

### Variables

```bash
# Define and use variable
jq '.users as $users | $users | length' data.json

# Multiple variables
jq '.users as $users | $users | map(select(.active)) as $active | $active | length' data.json

# Variable with filter
jq '.users[] | select(.active) as $user | $user.name' data.json
```

### Functions

```bash
# Built-in functions
jq 'map(has("name"))' data.json                    # Check if key exists
jq 'map(keys)' data.json                           # Get keys
jq 'map(length)' data.json                         # Get length
jq 'map(type)' data.json                           # Get type

# Custom function
jq 'def add_age: .age + 1; .users[] | add_age' data.json

# Function with parameters
jq 'def multiply($x): . * $x; .age | multiply(2)' <<< '{"age": 30}'

# Multiple definitions
jq 'def inc: . + 1; def dec: . - 1; [inc, dec]' <<< '5'
```

## Real-World Examples

### API Response Processing

```bash
# Parse GitHub API response
curl -s https://api.github.com/repos/stedolan/jq | jq '{name, stars: .stargazers_count, language}'

# Extract all issue numbers
curl -s https://api.github.com/repos/stedolan/jq/issues | jq '.[].number'

# Filter open issues
curl -s https://api.github.com/repos/stedolan/jq/issues | jq '.[] | select(.state == "open") | .title'
```

### Docker Container Information

```bash
# Get all container names
docker ps --format '{{json .}}' | jq -r '.Names'

# Get container IP addresses
docker inspect $(docker ps -q) | jq '.[] | {name: .Name, ip: .NetworkSettings.IPAddress}'

# Filter running containers with image name
docker ps --format '{{json .}}' | jq 'select(.Image | contains("nginx"))'
```

### Kubernetes Resource Processing

```bash
# Extract pod names
kubectl get pods -o json | jq '.items[].metadata.name'

# Get pod IPs
kubectl get pods -o json | jq '.items[] | {name: .metadata.name, ip: .status.podIP}'

# Filter by label
kubectl get pods -o json | jq '.items[] | select(.metadata.labels.app == "web") | .metadata.name'

# Get ready pods count
kubectl get pods -o json | jq '[.items[] | select(.status.conditions[]? | .type == "Ready" and .status == "True")] | length'
```

### Log Analysis

```bash
# Parse JSON logs
cat app.log | jq '.timestamp, .level, .message'

# Filter error logs
cat app.log | jq 'select(.level == "ERROR")'

# Extract specific fields from logs
cat app.log | jq '{time: .timestamp, user: .userId, action: .action}'

# Count by level
cat app.log | jq -r '.level' | sort | uniq -c | sort -rn
```

### Configuration Management

```bash
# Update config value
jq '.timeout = 30' config.json > config.tmp && mv config.tmp config.json

# Merge configs
jq -s '.[0] * .[1]' config1.json config2.json

# Delete key
jq 'del(.metadata)' config.json > config.tmp && mv config.tmp config.json

# Add new key
jq '. + {"newKey": "newValue"}' config.json > config.tmp && mv config.tmp config.json

# Conditional update
jq 'if .production then .timeout = 60 else .timeout = 30 end' config.json > config.tmp && mv config.tmp config.json
```

### CSV/JSON Conversion

```bash
# JSON to CSV
jq -r '.users[] | [.name, .age, .city] | @csv' data.json

# JSON to TSV
jq -r '.users[] | [.name, .age, .city] | @tsv' data.json

# JSON to array of arrays
jq '[.users[] | [.name, .age, .city]]' data.json
```

### Network Operations

```bash
# Parse JSON from URL
curl -s https://api.ipify.org?format=json | jq '.ip'

# Parse multiple URLs concurrently
cat urls.txt | xargs -P 10 -I {} curl -s {} | jq -c '.id, .title'

# Extract URLs from JSON
curl -s https://api.github.com/repos/stedolan/jq | jq '.html_url, .clone_url'
```

## Advanced Patterns

### Flatten Nested Structures

```bash
# Flatten nested object
jq '{name: .name, age: .details.age, city: .details.address.city}' <<< '{"name":"Alice","details":{"age":30,"address":{"city":"NYC"}}}'

# Flatten array of nested objects
jq '.users[] | {name: .name, email: .contact.email}' <<< '{"users":[{"name":"Alice","contact":{"email":"alice@example.com"}}]}'

# Recursive flatten
jq 'paths | select(. != "deep") as $p | getpath($p)' <<< '{"a":{"b":{"c":1}},"x":2}'
```

### Merge Multiple JSON Files

```bash
# Merge two files
jq -s '.[0] + .[1]' file1.json file2.json

# Merge arrays from multiple files
jq -s 'map(.[])' file1.json file2.json file3.json

# Combine multiple objects into array
jq -s '.' file1.json file2.json file3.json
```

### Conditional Logic

```bash
# If-then-else
jq 'if .age > 30 then "senior" elif .age > 20 then "adult" else "junior" end' <<< '{"age": 35}'

# Ternary operator
jq '(.age > 30) // "senior"' <<< '{"age": 35}'

# Multiple conditions
jq '.users[] | select(.active and (.age > 25 or .city == "NYC"))' data.json
```

### Error Handling

```bash
# Provide default for missing keys
jq '.missing_key // "default"' <<< '{"existing_key": "value"}'

# Check if key exists
jq 'has("missing_key")' <<< '{"existing_key": "value"}'  # false

# Try-catch (1-based)
jq 'try .missing_key catch "key not found"' <<< '{"existing_key": "value"}'

# Null coalescing
jq '.null_field // .fallback_field // "default"' <<< '{"fallback_field": "backup"}'
```

### Output Formatting

```bash
# Raw output (no quotes)
jq -r '.name' <<< '{"name": "Alice"}'

# Null terminated output
jq -rj '.name + "\u0000"' <<< '{"name": "Alice"}'

# Compact JSON output
jq -c '.' <<< '{"name": "Alice", "age": 30}'

# Colorize output
jq -C '.' <<< '{"name": "Alice", "age": 30}'

# Tab-separated
jq -r '@tsv' <<< '["Alice", 30, "NYC"]'
```

## Performance Tips

```bash
# Use -n for streaming large files
jq -n -f filter.jq large_file.json

# Process arrays efficiently
jq '.[]' large_array.json | jq '.'

# Use --stream for very large JSON
jq --stream '.' large_file.json

# Filter before processing
jq 'select(.active) | .name' data.json  # Faster than jq '.name' data.json | grep
```

## Common Pitfalls

```bash
# Wrong: Dot notation for keys with spaces
jq '.user name' <<< '{"user name": "Alice"}'  # Error

# Right: Use bracket notation
jq '."user name"' <<< '{"user name": "Alice"}'  # Works

# Wrong: Not handling null values
jq '.missing_key' <<< '{"key": "value"}'  # Returns null

# Right: Handle null explicitly
jq '.missing_key // "default"' <<< '{"key": "value"}'  # Returns "default"

# Wrong: Not quoting keys with special characters
jq '.user-name' <<< '{"user-name": "Alice"}'  # Error

# Right: Use bracket notation
jq '."user-name"' <<< '{"user-name": "Alice"}'  # Works
```

## Testing Your Queries

```bash
# Test with sample data
echo '{"key": "value"}' | jq '.'

# Use --debug for debugging
echo '{"key": "value"}' | jq --debug '.'

# Validate JSON
jq empty <<< '{"key": "value"}'  # Exit code 0 if valid
```
