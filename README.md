> **Archived.** The eval workflow now lives in
> [mikeland73/devbox-search](https://github.com/mikeland73/devbox-search)
> (`.github/workflows/eval.yml`, `eval.nix`), which is public and runs it on
> the same 16 GB runners. This repository is kept read-only so the issue and
> run links referenced from that code stay resolvable.
# devbox-search-indexer

Evaluates [nixpkgs](https://github.com/NixOS/nixpkgs) commits with `nix-env`
on GitHub Actions and archives the resulting package metadata as gzipped JSON.

That's all this repository does. It has no application code — one workflow,
[`eval.yml`](.github/workflows/eval.yml), dispatched with a commit hash. It
exists as a separate public repo because evaluating all of nixpkgs needs more
memory than GitHub's private-repo runners provide, and public repos get the
larger free runners.

## What it produces

For each `(commit, system)` pair, one object in an S3-compatible bucket:

```
{system}/{committer-unix-seconds}-{commit-hash}.json.gz
```

containing the output of

```sh
nix-env --file eval.nix \
  --query --available --meta --out-path --show-trace --json \
  --arg config '(import nixpkgs/pkgs/top-level/packages-config.nix) // { allowUnfree = true; ... }' \
  --argstr system <system>
```

[`eval.nix`](eval.nix) is nixpkgs with one adjustment: every top-level
derivation is listed under its own name even when a nested attribute that
`nix-env` visits first aliases it (`buildbotPackages.python` would otherwise
hide `python314`). The comment in the file explains the mechanism.

Systems evaluated by default: `x86_64-linux`, `aarch64-linux`,
`aarch64-darwin`. (`x86_64-darwin` was dropped by nixpkgs 26.11.) Darwin
evaluates fine on Linux — this is pure evaluation, nothing is built.

The Nix version is pinned (`NIX_VERSION` in the workflow) because the output
shape — which packages appear, how ones that refuse to evaluate are listed —
is `nix-env` behaviour and has changed between releases. Each object carries
the version that produced it as `x-amz-meta-nix-version`. Bumping the pin is
a deliberate change; check the consumer against the new output first.

## Running it

```sh
gh workflow run eval.yml \
  -f commit=<40-hex nixpkgs sha> \
  -f committed_at=<ISO 8601 committer date>
```

Secrets required: `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT`,
`R2_BUCKET`. Any S3-compatible store works; the endpoint is not hardcoded.

Only the repository owner can dispatch it, and workflow runs from forks don't
receive secrets.
