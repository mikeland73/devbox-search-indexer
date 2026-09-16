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
nix-env --file nixpkgs/default.nix \
  --query --available --meta --out-path --show-trace --json \
  --arg config '(import nixpkgs/pkgs/top-level/packages-config.nix) // { allowUnfree = true; ... }' \
  --argstr system <system>
```

Systems evaluated by default: `x86_64-linux`, `aarch64-linux`,
`x86_64-darwin`, `aarch64-darwin`. Darwin evaluates fine on Linux — this is
pure evaluation, nothing is built.

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
