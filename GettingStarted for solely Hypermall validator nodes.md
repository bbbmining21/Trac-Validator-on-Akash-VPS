# 🧱 Hypermall Validator on Akash (Automated Setup Guide)

> Run **one or two fully automated Hypermall validator nodes** on Akash using a **self-healing runtime container** with persistent storage.

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

➡️ Continue directly with **Step 2 [Understanding the Setup](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/GettingStarted%20for%20solely%20Hypermall%20validator%20nodes.md#-step-2)**

---

# 🔐 Step 1 — Wallet & Funding

## Install Keplr Wallet

👉 [https://www.keplr.app](https://www.keplr.app)

---

## 💰 Fund Your Wallet (Example given with USDC but you can also deposit BTC directly to Osmosis)

### USDC via Noble

👉 [https://express.noble.xyz/transfer](https://express.noble.xyz/transfer)

**Steps:**

1. Connect wallet
2. Send USDC → Noble
3. Funds arrive in Keplr

### Swap to AKT

👉 [https://app.osmosis.zone](https://app.osmosis.zone)

**Steps:**

1. Deposit USDC
2. Swap → AKT
3. Withdraw to Akash
4. Mint ACT

# ☝🏼You aswell can fund your wallet with credit card, if you don´t care about KYC and privacy☝🏼
For that you will have to create a new account and log in.
Just [Sign In](https://console.akash.network/login?tab=login&returnTo=%2F) here.

Skip signing in and just connect your Keplr wallet, if you don´t wanna pay in fiat!

![Sign In](/images/console-sign-in.png)
---

![Keplr Wallet](/images/keplr-wallet.png)
![Osmosis Swap](/images/osmosis-swap.png)
![Mint ACT](/images/mint-ACT.png)

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
  [Entrypoint1](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/SetUp%20Hypermall%20node%20on%20Akash%20with%20docker%20and%20Entrypoint.sh.md)
  [Entrypoint2](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/SetUp%20Two%20Hypermall%20nodes%20on%20Akash%20with%20docker%20and%20Entrypoint.sh.md)

* Docker image:
  [Docker Image1](https://hub.docker.com/repository/docker/bbbmining21/hypermall-runtime/tags/v0.2)
  [Docker Image2](https://hub.docker.com/repository/docker/bbbmining21/hypermall-runtime/tags/v0.5)
---

# 🚀 Step 3 — Deploy Your Node

Go to:

👉 [https://console.akash.network/sdl-builder](https://console.akash.network/sdl-builder)

---

## 📋 Paste This SDL for ONE node

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
          denom: uact
          amount: 1000

deployment:
  runtime:
    dcloud:
      profile: hypermall-runtime-one-node
      count: 1
```
---
## OR

## 📋 Paste This SDL for TWO nodes

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
          denom: uact
          amount: 1000

deployment:
  hypermall-test-runtime:
    dcloud:
      profile: hypermall-runtime-two-nodes
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
![Akash SDL Builder](/images/akash-sdl-builder.png)

---

## ▶️ Deploy

* Click **Deploy**
* Approve transaction
* Wait ~2 minutes

---

# 🔌 Step 4 — Connect to Server

```bash
ssh root@YOUR_IP
```

---
![SSH Login data source1](/images/ssh-login1.png)
![SSH Login data source2](/images/ssh-login2.png)
![SSH Login over terminal like MobaXterm](/images/ssh-login3.png)

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
hypermall
```
# OR with two nodes
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
![How a tmux session might look like](/images/tmux-session.png)
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

![Copy the seed phrase!](/images/msb-seed.png)

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

👉 [Onboarding with your license](https://onboarding.tracvalidator.com)

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
![Successfully running the node](/images/onboarding.png)
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
