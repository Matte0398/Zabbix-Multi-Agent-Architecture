# Zabbix Multi-Agent – Linux & Windows

Operational guide for configuring **two Zabbix Agent instances on the same machine**, in both **Linux** and **Windows** environments, while avoiding conflicts involving ports, configuration files, logs, PIDs, sockets, plugins, and services.

The guide covers the main scenarios with:

- **Zabbix Agent classic + Zabbix Agent classic**
- **Zabbix Agent classic + Zabbix Agent 2**
- **Zabbix Agent 2 + Zabbix Agent 2**

> For the standard installation of the first Agent, refer to the document  
> **“Zabbix – Setup Agent Unix-Linux-Windows”**.

---

## Table of Contents

- [Objective](#objective)
- [Prerequisites](#prerequisites)
- [Linux](#linux)
  - [Main parameters for the second Agent](#main-parameters-for-the-second-agent)
  - [Agent 2 plugins](#agent-2-plugins)
  - [systemd service](#systemd-service)
- [Windows](#windows)
  - [Example configuration](#example-configuration)
  - [Windows verification](#windows-verification)
- [Zabbix Frontend Configuration](#zabbix-frontend-configuration)
  - [Option 1 – Two separate hosts](#option-1--two-separate-hosts)
  - [Option 2 – Two Agent interfaces on the same host](#option-2--two-agent-interfaces-on-the-same-host)
- [Final Checklist](#final-checklist)
- [Quick Troubleshooting](#quick-troubleshooting)
  - [The custom port is not listening](#the-custom-port-is-not-listening)
  - [Agent 2 does not start](#agent-2-does-not-start)
  - [Active checks do not work](#active-checks-do-not-work)
  - [Passive checks do not work](#passive-checks-do-not-work)
- [Notes](#notes)

---

## Objective

Allow two Zabbix Agent instances to coexist on the same system while keeping the resources used by each instance separate.

In particular, the second Agent must use dedicated parameters to avoid overlapping with the first one.

Typical example:

| Parameter | First Agent | Second Agent |
|---|---|---|
| `ListenPort` | `10050` | `10056` |
| `Hostname` | Main host | Custom hostname |
| `LogFile` | Standard path | Custom path |
| `PidFile` | Standard path | Custom path |
| `PluginSocket` | Default | Dedicated socket |
| `ControlSocket` | Default | Dedicated socket |
| Service | Standard | Custom service |

Port `10051` normally remains the port used by the **Zabbix Server/Proxy** for active checks.

---

## Prerequisites

Before proceeding, make sure you have:

- administrative or `root` privileges;
- access to the **Zabbix Agent / Zabbix Agent 2** binaries or packages;
- the IP address or DNS name of the **Zabbix Server** or **Zabbix Proxy**;
- a free TCP port for the second Agent, for example `10056`;
- a different `Hostname` for the second instance, especially when using active checks;
- proper firewall rules for the new passive port.

---

# Linux

On Linux, it is recommended to keep the first Agent in the standard paths and install the second Agent in a dedicated location, for example:

```bash
/opt/zabbix_custom
```

Basic directory structure:

```bash
mkdir -p /opt/zabbix_custom/{etc,bin,sbin,var/log/zabbix,var/run/zabbix}
```

## Main parameters for the second Agent

Example:

```ini
Server=<ZABBIX_SERVER_OR_PROXY_IP>
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>
Hostname=<CUSTOM_HOSTNAME>
ListenPort=10056
```

For **Zabbix Agent 2**, the following parameters must also be separated:

```ini
Include=/opt/zabbix_custom/etc/zabbix_agent2.d/*.conf
Include=/opt/zabbix_custom/etc/zabbix_agent2.d/plugins.d/*.conf

PluginSocket=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2_custom_plugins.sock
ControlSocket=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2_custom_control.sock
```

### Agent 2 plugins

When loadable plugins are used, it is preferable to keep the second Agent 2 completely isolated inside the custom path.

Check for any hardcoded references to the standard path:

```bash
grep -R "/usr/sbin/zabbix-agent2-plugin" /opt/zabbix_custom/etc/zabbix_agent2.d/plugins.d/
```

If necessary, update the plugin `.conf` files so that they point to the custom path.

---

## systemd service

The second Agent must be managed through a dedicated service.

Example for Agent 2:

```ini
[Unit]
Description=Zabbix Agent 2 Custom
After=network.target

[Service]
Environment="CONFFILE=/opt/zabbix_custom/etc/zabbix_agent2.conf"
Type=simple
Restart=on-failure
PIDFile=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2.pid
ExecStart=/opt/zabbix_custom/sbin/zabbix_agent2 -c $CONFFILE
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
systemctl daemon-reload
systemctl enable zabbix_agent2_custom.service
systemctl start zabbix_agent2_custom.service
```

Verification:

```bash
systemctl status zabbix_agent2_custom.service
tail -f /opt/zabbix_custom/var/log/zabbix/zabbix_agent2.log
ss -lntp | grep 10056
```

Test from the Zabbix Server or Proxy:

```bash
zabbix_get -s <MONITORED_HOST_IP> -p 10056 -k agent.ping
```

Expected result:

```text
1
```

---

# Windows

On Windows, multiple Zabbix Agent instances can be run by installing them as **separate services**.

All commands must be executed from **PowerShell started as Administrator**.

Recommended custom path:

```text
C:\Program Files\Zabbix_custom
```

To install multiple instances, use:

```text
--multiple-agents
```

## Example configuration

```ini
Server=<ZABBIX_SERVER_OR_PROXY_IP>
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>
Hostname=<CUSTOM_HOSTNAME>
ListenPort=10056
LogFile=C:\Program Files\Zabbix_custom\zabbix_agent2.log
```

For Agent 2:

```ini
Include=C:\Program Files\Zabbix_custom\zabbix_agent2.d\*.conf
Include=C:\Program Files\Zabbix_custom\zabbix_agent2.d\plugins.d\*.conf

ControlSocket=\\.\pipe\zabbix_agent2_custom_control.sock
PluginSocket=\\.\pipe\zabbix_agent2_custom_plugin.sock
```

Install the second Agent 2 instance:

```powershell
Set-Location "C:\Program Files\Zabbix_custom"

.\zabbix_agent2.exe `
  --config "C:\Program Files\Zabbix_custom\zabbix_agent2.conf" `
  --install `
  --multiple-agents
```

Start the service:

```powershell
Get-Service -DisplayName "Zabbix Agent 2 [<CUSTOM_HOSTNAME>]" | Start-Service
```

List all Zabbix services:

```powershell
Get-Service -DisplayName "*Zabbix*"
```

---

## Windows verification

Check whether the custom port is listening:

```powershell
Get-NetTCPConnection -LocalPort 10056 -ErrorAction SilentlyContinue
```

Check the log:

```powershell
Get-Content "C:\Program Files\Zabbix_custom\zabbix_agent2.log" -Wait
```

If there are issues, start Agent 2 in foreground mode:

```powershell
& "C:\Program Files\Zabbix_custom\zabbix_agent2.exe" `
  --config "C:\Program Files\Zabbix_custom\zabbix_agent2.conf" `
  --foreground
```

---

# Zabbix Frontend Configuration

The two Agent instances can be represented in two main ways.

### Option 1 – Two separate hosts

Recommended when:

- both instances use active checks;
- the two Agents belong to different monitoring contexts;
- a complete logical separation is preferred.

### Option 2 – Two Agent interfaces on the same host

Useful when the same machine needs to be queried through two different passive ports.

Example:

```text
Agent 1 → TCP 10050
Agent 2 → TCP 10056
```

Always verify that:

- the `Hostname` configured in the Agent matches the one expected by Zabbix;
- templates and macros are associated with the correct instance;
- the port configured in the frontend matches the corresponding `ListenPort`.

---

# Final Checklist

Before considering the configuration complete, verify that:

- [ ] the second Agent uses a different `ListenPort`;
- [ ] the second Agent has a dedicated `Hostname` when active checks are used;
- [ ] `LogFile`, `PidFile`, and runtime directories are separated;
- [ ] `PluginSocket` and `ControlSocket` are different for Agent 2;
- [ ] on Linux, a custom `systemd` service is configured;
- [ ] on Windows, the second Agent is installed using `--multiple-agents`;
- [ ] the firewall allows the second Agent port;
- [ ] `zabbix_get` returns `1` for `agent.ping`;
- [ ] any Agent 2 plugins point only to the custom path;
- [ ] the Zabbix frontend configuration is consistent with `Hostname` and `ListenPort`.

---

# Quick Troubleshooting

### The custom port is not listening

Check:

- `ListenPort` configuration;
- possible conflicts with other processes;
- service status;
- Agent logs.

### Agent 2 does not start

Check in particular:

- `Include`;
- `PluginSocket`;
- `ControlSocket`;
- plugin paths;
- file and directory permissions.

### Active checks do not work

Check:

- `Hostname`;
- `ServerActive`;
- connectivity to the Zabbix Server/Proxy on port `10051`.

### Passive checks do not work

Check:

- `ListenPort`;
- firewall;
- `Server` parameter;
- connectivity to the Agent port from the Server/Proxy.

---

## Notes

This guide is intended for environments where **two independent Zabbix Agent instances must run on the same system**.

Before applying the procedure in production, it is recommended to test it in a staging or test environment, especially when using:

- Agent 2 plugins;
- custom configurations;
- UserParameters;
- firewall rules;
- different Zabbix versions.
