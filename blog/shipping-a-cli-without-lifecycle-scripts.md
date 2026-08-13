# Shipping platform-specific binaries via npm (without lifecycle scripts)

There is a comforting belief in JavaScript tooling that install time is a good time to do work.

If you want to ship a platform-specific binary via npm, the default advice is always the same: download it in `postinstall`. That is how most native CLIs do it, and most of the time it works.

Until it does not.

This post is about how a reasonable setup collapsed under its own correctness, and why the fix was not another lifecycle hook but removing lifecycle hooks entirely.

## The obvious approach

The MagicBell CLI is a Go binary, distributed via npm. The common pattern is familiar. You publish a small npm package, define a `bin` entry, and download the real binary during `postinstall`.

Simple. Boring. Usually reliable.

Our CLI lives inside a monorepo. Running `yarn install` at the repo root installs everything, including the CLI. That means `postinstall` runs during local development, while the CLI binary for that version often does not exist yet.

Downloading a binary that has not been published is a problem.

So we avoided running `postinstall` locally.

We used `pinst`. The repository contained `_postinstall`, which was enabled during `prepack` and reverted during `postpack`. Local installs were safe, published packages contained a real `postinstall`, and the git tree stayed clean.

This is where most setups stop thinking.

## When things get weird

Users then reported something odd. Running:

```sh
npm i -g magicbell-cli
```

…did nothing.

But running:

```sh
npm i -g ./magicbell-cli-1.2.0.tgz
```

worked fine. Same package. Same version. Same contents. Different outcome.

Looking closer revealed something uncomfortable. Registry metadata showed this:

```sh
npm view magicbell-cli@1.2.0 scripts --json
```

```json
{
  "_postinstall": "node src/install.js install"
}
```

But the tarball itself contained:

```sh
curl -sL "$(npm view magicbell-cli@1.2.0 dist.tarball)" \
  | tar -xzOf - package/package.json
```

```json
{
  "postinstall": "node src/install.js install"
}
```

Both are technically correct. That is the problem.

npm uses registry metadata to decide whether install scripts exist. If the metadata says there are no install scripts, npm may skip lifecycle execution entirely.

The tarball did contain `postinstall`, but the registry metadata said it did not. npm trusted the metadata, and `postinstall` never ran. Installing from a local tarball bypassed the metadata, which is why it worked.

We tried more hooks. `prepack`, `postpack`, `postpublish`, even forcing npm instead of Yarn. Some combinations worked in isolation, but none worked everywhere. `npm i -g`, `npx`, Yarn, and pnpm all behaved slightly differently.

Every fix depended on install-time side effects. And those are inherently fragile.

## Flipping the model

The insight was simple. We did not need the binary at install time. We only needed it when the CLI runs.

So we stopped installing.

The npm package now ships a tiny Node.js wrapper as its `bin`. On execution, it checks a cache directory for the real binary, downloads it if it is missing, and then spawns it with `stdio: inherit`.

From the user's perspective:

```sh
cat data.json | magicbell broadcast
```

still works. Pipes, exit codes, signals, unchanged.

There are no lifecycle scripts, no install-time side effects, no differences between `npm i -g` and `npx`, and no monorepo edge cases. The npm package is now a launcher, not an installer. That turns out to be a better abstraction.

Registry metadata matters more than tarball contents. `postinstall` is not a contract. If correctness depends on `prepack`, you do not control your build. Lazy execution beats eager installation.

If your CLI only needs a binary when it runs, do not download it before it runs.

This change did not just fix a bug. It removed an entire class of failure.

By moving work from install time to run time, the npm package stops being an installer and becomes a launcher. That shift eliminates lifecycle ordering issues, monorepo edge cases, and differences between `npm i -g` and `npx` in one move.

The broader lesson is simple. If a tool only needs something when it runs, defer the work until it runs. Install-time side effects feel convenient, but they scale poorly across ecosystems.

Or, more bluntly, do not make your correctness depend on npm doing the right thing.

