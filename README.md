## Why run Hypermall & TRAC Validator nodes on Akash?

Short answer: it’s often cheaper, gives you more control, and avoids some of the usual cloud headaches (AWS outages).

This setup comes from actually running nodes on Akash — not just testing it once.

---

### Costs

This is the main reason most people look at Akash.

Instead of fixed pricing (like AWS or similar), you’re picking from a marketplace of providers.

What that means in practice:

* you’ll usually pay quite a bit less
* you’re not locked into anything
* you can redeploy if prices change

It’s not always the absolute cheapest option at every moment, but for long-running nodes it tends to be worth it.

---

### Control

With normal cloud providers, you’re always depending on one company:

* account limits
* policy changes
* random restrictions

On Akash, you’re dealing with individual providers instead.

If something doesn’t work:

* you can just redeploy somewhere else
* no tickets, no waiting

That alone makes running nodes less annoying.

---

### Privacy

You’re not running everything inside one big centralized platform, which means:

* less visibility into your setup
* less coupling to a single provider

For node operators, that’s generally a plus.

---

### Makes sense for Web3

If you’re already running a TRAC or Hypermall validator node, you’re in a decentralized ecosystem anyway.

Running it on centralized infrastructure always felt a bit off.

Akash is not perfect, but it’s closer to the idea:

* distributed providers
* no single point of control

---

### About this repo

This is basically a working setup, not a theoretical guide.

It includes:

* configs that actually run
* a setup process that’s been tested
* a baseline you can adjust if needed

So you don’t have to start from zero.

*Don´t be scared of trying out this node installment method! It may all seem difficult at first glance, but you actually just copy paste a couple lines, change your password, click a couple times and then just either create a new wallet or import your seed phrase and you´re done.*

*As long as your server is funded, the validator runs all by itself indefinitely (until there´s an update for the Hypermall or Mainnet node runtime)*

---

## Bottom line

If you’re running a node long-term, Akash is worth trying.

* usually cheaper
* fewer provider-related issues
* more flexibility

There’s a small learning curve, but once it’s running, it’s pretty straightforward.

*If you have the courage and would like to run more than two nodes you will have to alter the according Dockerfile and entrypoint.sh, build and push those to your docker hub and alter the SDL so that it pulls your docker container. You can use the SetUp docs for two nodes as reference.*

*[Two Hypermall AND Trac Mainnet nodes](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/SetUp%20TWO%20Hypermall%20AND%20Trac%20Mainnet%20validator%20nodes%20on%20Akash.md)*
*OR*
*[Two Hypermall nodes](https://github.com/bbbmining21/Trac-Validator-on-Akash-VPS/blob/main/SetUp%20Two%20Hypermall%20nodes%20on%20Akash%20with%20docker%20and%20Entrypoint.sh.md)*
