# 🧱 Hypermall Validator on Akash (Automated Setup Guide)

> Run **2 fully automated Hypermall validator nodes** on Akash using a **self-healing runtime container** with persistent storage.

---

# ⚡ 🔍 Quick Overview

This setup will:

✔️ Deploy a ready-to-run validator environment

✔️ Automatically install and start nodes

✔️ Recover after container restarts

✔️ Run nodes in the background using `tmux`

---

## 🧠 For Experienced Cosmos Users

👉 You can skip **Step 1 (Wallet & Funding)** if you already:

* Use Keplr Wallet
* Have funds on the **Akash chain**

➡️ Continue directly with **Step 2 [Understanding the Setup](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/edit/main/GettingStarted.md#-step-2)**

---

# 🔐 Step 1 — Wallet & Funding

## Install Keplr Wallet

👉 [https://www.keplr.app](https://www.keplr.app)

---

## 💰 Fund Your Wallet

### Option A — USDC via Noble

👉 [https://express.noble.xyz/transfer](https://express.noble.xyz/transfer)

**Steps:**

1. Connect wallet
2. Send USDC → Noble
3. Funds arrive in Keplr

---

### Option B — Swap to AKT

👉 [https://app.osmosis.zone](https://app.osmosis.zone)

**Steps:**

1. Deposit USDC
2. Swap → AKT
3. Withdraw to Akash

---

📸 *Screenshot placeholder:*
`/images/keplr-wallet.png`
`/images/osmosis-swap.png`

---

# 📦 Step 2 
— What You’re Deploying

You are deploying a **prebuilt Docker runtime** that:

* Installs Hypermall automatically
* Creates user environments
* Starts validator nodes
* Restores after restarts

---

## 🔍 (Optional) Verify the Runtime

* Entrypoint script:
  [https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/SetUp%20Hypermall%20node%20on%20Akash%20with%20docker%20and%20Entrypoint.sh.md](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/SetUp%20Hypermall%20node%20on%20Akash%20with%20docker%20and%20Entrypoint.sh.md)

* Docker image:
  [https://hub.docker.com/repository/docker/bbbmining21/hypermall-runtime/tags/v0.5](https://hub.docker.com/repository/docker/bbbmining21/hypermall-runtime/tags/v0.5)

---

# 🚀 Step 3 — Deploy Your Node

Go to:

👉 [https://console.akash.network/sdl-builder](https://console.akash.network/sdl-builder)

---

## 📋 Paste This SDL

```yaml
version: "2.0"
services:
  hypermall-test-runtime:
    image: bbbmining21/hypermall-runtime:v0.5
    expose:
      - port: 22
        as: 22
        to:
          - global: true
    env:
      - ROOT_PASSWORD=supersecret
      - USER1_PASSWORD=user1secret
      - USER2_PASSWORD=user2secret
    params:
      storage:
        data:
          mount: /data
          readOnly: false

profiles:
  compute:
    hypermall-test-runtime:
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
        hypermall-test-runtime:
          denom: uakt
          amount: 10000

deployment:
  hypermall-test-runtime:
    dcloud:
      profile: hypermall-test-runtime
      count: 1
```

---

## 🔐 🚨 Change Passwords Before Deploying

```bash
ROOT_PASSWORD=yourpassword
USER1_PASSWORD=yourpassword
USER2_PASSWORD=yourpassword
```

---

📸 *Screenshot placeholder:*
`/images/akash-sdl-builder.png`

---

## ▶️ Deploy

* Click **Deploy**
* Approve transaction
* Wait ~2–5 minutes

---

# 🔌 Step 4 — Connect to Server

```bash
ssh root@YOUR_IP
```

---

📸 *Screenshot placeholder:*
`/images/ssh-login1.png`
`/images/ssh-login2.png`
`/images/ssh-login3.png`

---

# 🧵 Step 5 — tmux Sessions
What Is tmux?
---
Your nodes run inside **tmux sessions**.

👉 tmux allows processes to run in the background:

* Close terminal → node keeps running
* Disconnect SSH → node keeps running


Command to list tmux sessions:

```bash
tmux ls
```

Expected:

```bash
mall1
mall2
```

---

## ▶️ Attach (Connect to your tmux session, where your node is either already running or needs to be run)

```bash
tmux attach -t mall1
```
Inside User1

Respectively

```bash
tmux attach -t mall2
```
Inside User2

These commands are not to be run under root. So respectively, for each user, open two new windows in your terminal and log into them to interact with the corresponding tmux instance (mall1 for User1 and mall2 for User2) 

---

## ⏹ Detach (Go back to the terminal without closing your node)

```
Ctrl + b, then d
```

---

📸 *Screenshot placeholder:*
`/images/tmux-session.png`

---

# 📌 Step 6 — Identity Setup

When starting for the first time:

---

## 🆕 Create New Identity

```bash
1
```

✔ Creates new validator address
✔ Needed for onboarding
📸 *Screenshot placeholder:*
`/images/msb-seed.png`

---

## 🔁 Restore Existing Identity

```bash
2
```

Then:

* Paste 24-word seed phrase
* Press Enter

---
## 🔐 Backup Identity

```bash
cp stores/store1/db/keypair.json /your/backup/location/
```
---

# 🧵 tmux Recovery (If Closed Accidentally)

Reattach:

```bash
tmux attach -t mall1
```

---

## If Session Is Gone

```bash
su - User1
cd /data/hypermall/User1
node msb.mjs run . store1
```

## ⛔ Important Rule

* mall1 → store1
* mall2 → store2
* Never run same store twice⚠️ It will cause an error.

---

# 🧾 Step 7 — Whitelisting

👉 [https://onboarding.trac.network](https://onboarding.trac.network)

---

## Steps:

1. Connect Bitcoin wallet
2. Link license → MSB identity
3. Wait up to 24h

---

## Final Step

```bash
/add_writer
```

---

📸 *Screenshot placeholder:*
`/images/onboarding.png`

---

# 💰 You’re Live

* Node validates trades
* Fees accumulate
* Uptime = earnings

---

# 🧠 Summary

| Component      | Role                 |
| -------------- | -------------------- |
| Keplr          | Wallet               |
| Akash          | Compute/Cloud/Server |
| Docker runtime | Automation           |
| /data          | Persistent storage   |
| tmux           | Background execution |
| MSB            | Hypermall Validator  |

---

# ⚠️ Why is Persistent Storage added (Important)

Akash containers are **ephemeral**:

* `/root`, `/home` → wiped
* `/data` → persistent

👉 Your node stores everything in `/data` and **auto-recovers**
---

# 🔚 Final Notes

This setup is:

* ⚡ Automated
* 🔁 Self-recovering
* 🧵 Background-managed
* 🔐 Identity-safe

---
# 📸 Suggested Screenshot List

Create an `/images` folder with:

```
keplr-wallet.png
osmosis-swap.png
akash-sdl-builder.png
ssh-login.png
tmux-session.png
msb-seed.png
onboarding.png
```

---
