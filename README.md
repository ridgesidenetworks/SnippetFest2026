# Snippetfest 2026 Submission — `mallen_SF2026`

**Anti-Evasion / Covert-Channel Control Snippet**
Label: `Snippetfest-2026` · Author: Mark Allen · Fully self-contained in snippet scope

---

## Summary

Every allow-list ruleset governs *which apps* users can reach. This snippet
governs the *evasion methods* attackers use **inside** those allowed paths — the
covert channels that a normal ruleset leaves wide open. It closes DNS tunneling,
encrypted DNS (DoH/DoT), non-standard-port tunneling, anonymizers, and the
"unknown" traffic App-ID cannot inspect — in one drop-in, self-contained snippet.

## Why it matters

Most policy work is about *reach*: can this user get to that app. But a modern
attacker who already has a foothold doesn't need new reach — they need a way to
move command-and-control and exfiltrated data **past inspection** using channels
the allow-list never contemplated:

- DNS tunneling and **encrypted DNS (DoH/DoT)** that routes name resolution
  around the corporate resolver and DNS Security entirely.
- **Non-standard-port tunneling** — SSH on 443, proxies, SOCKS — that defeats
  port-based assumptions.
- **Anonymizers and circumvention tools** (Tor, Psiphon, Ultrasurf) built
  specifically to evade network controls.
- **"Unknown" traffic** that App-ID cannot classify — the classic signature of
  bespoke C2 and custom tunneling.

These are exactly the gaps a "block-what-you-don't-allow" posture misses, because
the evasion rides inside sessions the allow rules permit. This snippet is the
counterpart control: hardening and detection, not another allow rule.

## What's in it (every object carried by the snippet — nothing device-level)

| Object | Type | Purpose |
|--------|------|---------|
| `anti-evasion-blocked-apps` | Application group (18 App-IDs) | Anonymizers, encrypted DNS, proxies/tunnels, unsanctioned VPN protocols |
| `Risky Proxy Apps` | Application **filter** | Dynamically matches *every* proxy-category app with risk level 3 or higher — future-proof coverage beyond the named list |
| `unknown-traffic-apps` | Application group (3 App-IDs) | `unknown-tcp`, `unknown-udp`, `unknown-p2p` |
| `covert-channel-exceptions` | Dynamic address group | Pressure-release valve, seeded empty, customer-populated by tag |
| `anti-evasion-logfwd` | Log-forwarding profile | Forwards deny events to SIEM/syslog for triage |
| **5 security rules (ordered)** | Security policy | The enforcement layer — see below |

**App-ID membership** (validated live against the tenant's predefined applications):

- *Anonymizers / circumvention:* `tor`, `psiphon`, `ultrasurf`, `hola-unblocker`, `tor2web`
- *Encrypted DNS (evades DNS Security):* `dns-over-https`, `dns-over-tls`, `dnscrypt`, `tcp-over-dns`
- *Proxy / tunneling:* `http-proxy`, `socks`, `ssh-tunnel`
- *Unsanctioned VPN/tunnel protocols:* `open-vpn`, `l2tp`, `ike`, `ipsec-esp`, `ipsec-esp-udp`, `dtls`

### The five rules (order is load-bearing)

1. **`allow-covert-channel-exceptions`** — Explicit allow for sanctioned
   destinations that would otherwise trip the denies (internal custom apps seen
   as unknown, an approved DoH resolver, sanctioned VPN endpoints). Placed first,
   `application-default` service, logged. Keeps the broad denies from catching
   legitimate traffic.
2. **`block-circumvention-and-tunneling-apps`** — Denies the anonymizer / DNS /
   proxy / tunnel group **plus the `Risky Proxy Apps` application filter**, which
   dynamically matches every proxy-category app with a risk level of 3 or higher.
   The filter is the key move here: it catches risky proxy apps we didn't name
   explicitly — and any new ones Palo Alto adds to App-ID later — so coverage
   stays current without editing the snippet. Service is **any** (not
   application-default) on purpose: evasion tools deliberately use non-standard
   ports, so we block them everywhere. Logs to the forwarding profile.
3. **`block-public-encrypted-dns`** — Denies DoH/DoT/dnscrypt explicitly, forcing
   DNS back through the inspected resolver / Palo Alto DNS Security. Kept as its
   own rule (not just folded into rule 2) so it has a clean, separately labeled
   log stream. Aligns with PAN best practice to block both DoH and DoT.
4. **`block-unknown-traffic`** — Denies traffic App-ID cannot classify. Depends
   on the exceptions rule for legitimate internal apps; recommended deployment is
   log-only first, then enforce.
5. **`block-quic`** — Denies QUIC (`quic-base`, `gquic`) so browsers fall back to
   TCP-based TLS the firewall can actually see and decrypt. A deliberate
   visibility-vs-performance tradeoff, and tunable.


## Engineering notes (how it was built)

The entire snippet was built programmatically by a **coding agent driving the SCM
API** — no clicking through the UI. The agent worked against the platform through
the community Python SDK (`pan-scm-sdk`, `ScmClient`) over the REST API, and
authored an idempotent, re-runnable build tool along the way: dry-run by default
(validates and prints every payload with no writes), apply on explicit
confirmation, and env-only credentials.

Because the build ran through the API, it could do things a point-and-click
workflow can't easily do — most notably, it **verified all 23 App-ID references
against the tenant's ~10,800 predefined applications live** before any write, so
an invalid or renamed name aborts the build instead of silently half-applying
(this is how `openvpn`, `ipsec-base`, and `quic` were caught and corrected to
their real App-IDs). Every object is created strictly in snippet scope with **no
folder or device association** and nothing committed to a running configuration —
the result is a clean, shareable artifact produced entirely from code.

---

*Self-contained in snippet scope. Recommended companion controls: SSL Forward
Proxy decryption and a DNS Security profile. Strongest deployed together.*
