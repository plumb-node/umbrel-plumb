# Plumb for Umbrel

The Umbrel app image for [Plumb](https://github.com/plumb-node/plumb), Bitcoin
Knots with every reviewed spam filter on by default. It is Leo Haf's Bitcoin
Knots app for Umbrel ([Retropex/umbrel-bitcoin](https://github.com/Retropex/umbrel-bitcoin)),
itself built on Umbrel's Bitcoin Node app, with these changes:

- `bitcoind` and `bitcoin-cli` come from the signed Plumb release tarball. The
  image build stops unless `SHA256SUMS` carries a good signature from Jason
  Sopko's key, `89F0 E41D 72CE 523F 4AA1 CDB6 92CD FFB7 C40C D1BA`
  (`plumb/keys/`), and the tarball matches it. See the `plumb` stage in
  `apps/backend/Dockerfile`.
- Plumb's four filters are switches on the Policy tab, on by default. Knots'
  own `-rejectdeadbranches` has a switch there too.
- The data size starts at 42 bytes and the data cost at 2, the values in the
  recommended `plumb.conf` on [cesspool.lol/check](https://cesspool.lol/check/).
- The app shows Plumb's name and logo.

Install it from Leo Haf's Misc community app store,
[Horizon-btc/Bitcoin-store](https://github.com/Horizon-btc/Bitcoin-store), where it
is the btc-plumb app. Step-by-step instructions, how to check it is running
Plumb, and what to do when the install fails:
[plumbnode.org/umbrel](https://plumbnode.org/umbrel/).

## Images

`ghcr.io/plumb-node/umbrel-plumb`, for linux/amd64 and linux/arm64, built by
`.github/workflows/on-tag.yml` on each `v*` tag. The run summary prints the
image with its index digest, which the store's `docker-compose.yml` pins.

For a new Plumb release, change `PLUMB_VERSION`, `PLUMB_SHA256SUMS` and the
`/opt/bitcoind` paths in `apps/backend/Dockerfile`, and the version in
`AVAILABLE_BITCOIN_KNOTS_VERSIONS` (`libs/settings/settings.meta.ts`), then tag.

Any change a user can see (store files, ports, addresses, settings labels, a
new umbrelOS release) also means walking each step on
[plumbnode.org/umbrel](https://plumbnode.org/umbrel/) and fixing it in the
same change.

---

[![License](https://img.shields.io/github/license/getumbrel/umbrel-bitcoin?color=%235351FB)](https://github.com/getumbrel/umbrel-bitcoin/blob/master/LICENSE.md)
