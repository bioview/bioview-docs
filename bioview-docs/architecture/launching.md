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
process with `--local` and `--exit-when-idle`. The window is told which ports
the launcher parsed; a window left on its compiled-in defaults would look for
its server on 8998 while the launcher had just started one somewhere else. The
*data* port is not passed around at all: the server advertises it in its
discovery reply, which also gets it right for a server found on the LAN.

The server binds its listeners with `SO_EXCLUSIVEADDRUSE` (or the POSIX
equivalent), so if two windows race to spawn one, the loser's child exits on the
bind error rather than running alongside the winner and splitting clients
between them. The loser then reuses the winner's server. If no server answers
at all, the spawn is retried once; if that still produces nothing, the launcher
says so in a dialog pointing at `server.log` rather than opening a window that
could never connect.

## Keeping the server alive

A window is a client of its server long before it manages to connect: the
Monitor builds its client only once its configuration dialog has been answered,
and nobody is obliged to answer it. The server's `--exit-when-idle` is
therefore not a deadline for connecting by.

Every window instead holds a **claim** on its server. Each window process has an
opaque token, and a heartbeat -- an ordinary daemon thread, outside Qt so a
blocked event loop cannot stall it -- sends that token to the control port every
few seconds, along with the interval it intends to call at. The server keeps
each claim alive for three of those intervals, so a window that dies without
warning is forgotten without anyone having to notice that it died. A probe
carrying no token (a subnet scan, an older client) is answered but claims
nothing.

The server is therefore "in use" when it has connected clients *or* live window
claims, and only the idle countdown that runs when it has neither can retire it.

This is also what makes reuse safe. Without it, a window could find a server a
fraction of a second from its idle exit, skip the spawn on that basis, and then
watch it vanish. The probe that finds a server also claims it.

## Shutting the server down

The server is shared, so **only the last window standing may shut it down**. A
client count alone cannot decide that: a window that is still starting up has no
session on the server yet.

A closing window withdraws its claim and asks what is left in the same exchange:
the server drops that token and replies with the remaining window and client
counts. The window terminates the server **only** if it spawned it, no other
window still claims it, and no clients are connected.

If the answer is unavailable, nothing is killed. The server was spawned with
`--exit-when-idle`, so it retires by itself once it really has been abandoned —
a failed probe must never take a server out from under a window that is still
using it.

> This was a file of pids for a while (`windows.json`), with each window
> registering itself and liveness inferred from the OS. The claims replaced it:
> the server is what needs to know who wants it, it is the only party that
> cannot be wrong about that, and there is no longer a lock file to contend on,
> a registry to prune, or a reused pid to be fooled by.

## When the server dies anyway

A crash, a kill, a backend taking the process down: the window notices (its
handler watches the control socket for a closed peer) and reports a *lost*
server, which is deliberately distinct from the user's own Disconnect. Only a
loss re-arms the retry timers -- both stop themselves on the first successful
connect, and the Monitor pointedly does not relatch onto localhost, so that
someone who disconnected on purpose is not dragged straight back.

For the local server the window also offers to start a replacement, in a modal
that says what happened. A new server is an *empty* server -- whatever was
initialized is gone and any recording has already stopped -- so this is offered
rather than done silently, and the window drops what it believed about the
devices either way. If the restart fails, the reason is shown in the dialog
itself, and quitting joins trying again and carrying on regardless as options.

## Server logs

A GUI-spawned server is detached and windowless, so its stdout and stderr are
redirected to `server.log` in the BioView cache directory. That file is the only
record of what happened on the device side, and it is where to look when a
device will not initialize.
