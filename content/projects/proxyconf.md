+++
title = "ProxyConf"
description = "Spec-driven API management for Envoy Proxy. Control plane built with Elixir/OTP."
weight = 1

[extra]
url = "https://proxyconf.com"
repo = "https://github.com/proxyconf/proxyconf"
language = "Elixir"
+++

ProxyConf is an Elixir/OTP control plane for [Envoy Proxy](https://www.envoyproxy.io/) that uses OpenAPI specifications as the source of truth for API management. Instead of manually configuring routes and policies, you describe your APIs via specs and ProxyConf generates the corresponding Envoy configuration, including request validation via [API-Fence](/projects/api-fence/).

An ongoing exploration of how far spec-driven development can go at the proxy layer.
