<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/logo.svg">
    <img
      src="assets/logo.svg"
      alt="Voliere logo: a bird flying in a cage, its beak through the bars"
      width="160"
      height="160"
    >
  </picture>
</p>

# Voliere

A self-hosted sandbox for autonomous coding agents, built on Proxmox.

[![Project Status: Concept](https://www.repostatus.org/badges/latest/concept.svg)](https://www.repostatus.org/#concept)

> [!NOTE]
> **Status: concept.** There's no release yet. This repository holds the design for now.
>
> Voliere is deliberately opinionated about its tools: Proxmox, Tailscale,
> Gitea, Coolify, Cloudflare, and T3 Code. It's built around my own needs first,
> and support for alternatives may come later.

Voliere is a collection of OpenTofu and Ansible files that turns a Proxmox host
into a sandbox where coding agents run unattended,
with permission prompts off. You send a spec and get reviewed pull requests with
preview URLs back, on hardware and accounts you control.

Voliere runs no AI models itself. The agents use their providers' hosted
models, through your subscriptions or API keys.

Voliere assumes a prompt injection will eventually get root access inside the sandbox,
so it enforces isolation from outside the VMs.

## Stack

- **Proxmox** runs the VMs, and its firewall filters their egress.
- **Tailscale** provides cross-device access, and its ACL limits what the VMs can reach.
- **T3 Code** orchestrates worker and reviewer agents of your choice.
- **Gitea** holds the agents' repositories, with branch protection that requires CI
  and a review.
- **Coolify** deploys previews behind **Cloudflare Tunnel** and Access.
- **OpenTofu** and **Ansible** rebuild the entire environment in under an hour.

## Hardware

Voliere is designed for relatively cheap hardware. I run it on an HP EliteDesk
800 G5 Mini (Core i5-9500T, 32 GB RAM, 512 GB SSD). A cloud server that runs
Proxmox works too. Multi-node setups may come later.

## Documentation

The [design](docs/design.md) covers requirements, architecture, network
boundaries, and the agent workflow.

## Contributing

Issues that question a design decision are welcome.

## Name

Volière (pronounced "vol-YAIR") is French for an aviary.

## License

Voliere is licensed under the [GNU Affero General Public License v3.0](LICENSE).
