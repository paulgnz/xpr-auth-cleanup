# XPR Auth Cleanup

A non-custodial web tool that removes malicious **delegated permissions** from an XPR Network
account in a **single transaction the user signs themselves** — built in response to the
`xprdrop.com` wallet-drainer campaign (see `reports/xprdrop-phishing-campaign-2026-08-03.md`).

Victims of that campaign have a permission named `claim` (child of `active`) controlled by
`xprgrant@active` and linked to up to 20+ token `transfer` actions. Cleaning that up by hand
on a block explorer means ~22 separate `unlinkauth` calls plus a `deleteauth`. This tool does
it in one signed transaction.

## What it does

1. User connects their own WebAuth/Anchor wallet (Proton Web SDK). **The tool never sees keys.**
2. Reads the account's permissions and flags any controlled by a **foreign account** (an actor
   other than the user). Known campaign controllers (`xprgrant`, `symbiote`, `xprdrops`,
   `rasmr5`) are marked **MALICIOUS**.
3. Enumerates the currently-active `linkauth` links on that permission (net of link/unlink),
   via the public Hyperion history API.
4. Builds **one** transaction: `unlinkauth` × N, then `deleteauth` (order matters — a permission
   can't be deleted while links remain; a single ordered tx handles it atomically).
5. Shows every action in plain language **and** raw JSON before the user signs.

It can **only** ever emit `unlinkauth` / `deleteauth`. It cannot move funds or add permissions.

## ⚠️ Staked-funds ordering (critical)

For a victim with **staked XPR**, the drainer's permission is usually linked to token
*transfers* but **not** to `eosio::unstakexpr`. Because the malicious permission sits *below*
`active`, it cannot unstake — so staked XPR is temporarily out of reach. But the instant it's
unstaked to liquid, the linked `eosio.token::transfer` can sweep it (and the kit's backend
watches in real time). **Always delete the permission FIRST, then unstake.** The tool detects
staked balances and shows this warning.

## Trust model — why this must live on a recognized domain

This tool is, mechanically, the same shape as the scam it fixes: "connect your wallet and sign
an auth transaction." That's exactly why it must be:

- **Radically transparent** — every action shown before signing; single readable HTML file.
- **Non-custodial** — no keys, no backend, no funds movement possible.
- **Hosted on a domain users already trust** — e.g. a subdomain of a known block producer
  (`protonnz.com`), ideally **endorsed/linked by Metallicus/WebAuth**. Do **not** promote it
  from an anonymous domain, or we train users that random "connect + sign auth" sites are safe.

## Deploy (Vercel + protonnz.com subdomain)

Static single-file site — no build step.

```bash
cd packages/auth-cleanup
vercel                     # first run: link/create project -> PREVIEW url (test here)
# ... complete the testnet checklist below on the preview URL ...
vercel --prod              # promote to production
vercel domains add cleanup.protonnz.com   # then add the CNAME it prints at your DNS host
```

DNS: add the `CNAME` (or A/ALIAS) Vercel shows for `cleanup.protonnz.com` in the protonnz.com
zone. Vercel provisions TLS automatically.

## ✅ Verify BEFORE pointing the mainnet subdomain at it

This is a prototype. Confirm all of these on a **preview URL against testnet** first:

- [ ] The Proton Web SDK actually loads and the global name is correct
      (`@proton/web-sdk` UMD — verify version `4.2.16` and `window.ProtonWebSDK`; **vendor it
      locally for production instead of loading from unpkg** — a security tool should not pull
      its core dependency from a third-party CDN. This repo is literally about supply-chain risk).
- [ ] The testnet `chainId` in `index.html` is correct for the current testnet.
- [ ] Create a test account on testnet, plant a dummy `claim` permission + a couple of
      `linkauth`s, then confirm the tool detects them and the one-tx cleanup removes them.
- [ ] Confirm the staked-funds warning appears when the account has stake.
- [ ] Re-connect after cleanup and confirm the permission is gone.

Only after all boxes are checked: `vercel --prod` and attach `cleanup.protonnz.com`.

## Note on project scope

The red-team toolkit itself remains **read-only** — it never submits transactions. This tool
does not change that: *the user* signs and broadcasts their own remediation from their own
wallet. The toolkit ships the page; it never holds keys or signs.
