# 🧠 HYPERMALL VALIDATOR SETUP TUTORIAL FOR ONE NODE (v0.2)

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

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl git tmux nano ca-certificates \
    openssh-server build-essential python3 \
    apt-transport-https \
    && rm -rf /var/lib/apt/lists/*

# NodeJS 20
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
 && apt-get install -y nodejs

# Prepare SSH runtime
RUN mkdir /var/run/sshd

# Create only User1
RUN useradd -m -s /bin/bash User1

# Enable root login with password and fix PAM
RUN sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config

# Prepare persistent data directory
RUN mkdir -p /data/hypermall/User1 && chown -R User1:User1 /data

# Copy entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Expose SSH
EXPOSE 22

ENTRYPOINT ["/entrypoint.sh"]
```
---
## 3️⃣ Entrypoint script

Create `entrypoint.sh`:

```
#!/bin/bash
set -e

INIT_FILE="/data/.initdone"

echo "=== Hypermall runtime container boot ==="

# -------- persistent disk --------
mkdir -p /data
chmod 777 /data || true

# -------- check environment variables --------
if [ -z "$ROOT_PASSWORD" ] || [ -z "$USER1_PASSWORD" ]; then
    echo "ERROR: ROOT_PASSWORD or USER1_PASSWORD not set"
    exit 1
fi

# -------- set passwords --------
echo "root:${ROOT_PASSWORD}" | chpasswd
echo "User1:${USER1_PASSWORD}" | chpasswd

# -------- ensure User1 home directory --------
id -u User1 >/dev/null 2>&1 || useradd -m -s /bin/bash User1

# -------- first boot init --------
if [ ! -f "$INIT_FILE" ]; then
    echo "Initializing Hypermall directory for User1"
    mkdir -p /data/hypermall/User1
    chown -R User1:User1 /data/hypermall/User1
    touch "$INIT_FILE"
fi

# -------- install Hypermall node --------
su - User1 << EOF
cd /data/hypermall/User1

if [ ! -f "msb.mjs" ]; then
    echo "Installing Hypermall node (trac-msb@0.1.82)..."

    # Install NVM and Node 22
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
    export NVM_DIR="\$HOME/.nvm"
    [ -s "\$NVM_DIR/nvm.sh" ] && \. "\$NVM_DIR/nvm.sh"
    nvm install 22

    # Install npm packages
    npm install -g pear
    npm init -y
    npm install trac-msb@0.1.82

    # Copy node modules for validator
    cp -r node_modules/trac-msb/* .
else
    echo "Hypermall already installed"
fi

# -------- start validator in tmux --------
if ! tmux has-session -t hypermall 2>/dev/null; then
    echo "Starting tmux session 'hypermall'..."
    tmux new-session -d -s hypermall \
    "cd /data/hypermall/User1 && while true; do node msb.mjs run . store1; sleep 5; done"
else
    echo "Tmux session 'hypermall' already running"
fi
EOF

# -------- configure SSH for root login --------
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config

echo "Starting SSH daemon in foreground (root login enabled)..."
exec /usr/sbin/sshd -D
```
---

## 4️⃣ Build and push Docker image

```bash
# Build the Docker image from scratch, ignoring cache
docker build --no-cache -t bbbmining21/hypermall-runtime:v0.2 .

# Push the newly built image to Docker Hub
docker push bbbmining21/hypermall-runtime:v0.2
```
---

## 5️⃣ SDL (Akash deployment file)

```yaml
version: "2.0"

services:
  runtime:
    image: bbbmining21/hypermall-runtime:v0.2
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
          denom: uakt
          amount: 10000

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
