# Zabbix Multi-Agent Architecture

This guide explains how to install and configure **two Zabbix Agents on the same machine**, both on **Linux** and **Windows**.

The goal is to run two independent Zabbix Agent instances without conflicts on:

- listening port;
- configuration file;
- log file;
- PID file;
- internal sockets used by Zabbix Agent 2;
- operating system service;
- hostname used by Zabbix.

This setup is useful when the same server must be monitored by two different Zabbix environments, two different Zabbix Proxies, two different teams, or when you need to keep one standard Agent and one custom Agent with dedicated UserParameters, templates, or configuration settings.

---

## Table of contents

- [Scenario](#scenario)
- [Important concepts](#important-concepts)
- [Ports used](#ports-used)
- [Prerequisites](#prerequisites)
- [Linux configuration](#linux-configuration)
  - [Install the first Agent](#install-the-first-agent)
  - [Create a custom path for the second Agent](#create-a-custom-path-for-the-second-agent)
  - [Linux scenario 1: second classic Zabbix Agent](#linux-scenario-1-second-classic-zabbix-agent)
  - [Linux scenario 2: second Zabbix Agent 2](#linux-scenario-2-second-zabbix-agent-2)
  - [Linux scenario 3: first classic Agent and second Agent 2](#linux-scenario-3-first-classic-agent-and-second-agent-2)
  - [Create the systemd service](#create-the-systemd-service)
  - [Start and enable the second Agent](#start-and-enable-the-second-agent)
  - [Linux checks](#linux-checks)
- [Windows configuration](#windows-configuration)
  - [How it works on Windows](#how-it-works-on-windows)
  - [Windows scenario 1: second classic Zabbix Agent](#windows-scenario-1-second-classic-zabbix-agent)
  - [Windows scenario 2: second Zabbix Agent 2](#windows-scenario-2-second-zabbix-agent-2)
  - [Windows service management commands](#windows-service-management-commands)
  - [Windows checks](#windows-checks)
- [Zabbix Frontend configuration](#zabbix-frontend-configuration)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [References](#references)

---

## Scenario

A first Zabbix Agent is already installed on the machine using the standard installation method.

Example:

| Component | First Agent | Second Agent |
|---|---:|---:|
| Linux configuration path | `/etc/zabbix/` | `/opt/zabbix_custom/etc/` |
| Linux binary path | `/usr/sbin/` | `/opt/zabbix_custom/sbin/` |
| Passive listening port | `10050` | `10056` or another custom port |
| Zabbix Server/Proxy port | `10051` | `10051` |
| Linux service | `zabbix-agent` / `zabbix-agent2` | `zabbix_agentd_custom` / `zabbix_agent2_custom` |
| Log file | default path | custom path |
| PID file | default path | custom path |

> If your environment already defines a standard custom port, for example `10150`, use that port instead of `10056`.

---

## Important concepts

To run two Zabbix Agents on the same machine, you must avoid conflicts between the two instances.

The following elements should be different:

1. **passive listening port**, defined by `ListenPort`;
2. **configuration file**;
3. **log file**;
4. **PID file**;
5. **Zabbix Agent 2 sockets**, if Agent 2 is used;
6. **operating system service name**;
7. **Zabbix hostname**, especially for active checks and on Windows.

The most important parameter is `ListenPort`.

The first Agent usually listens on:

```ini
ListenPort=10050
```

The second Agent must listen on a different port, for example:

```ini
ListenPort=10056
```

or:

```ini
ListenPort=10150
```

---

## Ports used

### Passive checks

With passive checks, Zabbix Server or Zabbix Proxy connects to the Agent.

For this reason, two Agents on the same machine **cannot listen on the same port**.

Example:

| Agent | Port |
|---|---:|
| First Agent | `10050` |
| Second Agent | `10056` |

### Active checks

With active checks, the Agent connects to Zabbix Server or Zabbix Proxy.

The destination is configured with:

```ini
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>
```

By default, Zabbix Server or Proxy listens on:

```text
10051
```

This port usually does not need to be changed in the Agent configuration, unless your Zabbix Server or Proxy uses a custom listening port.

---

## Prerequisites

Before starting, make sure you have:

- administrative access to the Linux or Windows machine;
- the first Zabbix Agent already installed and working;
- IP address or DNS name of the Zabbix Server or Zabbix Proxy;
- passive port to assign to the second Agent;
- hostname to configure in Zabbix;
- local firewall opened on the second Agent port;
- Zabbix templates and items configured consistently with the selected interface and port.

---

# Linux configuration

## Install the first Agent

Install the first Agent using the standard procedure for your environment.

Examples:

```bash
dnf install zabbix-agent
```

or:

```bash
dnf install zabbix-agent2
```

On Debian/Ubuntu systems:

```bash
apt install zabbix-agent
```

or:

```bash
apt install zabbix-agent2
```

The first Agent keeps using the standard paths:

```text
/etc/zabbix/
/usr/sbin/
/var/log/zabbix/
/run/zabbix/
```

---

## Create a custom path for the second Agent

To avoid conflicts with the first Agent, create a dedicated directory structure for the second Agent:

```bash
mkdir -p /opt/zabbix_custom/{etc,bin,sbin,var/log/zabbix,var/run/zabbix}
```

Then set ownership:

```bash
chown -R zabbix:zabbix /opt/zabbix_custom
```

---

## Linux scenario 1: second classic Zabbix Agent

Use this procedure if the second Agent is the classic `zabbix_agentd`.

Copy the binary and configuration file:

```bash
cp /usr/sbin/zabbix_agentd /opt/zabbix_custom/sbin/
cp /etc/zabbix/zabbix_agentd.conf /opt/zabbix_custom/etc/
chown -R zabbix:zabbix /opt/zabbix_custom
```

Edit the custom configuration file:

```bash
vi /opt/zabbix_custom/etc/zabbix_agentd.conf
```

Recommended minimal configuration:

```ini
PidFile=/opt/zabbix_custom/var/run/zabbix/zabbix_agentd.pid
LogFile=/opt/zabbix_custom/var/log/zabbix/zabbix_agentd.log

Server=<ZABBIX_SERVER_OR_PROXY_IP>
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>

Hostname=<HOSTNAME_VISIBLE_IN_ZABBIX>

ListenPort=10056
```

Optional UserParameters:

```ini
UserParameter=check.fs.ro,if [ $(grep -E "ext.?|xfs|btrfs|vfat|gfs2|ocfs2" /proc/mounts | grep "ro," | grep -v "rw," | grep -v sentinelone | wc -l) -eq 0 ]; then echo 0; else echo 1; fi
UserParameter=check.args[*],echo "$1"
UserParameter=service.active[*],systemctl is-active "$1"
```

> If Zabbix Proxy and Agent are in the same network, configure the real Proxy IP address in both `Server` and `ServerActive`.

---

## Linux scenario 2: second Zabbix Agent 2

Use this procedure if the second Agent is `zabbix_agent2` and Agent 2 is already installed on the system.

Copy the binary, the main configuration file, and the include directory:

```bash
cp /usr/sbin/zabbix_agent2 /opt/zabbix_custom/sbin/
cp /etc/zabbix/zabbix_agent2.conf /opt/zabbix_custom/etc/
cp -r /etc/zabbix/zabbix_agent2.d /opt/zabbix_custom/etc/
chown -R zabbix:zabbix /opt/zabbix_custom
```

Edit the custom configuration file:

```bash
vi /opt/zabbix_custom/etc/zabbix_agent2.conf
```

Recommended minimal configuration:

```ini
PidFile=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2.pid
LogFile=/opt/zabbix_custom/var/log/zabbix/zabbix_agent2.log

Server=<ZABBIX_SERVER_OR_PROXY_IP>
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>

Hostname=<HOSTNAME_VISIBLE_IN_ZABBIX>

ListenPort=10056

Include=/opt/zabbix_custom/etc/zabbix_agent2.d/*.conf
Include=/opt/zabbix_custom/etc/zabbix_agent2.d/plugins.d/*.conf

PluginSocket=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2_plugins.sock
ControlSocket=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2_control.sock
```

Optional UserParameters:

```ini
UserParameter=check.fs.ro,if [ $(grep -E "ext.?|xfs|btrfs|vfat|gfs2|ocfs2" /proc/mounts | grep "ro," | grep -v "rw," | grep -v sentinelone | wc -l) -eq 0 ]; then echo 0; else echo 1; fi
UserParameter=check.args[*],echo "$1"
UserParameter=service.active[*],systemctl is-active "$1"
```

Important notes for Agent 2:

- if you use an external `.conf` file under `/opt/zabbix_custom/etc/zabbix_agent2.d/`, remember that `Include`, `PluginSocket`, and `ControlSocket` must still be configured in the main `zabbix_agent2.conf` file;
- default `Include`, `PluginSocket`, and `ControlSocket` values must be commented out or replaced;
- the second Agent 2 must not use the same internal sockets as the first Agent 2.

---

## Linux scenario 3: first classic Agent and second Agent 2

Use this procedure when the first installed Agent is the classic `zabbix_agentd`, but the second Agent must be `zabbix_agent2`.

In this case, download the required Zabbix Agent 2 packages manually and extract them without installing them over the existing Agent.

Example using RPM packages:

```bash
rpm2cpio zabbix-agent2-<zbx_version>-release1.<os-version>.rpm | cpio -idmv
```

Move the extracted files into the custom path:

```bash
mv etc/zabbix/zabbix_agent2.* /opt/zabbix_custom/etc/
mv usr/sbin/zabbix_agent2 /opt/zabbix_custom/sbin/
```

Remove only the temporary extracted directories:

```bash
rm -rf etc/ usr/ var/
```

> **Warning:** do not add `/` before these paths. Use `rm -rf etc/ usr/ var/`, not `rm -rf /etc /usr /var`.

If Agent 2 plugins are required, extract the related packages too.

Example for MongoDB plugin:

```bash
rpm2cpio zabbix-agent2-plugin-mongodb-<zbx_version>-release1.<os-version>.rpm | cpio -idmv
mv etc/zabbix/zabbix_agent2.d/plugins.d/mongodb.conf /opt/zabbix_custom/etc/zabbix_agent2.d/plugins.d/
mv usr/sbin/zabbix-agent2-plugin/ /opt/zabbix_custom/sbin/
rm -rf etc/ usr/ var/
```

Example for PostgreSQL plugin:

```bash
rpm2cpio zabbix-agent2-plugin-postgresql-<zbx_version>-release1.<os-version>.rpm | cpio -idmv
mv etc/zabbix/zabbix_agent2.d/plugins.d/postgresql.conf /opt/zabbix_custom/etc/zabbix_agent2.d/plugins.d/
mv usr/sbin/zabbix-agent2-plugin/zabbix-agent2-plugin-postgresql /opt/zabbix_custom/sbin/zabbix-agent2-plugin/
rm -rf etc/ usr/ var/
```

If required, create a symbolic link for the plugin directory:

```bash
ln -s /opt/zabbix_custom/sbin/zabbix-agent2-plugin /usr/sbin/zabbix-agent2-plugin
```

Then set ownership and remove downloaded packages:

```bash
chown -R zabbix:zabbix /opt/zabbix_custom
rm -f zabbix*.rpm
```

Edit the configuration file:

```bash
vi /opt/zabbix_custom/etc/zabbix_agent2.conf
```

Use the same configuration shown in [Linux scenario 2](#linux-scenario-2-second-zabbix-agent-2).

---

## Create the systemd service

Create a dedicated systemd service for the second Agent.

For Zabbix Agent 2:

```bash
vi /etc/systemd/system/zabbix_agent2_custom.service
```

Service file content:

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

For the classic Zabbix Agent, create for example:

```bash
vi /etc/systemd/system/zabbix_agentd_custom.service
```

Service file content:

```ini
[Unit]
Description=Zabbix Agent Custom
After=network.target

[Service]
Environment="CONFFILE=/opt/zabbix_custom/etc/zabbix_agentd.conf"
Type=forking
Restart=on-failure
PIDFile=/opt/zabbix_custom/var/run/zabbix/zabbix_agentd.pid
ExecStart=/opt/zabbix_custom/sbin/zabbix_agentd -c $CONFFILE
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```

---

## Start and enable the second Agent

Reload systemd:

```bash
systemctl daemon-reload
```

Enable the custom Agent at boot:

```bash
systemctl enable zabbix_agent2_custom.service
```

Start the service:

```bash
systemctl start zabbix_agent2_custom.service
```

For classic Agent, use:

```bash
systemctl enable zabbix_agentd_custom.service
systemctl start zabbix_agentd_custom.service
```

---

## Linux checks

Check the service status:

```bash
systemctl status zabbix_agent2_custom.service
```

Check the log file:

```bash
tail -f /opt/zabbix_custom/var/log/zabbix/zabbix_agent2.log
```

Check that the second Agent is listening on the custom port:

```bash
ss -lntp | grep 10056
```

or:

```bash
netstat -lntp | grep 10056
```

Test an item locally:

```bash
/opt/zabbix_custom/sbin/zabbix_agent2 -c /opt/zabbix_custom/etc/zabbix_agent2.conf -t agent.ping
```

If the systemd service fails to start, try running the Agent manually to see the error directly:

```bash
/opt/zabbix_custom/sbin/zabbix_agent2 -c /opt/zabbix_custom/etc/zabbix_agent2.conf
```

---

# Windows configuration

## How it works on Windows

On Windows, the same logic applies:

- the first Agent uses the default configuration and default service;
- the second Agent uses a separate directory;
- the second Agent uses a different configuration file;
- the second Agent listens on a different `ListenPort`;
- the second Agent is installed as a separate Windows service.

Zabbix supports running multiple Agent instances on Windows by installing each instance as a separate service and using different configuration files.

The most important parameters are:

```ini
Hostname=<HOSTNAME_VISIBLE_IN_ZABBIX>
ListenPort=10056
LogFile=C:\ZabbixAgentCustom\zabbix_agentd_custom.log
```

For active checks, the hostname should be unique and must match the host configured in the Zabbix Frontend.

---

## Windows scenario 1: second classic Zabbix Agent

### 1. Create the custom directory

Create a dedicated directory for the second Agent:

```powershell
New-Item -ItemType Directory -Path "C:\ZabbixAgentCustom" -Force
```

Copy the Agent executable and configuration file from the standard installation directory.

Example:

```powershell
Copy-Item "C:\Program Files\Zabbix Agent\zabbix_agentd.exe" "C:\ZabbixAgentCustom\"
Copy-Item "C:\Program Files\Zabbix Agent\zabbix_agentd.conf" "C:\ZabbixAgentCustom\zabbix_agentd_custom.conf"
```

If your Agent is installed in another path, adjust the source paths accordingly.

### 2. Edit the second Agent configuration file

Edit:

```text
C:\ZabbixAgentCustom\zabbix_agentd_custom.conf
```

Recommended configuration:

```ini
LogFile=C:\ZabbixAgentCustom\zabbix_agentd_custom.log

Server=<ZABBIX_SERVER_OR_PROXY_IP>
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>

Hostname=<HOSTNAME_VISIBLE_IN_ZABBIX>

ListenPort=10056
```

Optional examples of UserParameters on Windows:

```ini
UserParameter=check.args[*],echo $1
UserParameter=service.state[*],powershell -NoProfile -ExecutionPolicy Bypass -Command "(Get-Service -Name '$1').Status"
```

### 3. Install the second Agent as a Windows service

Open PowerShell or Command Prompt as Administrator.

Run:

```powershell
cd "C:\ZabbixAgentCustom"
.\zabbix_agentd.exe --config "C:\ZabbixAgentCustom\zabbix_agentd_custom.conf" --install --multiple-agents
```

The `--multiple-agents` option allows installing multiple Agent services on the same Windows host.

### 4. Start the second Agent service

The service name is generated using the Agent hostname value.

List the Zabbix services:

```powershell
Get-Service *zabbix*
```

Start the custom service:

```powershell
Start-Service "<ZABBIX_AGENT_CUSTOM_SERVICE_NAME>"
```

If you prefer using `sc.exe`:

```cmd
sc query type= service | findstr /I zabbix
sc start "<ZABBIX_AGENT_CUSTOM_SERVICE_NAME>"
```

---

## Windows scenario 2: second Zabbix Agent 2

### 1. Create the custom directory

```powershell
New-Item -ItemType Directory -Path "C:\ZabbixAgent2Custom" -Force
```

Copy the Agent 2 executable and configuration file:

```powershell
Copy-Item "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" "C:\ZabbixAgent2Custom\"
Copy-Item "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" "C:\ZabbixAgent2Custom\zabbix_agent2_custom.conf"
```

If present, copy the include directory too:

```powershell
Copy-Item "C:\Program Files\Zabbix Agent 2\zabbix_agent2.d" "C:\ZabbixAgent2Custom\zabbix_agent2.d" -Recurse
```

### 2. Edit the Agent 2 custom configuration file

Edit:

```text
C:\ZabbixAgent2Custom\zabbix_agent2_custom.conf
```

Recommended configuration:

```ini
LogFile=C:\ZabbixAgent2Custom\zabbix_agent2_custom.log

Server=<ZABBIX_SERVER_OR_PROXY_IP>
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>

Hostname=<HOSTNAME_VISIBLE_IN_ZABBIX>

ListenPort=10056

Include=C:\ZabbixAgent2Custom\zabbix_agent2.d\*.conf
Include=C:\ZabbixAgent2Custom\zabbix_agent2.d\plugins.d\*.conf

PluginSocket=C:\ZabbixAgent2Custom\zabbix_agent2_plugins.sock
ControlSocket=C:\ZabbixAgent2Custom\zabbix_agent2_control.sock
```

Important notes:

- use different `PluginSocket` and `ControlSocket` values from the first Agent 2 instance;
- use a different `ListenPort`;
- use a different `LogFile`;
- use a unique `Hostname`.

### 3. Install the second Agent 2 as a Windows service

Open PowerShell or Command Prompt as Administrator.

Run:

```powershell
cd "C:\ZabbixAgent2Custom"
.\zabbix_agent2.exe --config "C:\ZabbixAgent2Custom\zabbix_agent2_custom.conf" --install --multiple-agents
```

### 4. Start the second Agent 2 service

List the Zabbix services:

```powershell
Get-Service *zabbix*
```

Start the service:

```powershell
Start-Service "<ZABBIX_AGENT2_CUSTOM_SERVICE_NAME>"
```

---

## Windows service management commands

List Zabbix services:

```powershell
Get-Service *zabbix*
```

Stop a service:

```powershell
Stop-Service "<SERVICE_NAME>"
```

Start a service:

```powershell
Start-Service "<SERVICE_NAME>"
```

Restart a service:

```powershell
Restart-Service "<SERVICE_NAME>"
```

Uninstall a custom Agent service:

```powershell
.\zabbix_agentd.exe --config "C:\ZabbixAgentCustom\zabbix_agentd_custom.conf" --uninstall --multiple-agents
```

For Agent 2:

```powershell
.\zabbix_agent2.exe --config "C:\ZabbixAgent2Custom\zabbix_agent2_custom.conf" --uninstall --multiple-agents
```

---

## Windows checks

Check if the custom Agent is listening on the expected port:

```powershell
netstat -ano | findstr 10056
```

Test the Agent locally:

```powershell
cd "C:\ZabbixAgentCustom"
.\zabbix_agentd.exe --config "C:\ZabbixAgentCustom\zabbix_agentd_custom.conf" --test agent.ping
```

For Agent 2:

```powershell
cd "C:\ZabbixAgent2Custom"
.\zabbix_agent2.exe --config "C:\ZabbixAgent2Custom\zabbix_agent2_custom.conf" --test agent.ping
```

Check logs:

```powershell
Get-Content "C:\ZabbixAgentCustom\zabbix_agentd_custom.log" -Wait
```

or:

```powershell
Get-Content "C:\ZabbixAgent2Custom\zabbix_agent2_custom.log" -Wait
```

---

# Zabbix Frontend configuration

For the second Agent, configure the corresponding host in the Zabbix Frontend.

You can use one of two approaches.

## Option 1: same Zabbix host, second Agent interface

Use this approach if both Agents are monitored from the same Zabbix environment.

On the host configuration page:

1. open **Configuration → Hosts**;
2. select the host;
3. add a second Agent interface;
4. use the same IP address or DNS name;
5. configure the second Agent port, for example `10056`;
6. assign items/templates that must use the second Agent interface.

## Option 2: separate Zabbix host

Use this approach if the two Agents belong to different Zabbix environments, different Proxies, or different teams.

Create a separate host in Zabbix with:

```text
Host name: <HOSTNAME_VISIBLE_IN_ZABBIX>
Agent interface IP/DNS: <SERVER_IP_OR_DNS>
Agent interface port: 10056
```

For active checks, the `Hostname` configured in the Agent configuration file must match the host name configured in the Zabbix Frontend.

---

# Troubleshooting

## Port already in use

Error example:

```text
cannot bind to [[0.0.0.0]:10050]
```

Cause:

- both Agents are configured with the same `ListenPort`.

Fix:

- configure a different port for the second Agent, for example `10056` or `10150`.

---

## Wrong PID file or log file

Cause:

- both Agents are using the same `PidFile` or `LogFile`.

Fix:

- configure dedicated paths for the second Agent.

Example:

```ini
PidFile=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2.pid
LogFile=/opt/zabbix_custom/var/log/zabbix/zabbix_agent2.log
```

---

## Agent 2 socket conflict

Cause:

- two Agent 2 instances are using the same `PluginSocket` or `ControlSocket`.

Fix:

- configure dedicated socket paths for the second Agent 2 instance.

Example:

```ini
PluginSocket=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2_plugins.sock
ControlSocket=/opt/zabbix_custom/var/run/zabbix/zabbix_agent2_control.sock
```

---

## Active checks not working

Possible causes:

- `ServerActive` is wrong;
- the Agent cannot reach Zabbix Server or Proxy on port `10051`;
- `Hostname` does not match the host configured in Zabbix;
- the host is monitored by the wrong Proxy;
- firewall rules block outbound traffic.

Fix:

- verify connectivity to Zabbix Server or Proxy;
- verify the `Hostname` value;
- verify the Proxy assignment in the Zabbix Frontend;
- verify firewall rules.

---

## Passive checks not working

Possible causes:

- Zabbix Server or Proxy cannot reach the second Agent port;
- local firewall blocks the custom port;
- `Server` directive does not allow the Zabbix Server or Proxy IP;
- wrong Agent interface port in the Zabbix Frontend.

Fix:

- open the custom port on the host firewall;
- verify the `Server` directive;
- verify the Agent interface port in Zabbix;
- test the port from the Zabbix Server or Proxy.

Example:

```bash
zabbix_get -s <HOST_IP> -p 10056 -k agent.ping
```

---

# Security notes

- Allow only trusted Zabbix Server or Proxy IP addresses in the `Server` directive.
- Avoid using broad network ranges unless required.
- Keep the second Agent configuration under a controlled path.
- Restrict permissions on custom scripts and UserParameters.
- Do not expose the custom Agent port publicly.
- Document which Zabbix environment is using each Agent.

Example:

```ini
Server=10.10.10.10
ServerActive=10.10.10.10
```

Avoid insecure configurations such as:

```ini
Server=0.0.0.0/0
```

---

# References

- Zabbix official documentation: Windows agent installation and multiple Agent instances.
- Zabbix official documentation: Zabbix Agent and Zabbix Agent 2 configuration parameters.
- Internal Linux procedure used as the base for this guide.

---

# Final notes

The key point when running two Zabbix Agents on the same machine is isolation.

Each Agent instance must have its own:

- configuration file;
- listening port;
- log file;
- PID file;
- service name;
- Agent 2 sockets, when applicable;
- hostname, especially for active checks.

Once these elements are separated correctly, two Zabbix Agents can run on the same Linux or Windows host without conflicts.
