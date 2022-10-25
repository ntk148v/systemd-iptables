# Systemd Iptables Rules Management

- [Systemd Iptables Rules Management](#systemd-iptables-rules-management)
  - [1. Environment](#1-environment)
  - [2. Introduction](#2-introduction)
  - [3. Getting started](#3-getting-started)

## 1. Environment

- Linux.
- Docker is installed.

## 2. Introduction

- There is no standard method to persistent iptables rules:
  - Add restore script in `rc.local`.
  - iptables-persistent.
  - systemd.
  - ...
- Therefore, I create this repository to standardize this: use *iptables-save* rules formating to change iptables rules, and control these with systemd service.
  - With systemd we can control the order of the startup process.
  - Cover common mistakes when work with iptables on Docker environment:
    - Create a iptables rules template that nobody can't go wrong.
    - Save, flush and restore all iptables including Docker installed rules: Every time the Docker container changes, you need to save the current iptables configuration, otherwise when restarting the iptables it will load the old rules, which will lead to confusing iptables rules.
    - Place the rules in wrong place: Rules in `INPUT`, `OUTPUT` chains not gonna work if there are exposed ports from Docker containers.
- This repository is highly inspired by [systemd-service-iptables](https://github.com/boTux-fr/systemd-service-iptables).
- The strategy used is **whitelist**: block all, allow some.
- Use `iptables-restore -n` turns off implicit global refresh and only performs our manual explicit refresh. But why?
  - As mentioned, our environment is Docker. Docker manipulates `iptables` rules to provide network isolation. Docker generates serveral rules, then adds to the `DOCKER` chain. If you save all current rules with `iptables-save` (including `DOCKER` chain rules as well) then flush + restore, it may not work as expected: container bridge ip address may be changed dynamically,...
- There are three chains: `INPUT`, `OUTPUT`, and `DOCKER-USER`. You may ask what the hell `DOCKER-USER` is. Docker installs two custom iptables chains named `DOCKER-USER` and `DOCKER`, and it ensures that incoming packets are always checked by these two chains first. All of Docker’s iptables rules are added to the `DOCKER` chain. Do not manipulate this chain manually. If you need to add rules which load before Docker’s rules, add them to the `DOCKER-USER` chain. These rules are applied before any rules Docker creates automatically.
- Each chain is consisted by the following parts. Check out the [template](etc/iptables/base.rules).
  - Allow packets on localhost and bridge interfaces.
  - Allow packets on established connections.
  - Your custom allow rules.
  - Write log before reject for troubleshooting.
  - Reject all other packets.

## 3. Getting started

- Ofc you need iptables and systemd installed.
- On the Linux, run as root:

```shell
git clone https://github.com/ntk148v/systemd-iptables
cd systemd-iptables
# Edit the rules in etc/iptables/base.rules as needed.
# and install the service
cp -Rv etc/. /etc/
```

- Make changes in `/etc/iptables/base.rules`.
  - Replace the placeholder `extinf` interface in the rulebook with your actual external interface (`eth0` for e.x).
  - Add your custom allow rules in the right place!
    - Allow inbound connections -> INPUT Custom Accept Block.
    - Allow outbound connections -> OUTPUT Custom Accept Block.
    - Allow inbound/outbound connections to your Docker containers using network bridge -> DOCKER-USER Custom Accept Block.
- After that, enable the serivces and we are done:

```shell
systemctl daemon-reload
systemctl enable iptables.service
systemctl enable iptables@base.service
systemctl start iptables@base.service
# Check status
systemctl status iptables@base.service
```

- If you make any changes in the future, make sure to restart/reload your service.

```shell
systemctl restart iptables@base.service
```
