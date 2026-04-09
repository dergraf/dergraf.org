+++
title = "Canister"
description = "A lightweight sandbox for running untrusted code safely on Linux. No root required."
weight = 5

[extra]
url = "https://github.com/dergraf/canister"
repo = "https://github.com/dergraf/canister"
language = "Rust"
+++

Canister (`can`) runs any command inside an isolated sandbox with restricted filesystem, network, and syscall access. It combines user namespaces, mount isolation, seccomp BPF, and network namespaces into a single binary -- no root, no daemon, no container runtime.

A holiday project born from the realization that we routinely execute AI-generated code and build scripts with full user privileges. Canister uses composable TOML recipes to define per-command policies: which paths are visible, which domains are reachable, which binaries may be exec'd.
