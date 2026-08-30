# KiroCrew — Remote Host SSH Connection Issue (RESOLVED)

**Date:** 2026-08-29
**Host:** dev-dsk-mahicg-2023-1b-7a0519f9.us-east-1.amazon.com
**Dashboard port:** 5477 (non-default; default is 5476)
**OS (client):** macOS

---

## Symptom

Connecting to the remote dev desktop from the KiroCrew desktop app failed with:

```
Connect to devdesktop failed: instances manager not running
```

## Root cause

Two separate problems were in play:

1. **Missing local SSH certificate.** On the Mac, `mwinit -o` (OTP) saved a
   session cookie but returned `FAILED to get certificate. You are required by
   AWS Security to use third-factor with OTP for access.` Without the local SSH
   cert the connect could not complete. A successful `mwinit` run *on the remote
   dev desktop* does nothing for the Mac-initiated connection — auth must happen
   on the Mac that initiates the connection.

2. **Gateway + connection wiring.** The gateway was being run by hand
   (`kirocrew gateway`, headless) so it did not survive disconnects, and the
   app's remote-host SSH config was not set up for a persistent connection.

## Fix (what actually resolved it)

### 1. Fix local Midway auth (Mac) — get the SSH cert

Use FIDO2 / on-token PIN, NOT `-o`:

```
mwinit -f
```

Confirms with:
```
Successfully authenticated using WebAuthN, session cookie saved in /Users/mahicg/.midway/cookie
Successfully signed SSH public key "/Users/mahicg/.ssh/id_ecdsa.pub".
The SSH certificate was saved in "/Users/mahicg/.ssh/id_ecdsa-cert.pub".
```

> NOTE: `mwinit -o` (OTP) is no longer sufficient for cert issuance — it saves a
> cookie but fails to get the certificate. Always use `mwinit -f` (or plain
> `mwinit`, which auto-selects WebAuthn).

### 2. Make the gateway permanent (on the REMOTE dev desktop)

Stop the manually-started headless gateway (Ctrl+C), then install it as a
managed service so it survives SSH disconnects, reboots, and crashes:

```
kirocrew service install
kirocrew service status
```

After this, never run `kirocrew gateway` by hand again.

### 3. Add the remote host to the local Mac SSH config

Added to `/Users/mahicg/.ssh/config`:

```
Host dev-dsk-mahicg-2023
   HostName dev-dsk-mahicg-2023-1b-7a0519f9.us-east-1.amazon.com
   ProxyCommand /usr/local/bin/wssh proxy %h
   User mahicg
   IdentityFile /Users/mahicg/.ssh/id_ecdsa
   ServerAliveInterval 15
   ServerAliveCountMax 44
```

Key points in this config:
- `ProxyCommand /usr/local/bin/wssh proxy %h` — routes the connection through
  the Amazon `wssh` proxy (the supported path for reaching the dev desktop).
- `IdentityFile /Users/mahicg/.ssh/id_ecdsa` — the key whose cert `mwinit -f`
  signs (`id_ecdsa-cert.pub` is picked up automatically alongside it).
- `ServerAliveInterval 15` / `ServerAliveCountMax 44` — keepalives so the
  connection is not dropped by idle timeouts (~11 min of tolerated silence).

### 4. Update the remote host config in the KiroCrew app

Pointed the app's remote-host setting at this SSH host entry, then connected
successfully.

---

## Recurring gotcha (Amazon dev desktops)

The Midway SSH cert from `mwinit -f` **expires** (typically ~18–20 hours). When
the connection stops working after being fine, re-run:

```
mwinit -f
```

on the Mac. No SSH/tunnel config can avoid this — it is a Midway cert lifetime
limitation, not a KiroCrew one.

## Quick re-connect checklist

1. On Mac: `mwinit -f` (if cert may have expired)
2. On remote (only if gateway not already a service): `kirocrew service status`
   — should show running; if not, `kirocrew service install`
3. Connect in the app to host `dev-dsk-mahicg-2023`

## Notes / non-issues seen along the way

- Gateway warning `Unknown key GITHUB_MCP_PAT in .env is not a recognised
  credential` is **non-fatal** — the PAT still propagates to child processes but
  is not agent-isolated.
