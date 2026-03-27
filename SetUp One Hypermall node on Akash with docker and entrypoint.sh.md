# 🧠 HYPERMALL VALIDATOR SETUP TUTORIAL FOR ONE NODE (v0.1)

## 1️⃣ Project structure

Create a working folder for your runtime:

```bash
mkdir hypermall-runtime
cd hypermall-runtime
```
---

## 2️⃣ Dockerfile

Create `Dockerfile`:

```
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y \
    openssh-server \
    sudo \
    curl \
    wget \
    git \
    nano \
    npm \
    tmux \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir /var/run/sshd

# Create ONLY User1
RUN useradd -m -s /bin/bash user1 && \
    usermod -aG sudo user1

# Prepare persistent storage
RUN mkdir -p /data/hypermall/User1 && chown -R user1:user1 /data

WORKDIR /home/user1

# Copy Entrypoint
COPY Entrypoint.sh /Entrypoint.sh
RUN chmod +x /Entrypoint.sh

# Expose SSH
EXPOSE 22

# Enable root login via SSH with password
RUN sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config

ENTRYPOINT ["/Entrypoint.sh"]
```
---
## 3️⃣ Entrypoint script

Create `Entrypoint.sh`:

```
#!/bin/bash

echo "=== Hypermall Runtime (Single User: User1) ==="

# Validate environment variables
if [ -z "$ROOT_PASSWORD" ] || [ -z "$USER1_PASSWORD" ]; then
  echo "ERROR: ROOT_PASSWORD or USER1_PASSWORD not set"
  exit 1
fi

# Set root and User1 passwords
echo "root:$ROOT_PASSWORD" | chpasswd
echo "user1:$USER1_PASSWORD" | chpasswd

# Start SSH service
service ssh start

# Prepare persistent storage
mkdir -p /data/hypermall/User1
chown -R user1:user1 /data

# Switch to User1 environment
su - user1 << 'EOF'

cd /data/hypermall/User1

# Install Hypermall node only if not present
if [ ! -f "msb.mjs" ]; then
  echo "Installing Hypermall node (trac-msb@0.1.82)..."

  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
  source ~/.nvm/nvm.sh
  nvm install 22

  npm install -g pear
  npm install trac-msb@0.1.82  # PINNED for Hypermall

  cp -r node_modules/trac-msb/* .
  npm install
fi

# Start validator in a single tmux session
if ! tmux has-session -t hypermall 2>/dev/null; then
  echo "Starting tmux session 'hypermall'..."
  tmux new-session -d -s hypermall \
    "cd /data/hypermall/User1 && node msb.mjs run . store1"
else
  echo "Tmux session 'hypermall' already running"
fi

EOF

# Keep container alive
tail -f /dev/null
```
---

## 4️⃣ Build and push Docker image

```bash
docker build -t bbbmining21/hypermall-runtime:v0.1 .
docker push bbbmining21/hypermall-runtime:v0.1
```
---

## 5️⃣ SDL (Akash deployment file)

```yaml
version: "2.0"

services:
  runtime:
    image: bbbmining21/hypermall-runtime:v0.1
    env:
      - "ROOT_PASSWORD=supersecret"
      - "USER1_PASSWORD=user1secret"
    expose:
      - port: 22
        as: 22
        to:
          - global: true
    params:
      storage:
        data:
          mount: /data

profiles:
  compute:
    runtime:
      resources:
        cpu:
          units: 8
        memory:
          size: 16Gi
        storage:
          - size: 50Gi
          - name: data
            size: 500Gi
            attributes:
              persistent: true
              class: beta3

  placement:
    dcloud:
      pricing:
        runtime:
          denom: uact
          amount: 1000

deployment:
  runtime:
    dcloud:
      profile: runtime
      count: 1
```
---
## 6️⃣ Deploy and verify

1. Deploy the SDL via Akash CLI.
2. SSH into the container:

```bash
ssh root@<CONTAINER_IP>
```

3. Check tmux sessions:

```bash
tmux ls
```

Expected output:

```
hypermall
```

4. Attach to a validator session:

```bash
tmux attach -t hypermall
```

5. Node logs are visible. Validators are running inside the correct `/data/hypermall/UserX` directories.
