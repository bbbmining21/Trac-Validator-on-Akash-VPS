# 🧱 Run One or More Akash Trac Validator Nodes on One Server

> This is a step-by-step guide for running **multiple Trac validator nodes** (`trac-msb`) in **separate user environments** on a single Akash container using **Ubuntu 22.04**. This includes proper persistent storage, login configuration, updates, and node launching.

---

## 🔰 What’s This For?

This tutorial helps you:

* Launch a validator node inside Akash Cloud (https://console.akash.network/sdl-builder (click on 'import')).
* Run **two separate nodes** in the same container.
* Avoid permission issues and data loss.
* Use **persistent volumes** so data survives reboots.
* Understand the full process from **deployment to management**.

---

## 📦 0. What Is an SDL?

SDL stands for **Stack Definition Language**. It’s like a recipe file that tells Akash:

* What **OS image** to use (`ubuntu:22.04`)
* What **software** to install when the server boots
* What **storage**, **CPU**, and **memory** you need
* What **ports** to expose
* How to configure users and passwords

You deploy this file clicking on the red 'Deploy' button.

Please get familiar with the Keplr Wallet (and the COSMOS Ecosystem) beforehand -if you haven´t done this yet-, as payment is done in either $AKT or $USDC (Noble).

---

## 📋 Example SDL (Your Deployment File)

Below is your finalized SDL — ready to deploy:

```yaml
version: "2.0"

services:
  ubuntu:
    image: ubuntu:22.04
    command:
      - /bin/bash
      - -c
      - |
        apt-get update && apt-get install -y openssh-server && \
        mkdir /var/run/sshd && \
        sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
        sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
        echo 'root:AssignRootPasswordHere' | chpasswd && \
        useradd -m -s /bin/bash UnnamedPlayer1 && \
        echo 'UnnamedPlayer1:AssignPasswordHere' | chpasswd && \
        useradd -m -s /bin/bash UnnamedPlayer2 && \
        echo 'UnnamedPlayer2:AssignPasswordHere' | chpasswd && \
        service ssh start && \
        tail -f /dev/null
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
    ubuntu:
      resources:
        cpu:
          units: 8
        memory:
          size: 16Gi
        storage:
          - size: 20Gi
          - name: data
            size: 500Gi
            attributes:
              persistent: true
              class: beta3

  placement:
    dcloud:
      pricing:
        ubuntu:
          denom: uakt
          amount: 10000

deployment:
  ubuntu:
    dcloud:
      profile: ubuntu
      count: 1
```

### 🔐 Replace the Passwords

Replace:

* `AssignRootPasswordHere` → your root password
* `AssignPasswordHere` → each user’s login password (choose something secure)

---

## 🧰 1. First Time Setup (as `root` in Akash container)

After SSH’ing into your container as `root`, run:

```bash
apt update && apt install -y \
  curl git build-essential python3 tmux
```

Install Node.js:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
```

---

## 🗂 2. Create Project Directories

Make persistent folders for your validator nodes:

```bash
mkdir -p /data/trac-validator/UnnamedPlayer1
mkdir -p /data/trac-validator/UnnamedPlayer2

chown -R UnnamedPlayer1:UnnamedPlayer1 /data/trac-validator/UnnamedPlayer1
chown -R UnnamedPlayer2:UnnamedPlayer2 /data/trac-validator/UnnamedPlayer2
```

---

## 📦 3. Install `trac-msb` Validator (as `root`, once per user)

Repeat this block for **each user’s folder**:

```bash
cd /data/trac-validator/UnnamedPlayer1
npm init -y
npm install trac-msb
cp -fr node_modules/trac-msb/* .
npm install
```

And then for user 2:

```bash
cd /data/trac-validator/UnnamedPlayer2
npm init -y
npm install trac-msb
cp -fr node_modules/trac-msb/* .
npm install
```

✅ This installs the validator directly into persistent storage. Do **not** install outside of `/data` — it will be deleted on reboot.

---

## 🔒 4. Fix Permissions

```bash
chmod -R ugo+rwX /data/trac-validator
```

This ensures all users can read/write everything they need.

---

## 🧵 5. Start Validator Nodes via `tmux`

Since `pm2` and other tools may fail in Akash containers, use `tmux` to run each node in its own terminal tab in the background.

### For UnnamedPlayer1

```bash
su - UnnamedPlayer1
cd /data/trac-validator/UnnamedPlayer1
tmux new -s msb1
node msb.mjs run . store1
```

Detach from `tmux`: press `Ctrl + b`, then `d`

### For UnnamedPlayer2

```bash
su - UnnamedPlayer2
cd /data/trac-validator/UnnamedPlayer2
tmux new -s msb2
node msb.mjs run . store2
```

---

## 🔄 How to Update `trac-msb`

1. Reconnect via SSH.

2. Attach to the `tmux` session and stop the node with `Ctrl + C`:

   ```bash
   tmux attach -t msb1
   # Press Ctrl+C
   ```

3. Then run the update steps:

```bash
cd /data/trac-validator/UnnamedPlayer1
npm install trac-msb
cp -fr node_modules/trac-msb/* .
npm install

cd /data/trac-validator/UnnamedPlayer2
npm install trac-msb
cp -fr node_modules/trac-msb/* .
npm install
```

4. Relaunch the node with the same `node` command in its `tmux` session.

---

## 📁 Why Everything Goes Into `/data/`

Akash containers **wipe all files** from paths like:

* `/root/`
* `/home/`
* `/tmp/`

But `/data/` is **persistent** thanks to your SDL config. That’s why:

* All `npm install` commands go inside `/data`
* The validator runs **only** from `/data`
* RocksDB and key files stay safe across reboots

---

## 🧠 Summary

| Task                                | Why                                         |
| ----------------------------------- | ------------------------------------------- |
| SDL sets up Ubuntu + users          | Defines your Akash container setup          |
| Everything goes in `/data`          | Akash wipes other folders                   |
| Use `tmux`                          | Lightweight way to run background processes |
| Avoid shared stores like `stores2/` | Prevents file permission conflicts          |

---

After the set up is done and your node is running, please continue with the steps from here https://docs.trac.network/documentation/validators/installation/hypermall#post-install
