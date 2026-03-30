# 🧠 HYPERMALL VALIDATOR SETUP TUTORIAL FOR TWO NODES (v0.5)

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
```Dockerfile
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

# Pre-create validator users
RUN useradd -m -s /bin/bash User1 \
 && useradd -m -s /bin/bash User2

# Copy entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

✅ Notes:

* Users **User1** and **User2** exist internally.
* Passwords are **set dynamically via SDL env variables**.
* No global Hypermall CLI is required.

---

## 3️⃣ Entrypoint script

Create `entrypoint.sh`:
```bash
nano entrypoint.sh
```
```bash
#!/bin/bash
# DO NOT exit on error (important for debugging)
set +e

INIT_FILE="/data/.initdone"

echo "=== Hypermall runtime container boot ==="

# -------- basic info --------
echo "User: $(whoami)"
echo "Node version: $(node -v)"
echo "NPM version: $(npm -v)"

# -------- ensure persistent disk --------
mkdir -p /data
chmod 777 /data || true

# -------- SSH config --------
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
service ssh restart || service ssh start || true

# -------- ensure users --------
id -u User1 >/dev/null 2>&1 || useradd -m -s /bin/bash User1
id -u User2 >/dev/null 2>&1 || useradd -m -s /bin/bash User2

# -------- set passwords --------
echo "root:${ROOT_PASSWORD}" | chpasswd
echo "User1:${USER1_PASSWORD}" | chpasswd
echo "User2:${USER2_PASSWORD}" | chpasswd

# -------- first boot init --------
if [ ! -f "$INIT_FILE" ]; then
    echo "Initializing Hypermall directories"

    mkdir -p /data/hypermall/User1
    mkdir -p /data/hypermall/User2

    chown -R User1:User1 /data/hypermall/User1
    chown -R User2:User2 /data/hypermall/User2

    touch "$INIT_FILE"
fi

# -------- debug user environments --------
echo "Checking User1 environment:"
su - User1 -c "whoami && node -v && npm -v"

echo "Checking User2 environment:"
su - User2 -c "whoami && node -v && npm -v"

# -------- install Hypermall locally --------
install_hypermall () {
    USER=$1
    DIR=$2

    echo "Checking installation for $USER in $DIR"
    ls -la $DIR

    if [ ! -f "$DIR/msb.mjs" ]; then
        echo "Installing Hypermall for $USER"
        su - "$USER" -c "cd $DIR && npm install trac-msb@0.1.82 && cp -fr node_modules/trac-msb/* . && npm install" \
        || echo "❌ INSTALL FAILED for $USER"
    else
        echo "Hypermall already installed for $USER"
    fi

    echo "Post-install check:"
    ls -la $DIR
}

install_hypermall User1 /data/hypermall/User1
install_hypermall User2 /data/hypermall/User2

# -------- start validator nodes --------
start_node () {
    SESSION=$1
    USER=$2
    DIR=$3
    STORE=$4

    echo "Starting validator $SESSION for $USER"

    su - "$USER" -c "tmux has-session -t $SESSION" 2>/dev/null
    if [ $? -eq 0 ]; then
        echo "$SESSION already running"
        return
    fi

    if [ ! -f "$DIR/msb.mjs" ]; then
        echo "❌ Cannot start $SESSION — msb.mjs missing"
        return
    fi

    su - "$USER" -c \
    "tmux new -d -s $SESSION 'cd $DIR && while true; do node msb.mjs run . $STORE; sleep 5; done'" \
    || echo "❌ FAILED to start tmux session $SESSION"
}

start_node mall1 User1 /data/hypermall/User1 store1
start_node mall2 User2 /data/hypermall/User2 store2

echo "=== Container ready ==="

# -------- keep container alive --------
tail -f /dev/null
```

✅ Notes:

* Hypermall is installed **locally in `/data/hypermall/UserX`**.
* Each validator runs in **its own tmux session**.
* Auto-restart loop ensures nodes restart if they crash.
* User passwords are **configurable via SDL env**.

---

## 4️⃣ Build and push Docker image

```bash
docker build -t bbbmining21/hypermall-runtime:v0.5 .
docker push bbbmining21/hypermall-runtime:v0.5
```

---

## 5️⃣ SDL (Akash deployment file)

```yaml
version: "2.0"

services:
  runtime:
    image: bbbmining21/hypermall-runtime:v0.5
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

✅ Notes:

* Persistent storage ensures Hypermall data is kept between redeploys.
* Passwords can be changed without rebuilding the image.
* Usernames remain **fixed internally** (`User1`, `User2`) for storage consistency.

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
mall1
mall2
```

4. Attach to a validator session:

```bash
tmux attach -t mall1
```

5. Node logs are visible. Validators are running inside the correct `/data/hypermall/UserX` directories.

---

## 7️⃣ Summary

✅ Hypermall installed locally per user
✅ Nodes start automatically in correct directories
✅ Persistent storage ensures data survives redeploys
✅ Passwords configurable via SDL
✅ Auto-restart loop prevents downtime
✅ Fully Akash-compatible, production-ready container
