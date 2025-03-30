# Docker Workshop: Instructor Setup Guide

## VM Requirements
- 32 cores, 64GB RAM (DigitalOcean CPU-Optimized droplet)
- Ubuntu 22.04 LTS
- DNS: vm.selise.dev, *.cst.selise.dev

## Setup Steps

### 2. Install Docker (Official Method)

```bash
# Update package index
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg net-tools apache2-utils nginx git htop
```
# Add Docker's official GPG key
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
# Set up the repository
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
# Install Docker Engine
```
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Change docker config
```bash
# Create Nginx configuration
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "default-address-pools": [
    {
      "base": "192.168.0.0/16",
      "size": 24
    }
  ]
}
EOF
```
```
systemctl restart docker
```
### 3. Configure Nginx for Dynamic Proxying

```bash
# Create Nginx configuration
sudo tee /etc/nginx/conf.d/cst.conf > /dev/null << 'EOF'
server {
    server_name "~^(?<student_id>\d{5})\.cst\.selise\.dev$";

    location / {
        proxy_pass http://127.0.0.1:$student_id;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/nginx/letsencrypt/live/cst.selise.dev/fullchain.pem;
    ssl_certificate_key /etc/nginx/letsencrypt/live/cst.selise.dev/privkey.pem;
    include /etc/nginx/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/nginx/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
EOF
```

# Test Nginx configuration
```
sudo nginx -t
sudo systemctl restart nginx
```

### 4. Add UNIQUE_ID generator

```bash
sudo tee /usr/local/bin/unique_id > /dev/null << 'EOF'

#!/bin/bash

# Script to extract port number from either email or student ID
# Usage: ./get_port.sh <email_or_id>
# Example: ./get_port.sh 02210240.cst@rub.edu.bt
#          ./get_port.sh 02210240

function generate_port() {
  local id=$1
  
  # Extract the required digits to form the port number
  # - Take digits 3-4 (position 2-3 in 0-indexed)
  # - Take digits 6-8 (position 5-7 in 0-indexed)
  middle_digits="${id:2:2}"
  last_digits="${id:5:3}"
  
  # Combine to create port number
  echo "${middle_digits}${last_digits}"
}

# Check if argument is provided
if [ $# -eq 0 ]; then
  echo "Error: Please provide an email address or student ID."
  echo "Usage: $0 <email_or_id>"
  echo "Example: $0 02210240.cst@rub.edu.bt"
  echo "         $0 02210240"
  exit 1
fi

input=$1

# Check if input contains @ symbol (email)
if [[ $input == *"@"* ]]; then
  # Extract ID part before the @ symbol
  full_id=$(echo "$input" | cut -d '@' -f 1)
  # Extract just the numeric part from the ID (before any possible dot)
  id=$(echo "$full_id" | cut -d '.' -f 1)
  echo "Email: $input"
  echo "ID: $id"
else
  # Input is already an ID
  id=$input
  echo "ID: $id"
fi

# Check if ID has correct format (at least 8 digits)
if [[ ! $id =~ ^[0-9]{8,}$ ]]; then
  echo "Error: ID format is incorrect. It should be at least 8 digits."
  echo "Examples: 02210240.cst@rub.edu.bt or 02210240"
  exit 1
fi

# Generate port number
port=$(generate_port "$id")
echo "Port number: $port"

# Verify port is in valid range
if (( port < 1024 || port > 65535 )); then
  echo "Warning: Port $port is outside the recommended range (1024-65535)."
fi

# Display the export command for easy copy-paste
echo ""
echo "Copy and paste this command to set your unique port:"
echo "-------------------------------------------------"
echo "export UNIQUE_ID=$port"
echo "-------------------------------------------------"

exit 0
EOF
# Make the script executable
sudo chmod +x /usr/local/bin/unique_id
```


# Create a working directory
```
mkdir workspace
```


# Add DNS record for vm.selise.dev

```
scp -r /Users/teknath/cst/letsencrypt root@104.248.145.177:/etc/nginx
```


