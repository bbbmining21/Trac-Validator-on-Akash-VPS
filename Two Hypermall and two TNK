## 🧠 HYPERMALL AND TRAC MAINNET VALIDATOR SETUP TUTORIAL FOR TWO USERS (v0.4)

This setup runs:

* **User1**

  * Hypermall → `store1`
  * TNK → `store3`

* **User2**

  * Hypermall → `store2`
  * TNK → `store4`

Each runs in its own **tmux session**.

---

# 1️⃣ Project structure

```bash
mkdir hypermall-runtime
cd hypermall-runtime
```

---

# 2️⃣ Dockerfile

Create `Dockerfile`:

```dockerfile
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

# Create users
RUN useradd -m -s /bin/bash User1 && \
    useradd -m -s /bin/bash User2

# Enable root login
RUN sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config

# Prepare persistent dirs
RUN mkdir -p \
    /data/hypermall/User1 \
    /data/hypermall/User2 \
    /data/TNK/User1 \
    /data/TNK/User2 && \
    chown -R User1:User1 /data && \
    chown -R User2:User2 /data

# Copy entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 22

ENTRYPOINT ["/entrypoint.sh"]
```

---

# 3️⃣ Entrypoint script

Create `entrypoint.sh`:

```bash
#!/bin/bash
set -e

INIT_FILE="/data/.initdone"

echo "=== Hypermall + TNK runtime boot (2 users) ==="

mkdir -p /data
chmod 777 /data || true

# -------- env check --------
if [ -z "$ROOT_PASSWORD" ] || [ -z "$USER1_PASSWORD" ] || [ -z "$USER2_PASSWORD" ]; then
    echo "ERROR: missing passwords"
    exit 1
fi

# -------- passwords --------
echo "root:${ROOT_PASSWORD}" | chpasswd
echo "User1:${USER1_PASSWORD}" | chpasswd
echo "User2:${USER2_PASSWORD}" | chpasswd

# -------- ensure users --------
id -u User1 >/dev/null 2>&1 || useradd -m -s /bin/bash User1
id -u User2 >/dev/null 2>&1 || useradd -m -s /bin/bash User2

# -------- init dirs --------
if [ ! -f "$INIT_FILE" ]; then
    mkdir -p /data/hypermall/User1 /data/hypermall/User2
    mkdir -p /data/TNK/User1 /data/TNK/User2

    chown -R User1:User1 /data/hypermall/User1 /data/TNK/User1
    chown -R User2:User2 /data/hypermall/User2 /data/TNK/User2

    touch "$INIT_FILE"
fi

# =========================
# USER1 SETUP
# =========================
su - User1 << EOF

# ---- Hypermall (User1) ----
cd /data/hypermall/User1

if [ ! -f "msb.mjs" ]; then
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
    export NVM_DIR="\$HOME/.nvm"
    [ -s "\$NVM_DIR/nvm.sh" ] && \. "\$NVM_DIR/nvm.sh"
    nvm install 22

    npm install -g pear
    npm init -y
    npm install trac-msb@0.1.82
    cp -r node_modules/trac-msb/* .
fi

if ! tmux has-session -t hypermall1 2>/dev/null; then
    tmux new-session -d -s hypermall1 \
    "cd /data/hypermall/User1 && while true; do node msb.mjs run . store1; sleep 5; done"
fi

# ---- TNK (User1) ----
cd /data/TNK/User1

if [ ! -f "msb.mjs" ]; then
    npm init -y
    npm install trac-msb@0.2.2
    cp -fr node_modules/trac-msb/* .
    npm install || true
fi

if ! tmux has-session -t TNK1 2>/dev/null; then
    tmux new-session -d -s TNK1 \
    "cd /data/TNK/User1 && while true; do node msb.mjs run . store3; sleep 5; done"
fi

EOF

# =========================
# USER2 SETUP
# =========================
su - User2 << EOF

# ---- Hypermall (User2) ----
cd /data/hypermall/User2

if [ ! -f "msb.mjs" ]; then
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
    export NVM_DIR="\$HOME/.nvm"
    [ -s "\$NVM_DIR/nvm.sh" ] && \. "\$NVM_DIR/nvm.sh"
    nvm install 22

    npm install -g pear
    npm init -y
    npm install trac-msb@0.1.82
    cp -r node_modules/trac-msb/* .
fi

if ! tmux has-session -t hypermall2 2>/dev/null; then
    tmux new-session -d -s hypermall2 \
    "cd /data/hypermall/User2 && while true; do node msb.mjs run . store2; sleep 5; done"
fi

# ---- TNK (User2) ----
cd /data/TNK/User2

if [ ! -f "msb.mjs" ]; then
    npm init -y
    npm install trac-msb@0.2.2
    cp -fr node_modules/trac-msb/* .
    npm install || true
fi

if ! tmux has-session -t TNK2 2>/dev/null; then
    tmux new-session -d -s TNK2 \
    "cd /data/TNK/User2 && while true; do node msb.mjs run . store4; sleep 5; done"
fi

EOF

# -------- SSH --------
exec /usr/sbin/sshd -D
```

---

# 4️⃣ Build and push

```bash
docker build --no-cache -t bbbmining21/hypermall-runtime:v0.4 .
docker push bbbmining21/hypermall-runtime:v0.4
```

---

# 5️⃣ SDL (UPDATED FOR 2 USERS)

```yaml
version: "2.0"

services:
  runtime:
    image: bbbmining21/hypermall-runtime:v0.4
    env:
      - "ROOT_PASSWORD=supersecret"
      - "USER1_PASSWORD=user1secret"
      - "USER2_PASSWORD=user2secret"
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

# 🎯 Running Sessions

| User  | Node      | Store  | tmux         |
| ----- | --------- | ------ | ------------ |
| User1 | Hypermall | store1 | `hypermall1` |
| User1 | TNK       | store3 | `TNK1`       |
| User2 | Hypermall | store2 | `hypermall2` |
| User2 | TNK       | store4 | `TNK2`       |

---

# 🔌 Attach Sessions

```bash
tmux attach -t hypermall1
tmux attach -t TNK1
tmux attach -t hypermall2
tmux attach -t TNK2
```

Detach:

```
CTRL + B, then D
```

---

# 🚀 Result

You now have:

* 4 validator processes
* 2 independent users
* full isolation via directories + tmux
* persistent storage on Akash
* identical behavior to your working single-user setup
