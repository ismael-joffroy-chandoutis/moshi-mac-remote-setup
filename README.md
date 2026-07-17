**English** · [Français](README.fr.md)

# Drive your Mac from your iPhone with Moshi: the complete setup (and the gotchas nobody documents)

A field guide to running [Moshi](https://getmoshi.app) (the iOS terminal + AI-agent companion) against a Mac, end to end:

- an **Agent inbox** on your iPhone and Apple Watch (your Claude Code / Codex sessions, notifications, turn completions)
- **friendly session names** in Moshi that match your local multiplexer (cmux), instead of `ttys000`
- a **real terminal from anywhere** (5G, any Wi-Fi) over Tailscale, so you can type in a live session when you are away from home

Most of this is documented somewhere. The three things that cost me hours, and that you will not find spelled out anywhere, are in the **Gotchas** sections. If you only read one thing, read those.

> Placeholders: replace `100.x.y.z` with your Mac's Tailscale IP, `youruser` with your macOS short username, and `you@example.com` with your Tailscale login.

**Companion guide:** for the bigger picture — how Moshi compares to native **Remote Control** (Claude app / ChatGPT app for Codex) and per-tool mobile apps, when to use which, the persistence truth, and the multi-machine / parity patterns — see **[Driving coding agents from your phone](docs/driving-coding-agents-remote.md)**.

---

## What you get

| Capability | Transport | Works away from home? |
|---|---|---|
| Agent inbox, notifications, Apple Watch | moshi-hook daemon -> Moshi cloud | Yes (cloud, no VPN needed) |
| Friendly session names | local script + launchd | n/a (local) |
| Type in a live session ("Open in Terminal") | SSH/Mosh over Tailscale | Yes, once Tailscale is set up right |

The first row is the important insight: **the agent inbox does not need any VPN or open port.** The daemon makes an outbound WebSocket to Moshi's cloud, so notifications and session triage work on cellular out of the box. Only the *terminal* needs network reachability to the Mac.

---

## Prerequisites

- macOS with Homebrew
- [Moshi](https://apps.apple.com/app/id6757859949) on iPhone (and the watchOS app, optional)
- `tmux` (or Zellij/Herdr) for terminal sessions
- Optional: [cmux](https://github.com/) if you want the friendly-names bridge

---

## Part 1: Agent hooks (inbox, notifications, Apple Watch)

This is the `moshi-hook` daemon. It installs hooks into your agent CLIs and forwards lifecycle events (session start/end, turn complete, permission prompts, questions) to the Moshi app.

```bash
brew tap rjyo/moshi
brew install moshi-hook

# Settings -> Hooks in the Moshi app shows a pairing token:
moshi-hook pair --token <token-from-the-app>
moshi-hook install            # writes hook config into ~/.claude/settings.json, ~/.codex/hooks.json, etc.
brew services start moshi-hook
moshi-hook status             # should say: status: paired
```

Verify the daemon actually sees your sessions:

```bash
moshi-hook cwd-list           # lists the agent working dirs the daemon has recorded
```

If your projects show up here, the host side is done. The sessions appear under the **Agents** tab in the app (not the Hooks/setup screen).

### Gotcha #1: stale hooks after an upgrade

If your inbox is empty even though the daemon is `paired` and running, your `settings.json` hooks may be **stale** (written by an older version). The daemon says so in its log:

```
WARN "agent hooks missing or stale; rerun install" command="moshi-hook install --target claude"
```

Fix:

```bash
moshi-hook install            # rewrites the current hook set, removes retired events
```

This was the single thing that made an empty inbox come alive for me. The daemon log lives at `~/Library/Application Support/Moshi/hook.log`, but note it only logs the usage poller, **not** individual hook calls, so an empty log does not mean hooks are failing. The real signal is the app, or `moshi-hook cwd-list`.

### Note: `--dangerously-skip-permissions` removes Allow/Deny cards

If you run your agent with skip-permissions, the agent never asks for permission, so the **PermissionRequest** event never fires and you will never get Allow/Deny cards in the app. You still get turn-completion notifications and question cards. That is a design consequence of your own flag, not a bug.

---

## Part 2: Friendly session names (cmux -> Moshi)

If you wrap your sessions in [cmux](https://github.com/), Moshi shows them as `ttys000`, `ttys001`, ... because **cmux never propagates its friendly workspace titles to the underlying tmux session names.**

The bridge: read cmux's titles from its RPC, map each `tmux new-session -s ttysNNN` process to its `CMUX_WORKSPACE_ID`, and `tmux rename-session` to the friendly title. A launchd job re-applies it every couple of minutes so new workspaces get named too.

- `scripts/moshi-name-sessions.sh` does the mapping and renaming (idempotent: it only touches sessions still named `ttysNNN`).
- `scripts/com.example.moshi-name-sessions.plist` runs it every 120s.

```bash
cp scripts/moshi-name-sessions.sh ~/.claude/scripts/
chmod +x ~/.claude/scripts/moshi-name-sessions.sh

# edit the plist path to your user, then:
cp scripts/com.example.moshi-name-sessions.plist ~/Library/LaunchAgents/com.you.moshi-name-sessions.plist
launchctl load ~/Library/LaunchAgents/com.you.moshi-name-sessions.plist
```

It is non-destructive: renaming a tmux session is a label change, it does not touch the running process. While cmux is attached, the rename sticks (cmux tracks panes, not session names). To revert, unload the launchd job.

---

## Part 3: A terminal from anywhere (Tailscale)

For "Open in Terminal" to work when you are on 5G, Moshi connects over SSH/Mosh, and the Mac must be reachable. The right tool is **Tailscale**, not Cloudflare (Cloudflare Tunnel for SSH needs a client-side proxy that an iOS terminal app does not have).

Mac side:

```bash
sudo systemsetup -setremotelogin on          # or: moshi-hook host enable-ssh
brew install tailscale && sudo tailscale up   # (or the GUI app)
tailscale ip -4                                # your Mac's 100.x.y.z
```

In Moshi, create a saved connection (or use `moshi-hook host setup` for Easy Pair):

- **Host: the Tailscale IP `100.x.y.z`** (NOT the `.local` Bonjour name, which only resolves on the home LAN)
- User: `youruser`, Port: 22, type: Auto

### Gotcha #2 (the big one): a strict tailnet ACL silently drops the phone

Symptom: the Tailscale app says **Connected**, `tailscale ping` from the Mac gets a pong, but the phone cannot reach the Mac at all. SSH times out (`os error 60`), and so does plain HTTP in Safari. It looks like a routing or iOS bug. It is not.

If your tailnet uses [ACLs](https://tailscale.com/kb/1018/acls) (common once you have Tailnet Lock or any custom policy), the Mac **drops every packet from a device that no rule permits.** `tailscale ping` works because the low-level disco ping is not subject to the data-packet filter, which is exactly why this is so confusing.

Diagnose it from the Mac. This prints the packet filter the Mac actually received, i.e. which source IPs are allowed to reach it:

```bash
tailscale debug netmap | python3 -c "import json,sys; d=json.load(sys.stdin); [print(r.get('Srcs')) for r in d.get('PacketFilter',[])]"
```

If your phone's `100.x` IP is **not** in any `Srcs` list, that is your answer.

Fix: make the phone match a rule. A clean tag-based ACL looks like this, where mobile devices may reach trusted devices on the ports you care about:

```json
{
  "tagOwners": { "tag:trusted": ["autogroup:admin"], "tag:mobile": ["autogroup:admin"] },
  "grants": [
    { "src": ["tag:trusted"], "dst": ["*"], "ip": ["*"] },
    { "src": ["tag:mobile"], "dst": ["tag:trusted"],
      "ip": ["tcp:22", "tcp:80", "tcp:443", "tcp:5900", "tcp:8080", "udp:60000-61000"] }
  ]
}
```

Then **tag the phone** so it actually falls under `tag:mobile`: admin console -> Machines -> your phone -> Edit ACL tags -> add `tag:mobile`. The moment you save, the phone's IP shows up in the `netmap` filter and SSH/Moshi connects.

Include `udp:60000-61000` so Moshi can use **Mosh**, which survives the constant address changes of a cellular connection. SSH alone (TCP) will work but can stall when your 5G address rebinds.

### Gotcha #3: reinstalling Tailscale on iOS locks the device out (Tailnet Lock)

If your tailnet has [Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock) enabled and you delete + reinstall the Tailscale app, the phone generates a **new node key** that is now unsigned. You get "Admin signing required", and again: Connected, but no traffic. Do not chase this as an iOS bug.

Sign it from a trusted signing node (e.g. your Mac, if it is one):

```bash
tailscale lock status                                  # lists locked-out nodes + their nodekey, and whether THIS node can sign
tailscale lock sign nodekey:<the-locked-out-key>
```

Lesson: do not reinstall the Tailscale iOS app unless you have to. If you do, re-sign from the Mac.

### Keep the Mac awake

If the Mac sleeps at home, it is unreachable no matter how perfect the rest is.

```bash
sudo pmset -c sleep 0 displaysleep 10     # never sleep on AC power; screen can still dim
```

Keep it plugged in with the lid open. Lid-closed clamshell without an external display still sleeps unless you set `pmset -c disablesleep 1`, which also disables sleep on battery and can overheat in a bag, so use it deliberately.

---

## Quick diagnostic cheat-sheet

```bash
# is the daemon healthy and seeing sessions?
moshi-hook status
moshi-hook cwd-list

# is SSH actually up? (lsof without root cannot see the root-owned sshd socket: false negative)
nc -vz 127.0.0.1 22

# is the Mac reachable over Tailscale, and how (direct vs DERP relay)?
tailscale ping <phone-name>
tailscale netcheck

# is the macOS firewall blocking inbound?
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# the decisive one: does the ACL filter allow my phone to reach this Mac?
tailscale debug netmap | python3 -c "import json,sys; d=json.load(sys.stdin); [print(r.get('Srcs')) for r in d.get('PacketFilter',[])]"
```

---

## License

- Documentation (text): [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)
- Code (scripts): [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)

Not affiliated with Moshi or Tailscale. Tool names belong to their owners.

By [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com).
