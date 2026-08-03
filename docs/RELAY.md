# Relay mode

Relay mode backs up a SQL Server without running the backup CPU on it. A thin
agent on the SQL Server host runs only the backup device. It streams the raw
backup over an encrypted link to a proxy machine. The proxy does the chunking,
deduplication, compression, encryption, and upload to PBS.

Use it when the database server cannot spare cores during the backup window, or to
back up many SQL Servers through one proxy.

Read [DEPLOYMENT.md](DEPLOYMENT.md) first for the base install and requirements.
This doc covers the relay setup on top of that.

## Why relay mode exists

The heavy part of a PBS backup is on the client: content-defined chunking, SHA-256
hashing, zstd compression, and AES encryption of every chunk. In local mode that
runs on the SQL Server host and competes with the live database. A half-cores
throttle helps but does not remove the load.

SQL Server's Virtual Device Interface is local-only. The backup device must run on
the same host as the instance. So the backup cannot simply run on another machine.

Relay mode splits the work at the one point that must stay local. The agent runs
the VDI device on the SQL host and does nothing else. Everything after the raw
byte stream runs on the proxy. The database server pays for reading the database
and encrypting the LAN link. It does not pay for chunking, compression, or dedup.

## Topology

```
  SQL Server host                         Proxy host                    PBS
  +-------------------+                    +-------------------+         +-------+
  | pbsgui agent      |  raw backup bytes  | pbsgui proxy      |  dedup  |       |
  | (VDI device only) | =================> | chunk/zstd/AES/up  | ======> |  PBS  |
  |                   |   TLS, TCP 8317    | + runs the jobs   |  HTTPS  |       |
  +-------------------+   (agent dials     +-------------------+  8007   +-------+
        ^   |             out to proxy)          |
        |   | TDS 1433 (BACKUP/RESTORE, gating)  |
        +---+------------------------------------+
```

- The **agent** runs the VDI device on the SQL host. It holds no SQL or PBS
  credentials.
- The **proxy** runs the backup jobs. It holds the PBS token and the SQL
  connection login. It issues BACKUP and RESTORE over TDS to the SQL instance, and
  it does all the CPU-heavy backup work.
- The agent dials **out** to the proxy on TCP 8317. The SQL host needs no inbound
  port.

## Network and firewall

| From | To | Port | Purpose |
| --- | --- | --- | --- |
| SQL host (agent) | Proxy | TCP 8317 | Relay control and data. Agent dials out. |
| Proxy | SQL instance | TCP 1433 | TDS: BACKUP/RESTORE statements, replica gating, metadata. |
| Proxy | PBS | TCP 8007 | Backup upload and restore download. |

Open inbound 8317 on the proxy from the SQL hosts. Example, on the proxy:

```
New-NetFirewallRule -DisplayName "pbsgui relay" -Direction Inbound `
  -Protocol TCP -LocalPort 8317 -Action Allow
```

The relay link is TLS. The agent pins the proxy's certificate by SHA-256
fingerprint, the same way pbsgui pins PBS. Each agent authenticates with a token.

The bytes on the wire are the native SQL backup stream, uncompressed. That is the
same volume any remote backup sends. Do not turn on SQL `WITH COMPRESSION`. pbsgui
does not, because it would defeat the proxy's deduplication.

## Credentials and trust

- The **proxy** holds the PBS API token and the SQL connection login. The SQL
  login runs BACKUP and RESTORE over TDS, so it must be in the `sysadmin` server
  role. With integrated auth the proxy connects as its own machine account
  (`DOMAIN\PROXY$`), which then needs a login and `sysadmin` on the target
  instance. With a SQL login, store it on the connection.
- The **agent** holds only the proxy address, the proxy's pinned fingerprint, and
  its own token. It has no SQL login and no PBS token. A compromised SQL host
  learns nothing about PBS. The agent runs the VDI device as LocalSystem, the same
  identity a local install uses.

## Deploy

### 1. Install pbsgui on both machines

Install pbsgui on the proxy host and on each SQL Server host. Same installer, from
[DEPLOYMENT.md](DEPLOYMENT.md) step 3. The proxy is a normal install. Each SQL host
is a normal install that will run in agent mode.

The proxy needs a route to each SQL instance (TDS 1433) and to PBS (8007).

### 2. Add an agent on the proxy

On the proxy, for each SQL host, run:

```
pbsgui-engine relay add-agent stanley
```

Use a short name for each host (letters, digits, dashes). It prints the token, the
proxy fingerprint, and the exact `relay join` command to run on the SQL host. It
writes `relay-server.json` and stores the token in the credential store.

Copy the printed `relay join` command. It contains the proxy address, fingerprint,
agent name, and token. The command prints `<THIS-HOST>` in place of the proxy
address. Replace it with the proxy's hostname or IP that the SQL host can reach.

### 3. Restart the proxy service

The relay listener starts when the engine starts, so restart the service on the
proxy so it picks up the new agent:

```
Restart-Service pbsgui-engine
```

Do this after every `add-agent`.

### 4. Join from the SQL host

On the SQL Server host, run the command that `add-agent` printed. It looks like:

```
pbsgui-engine relay join --proxy proxyhost:8317 `
  --fingerprint AA:BB:...:FF --name stanley --token <token>
```

