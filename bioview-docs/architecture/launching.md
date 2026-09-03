# Launching and process lifetime

Everything starts through `bioview_client.launch`. It is a multi-call entry
point: `--role` selects what to run.

| Role | What it does |
| --- | --- |
| `monitor` (default) | Opens the Monitor GUI. |
| `configurator` | Opens the Configurator GUI. |
| `server` | Runs the headless server. Used only as the child process a GUI role spawns for itself. |

There is no separate "launcher" role and no way to open a client without a
server. The launcher's only job is to decide *which window* opens and to make
sure a server is there for it.

## Entry points

```bash
bioview                  # Monitor
bioview-monitor          # Monitor
bioview-configurator     # Configurator

python -m bioview_client.launch --role monitor
python -m bioview_client.launch --role configurator
```

`python -m bioview_client.monitor` and `python -m bioview_client.configurator`
are routed through the launcher too, so they behave identically.

The frozen bundle is one binary that re-execs itself with `--role server` to
spawn its own server.

## One shared server

A window first probes `127.0.0.1:8998` with the discovery handshake — not a bare
TCP connect, which any unrelated listener would satisfy. If a BioView server
answers, it is reused. If none does, the window spawns one as a detached child
process with `--local` and `--exit-when-idle`.

The server binds its listeners with `SO_EXCLUSIVEADDRUSE` (or the POSIX
equivalent), so if two windows race to spawn one, the loser's child exits on the
bind error rather than running alongside the winner and splitting clients
between them.

## Shutting the server down

The server is shared, so **only the last window standing may shut it down**. A
client count alone cannot decide that: a window that is still starting up has no
session on the server yet, and the probe that asks for the count can itself
fail.

Every GUI process therefore registers itself in a small JSON file in the BioView
cache directory (`windows.json`), keyed by pid and control port. On exit a
window:

1. Deregisters itself, then asks whether any other window is registered against
   the same port. If one is, nothing happens.
2. Otherwise asks the server for its live client count as a second opinion.
3. Terminates the server **only** if it spawned it, no other window is
   registered, and the server reports no clients.

If any of those answers is unavailable, nothing is killed. The server was
spawned with `--exit-when-idle`, so it retires by itself once it really has been
abandoned — a failed probe must never take a server out from under a window that
is still using it.

Registry entries left behind by a crashed window are pruned by checking whether
the pid is still alive (via `OpenProcess`/`GetExitCodeProcess` on Windows —
`os.kill(pid, 0)` there would *terminate* the process being asked about).

## Server logs

A GUI-spawned server is detached and windowless, so its stdout and stderr are
redirected to `server.log` in the BioView cache directory. That file is the only
record of what happened on the device side, and it is where to look when a
device will not initialize.
