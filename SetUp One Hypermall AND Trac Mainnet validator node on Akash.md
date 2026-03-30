# 🧠 HYPERMALL AND TRAC MAINNET VALIDATOR SETUP TUTORIAL FOR ONE NODE (v0.3)

## 1️⃣ Project structure

Create a working folder for your runtime:

```bash
mkdir hypermall-runtime
cd hypermall-runtime
```
---

## 2️⃣ Dockerfile

Create `Dockerfile`:

```bash
nano Dockerfile
```
```

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl git tmux nano ca-certificates \
    openssh-server build-essential python3 \
    apt-transport-https \
    && rm -rf /var/lib/apt/lists/*

# NodeJS 20 (base)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
 && apt-get install -y nodejs

# Prepare SSH runtime
RUN mkdir /var/run/sshd

# Create User1
RUN useradd -m -s /bin/bash User1

# Enable root login
RUN sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config

# Prepare persistent dirs
RUN mkdir -p /data/hypermall/User1 /data/TNK/User1 && \
    chown -R User1:User1 /data

# Copy entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 22

ENTRYPOINT ["/entrypoint.sh"]
```
---
## 3️⃣ Entrypoint script

Create `entrypoint.sh`:
```bash
nano entrypoint.sh
```
```
#!/bin/bash
set -e

INIT_FILE="/data/.initdone"

echo "=== Hypermall + TNK runtime boot ==="

# -------- storage --------
mkdir -p /data
chmod 777 /data || true

# -------- env check --------
if [ -z "$ROOT_PASSWORD" ] || [ -z "$USER1_PASSWORD" ]; then
    echo "ERROR: missing passwords"
    exit 1
fi

# -------- passwords --------
echo "root:${ROOT_PASSWORD}" | chpasswd
echo "User1:${USER1_PASSWORD}" | chpasswd

# -------- ensure user --------
id -u User1 >/dev/null 2>&1 || useradd -m -s /bin/bash User1

# -------- init dirs --------
if [ ! -f "$INIT_FILE" ]; then
    mkdir -p /data/hypermall/User1
    mkdir -p /data/TNK/User1

    chown -R User1:User1 /data
    touch "$INIT_FILE"
fi

# =========================
# RUN EVERYTHING AS USER1
# =========================
su - User1 << EOF

# ---------- HYPERMALL ----------
cd /data/hypermall/User1

if [ ! -f "msb.mjs" ]; then
    echo "Installing Hypermall (0.1.82)..."

    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
    export NVM_DIR="\$HOME/.nvm"
    [ -s "\$NVM_DIR/nvm.sh" ] && \. "\$NVM_DIR/nvm.sh"
    nvm install 22

    npm install -g pear
    npm init -y
    npm install trac-msb@0.1.82

    cp -r node_modules/trac-msb/* .
fi

if ! tmux has-session -t hypermall 2>/dev/null; then
    tmux new-session -d -s hypermall \
    "cd /data/hypermall/User1 && while true; do node msb.mjs run . store1; sleep 5; done"
fi


# ---------- TNK ----------
cd /data/TNK/User1

if [ ! -f "msb.mjs" ]; then
    echo "Installing TNK (0.2.2)..."

    npm init -y
    npm install trac-msb@0.2.2

    if [ -d "node_modules/trac-msb" ]; then
        cp -fr node_modules/trac-msb/* .
    else
        echo "ERROR: trac-msb not found"
    fi

    npm install || true
fi

if ! tmux has-session -t TNK1 2>/dev/null; then
    tmux new-session -d -s TNK1 \
    "cd /data/TNK/User1 && while true; do node msb.mjs run . store3; sleep 5; done"
fi

EOF

# -------- SSH config --------
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config

echo "Starting SSH..."
exec /usr/sbin/sshd -D
```

## 4️⃣ Build and push Docker image

```bash
# Build the Docker image from scratch, ignoring cache
docker build --no-cache -t bbbmining21/hypermall-runtime:v0.3 .

# Push the newly built image to Docker Hub
docker push bbbmining21/hypermall-runtime:v0.3
```
---

## 5️⃣ SDL (Akash deployment file)

```yaml
version: "2.0"

services:
  runtime:
    image: bbbmining21/hypermall-runtime:v0.3
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
## Attach Sessions

```bash
tmux attach -t hypermall
tmux attach -t TNK1
```
Detach: Ctrl + B, then D