It writes `relay.json` and stores the token. Then restart the service so the agent
dials out:

```
Restart-Service pbsgui-engine
```

### 5. Confirm the agent is connected

On the proxy, open the app, go to the **SQL Servers** tab. The agent shows as
connected with the host name and build. Or run on either machine:

```
pbsgui-engine relay show
```

### 6. Route a connection through the agent

On the proxy app, **SQL Servers** tab:

1. Save a SQL connection for the remote instance. The server is the SQL host name,
   or `HOST\INSTANCE` for a named instance. Set auth: integrated (the proxy machine
   account) or a SQL login.
2. On the saved connection, click **Relay** and enter the agent name (for example
   `stanley`). The connection now shows a `via relay` badge.

### 7. Create and run a job

Create a job with that SQL connection as the source and a PBS server as the
destination, as in [DEPLOYMENT.md](DEPLOYMENT.md) step 4. Run it.

Watch CPU during the run. The chunk, compression, and encryption load is on the
proxy. The SQL host shows the database read and the TLS link only.

Restore works the same way. Point-in-time and full restore both stream from the
proxy back into the SQL host through the agent.

## Fleet notes

- One proxy serves many agents. Add one agent per SQL host.
- Each agent has its own token. Re-running `add-agent` for a name rotates its
  token; re-run `relay join` on that host with the new token.
- Size the proxy for the backup CPU of all hosts that back up in the same window.
- The proxy can also run local backups of its own for non-SQL sources.

## Config and state

On each machine, under `C:\ProgramData\pbsgui`:

| File | Machine | Contents |
| --- | --- | --- |
| `relay-server.json` | Proxy | Listener bind, port, and the configured agent names. |
| `relay\relay-cert.der`, `relay\relay-key.der` | Proxy | The relay TLS certificate and key. Generated on first use. |
| `relay.json` | Agent | Proxy address, pinned fingerprint, and this agent's name. |

Tokens are in the Windows Credential Manager, not in these files. On the proxy each
agent's token is under `relayagent:<name>`. On the agent the token is under
`relay:proxy`.

To change the listener port or bind address, edit `relay-server.json` on the proxy
and restart the service. The default is `0.0.0.0:8317`.

## Troubleshooting

- **Agent shows offline / "relay agent X is not connected".** The agent service is
  not running, cannot reach the proxy on 8317, or the fingerprint or token does not
  match. Check the firewall, then `relay show` on both, then restart both services.
- **Fingerprint mismatch on join.** The proxy regenerated its certificate (the
  `relay\` files were deleted). Re-run `add-agent`, then `relay join` with the new
  fingerprint.
- **Backup fails with a sysadmin error.** The proxy's SQL connection login is not in
  `sysadmin` on the target instance. Fix the login, not the agent.
- **A restart on the proxy did not add the agent.** `add-agent` writes the config,
  but the running listener reads it at startup. Restart the service.
- **Renamed or cross-instance restore of a very old snapshot fails over the relay.**
  Snapshots taken before pbsgui stored the database file list cannot be relocated
  over the relay. Restore under the original name, or run the restore from pbsgui
  installed on the SQL host.
