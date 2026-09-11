# Security

Snø Desk is designed to place trades in real brokerage accounts. Everything below is a design commitment, not a description of shipped code — **there is no code yet.** The threat model is published first so it can be challenged before an implementation exists.

## Threat model

| Threat | What goes wrong | Defense |
|---|---|---|
| **Prompt injection** | A news article, filing, email or web page contains text instructing the agent to sell. | **Two channels.** Content the agent reads is data and is structurally unable to produce an `OrderIntent`; only direct user input can. Enforced in types and code, not requested in a prompt. |
| **Misparsed instruction** | "Sell 100" becomes 1,000, or "half" is applied across every account. | Typed `OrderIntent` with explicit account, quantity kind and order type. Ambiguities are listed on the ticket. Per-order caps in the risk gate. Human approval on every ticket. |
| **Credential theft** | A config file or backup containing broker tokens is exfiltrated. | Tokens live only in the OS keychain. Nothing credential-shaped is written to disk, logs, backups or the journal. |
| **Malicious shared agent** | Someone shares an agent that trades against the person who imports it. | Imported agents arrive `paused` and preview-only and cannot arm themselves. There is no marketplace that can execute. |
| **Expired or stale auth** | An expired token silently serves cached positions as current. | Hard stop with a specific, loud error. Never a cached fallback presented as live data. |
| **Tampered release** | A modified binary or install script steals credentials. | Signed releases, published checksums, reproducible builds. No `curl \| sh` installation. |
| **Runaway agent** | An armed agent fires repeatedly on a flapping condition. | Per-agent fire limits and cool-downs, plus a global kill switch: `desk halt`. |
| **Local compromise** | Malware is already running on the user's machine. | Out of scope to fully defend. Execution can be disabled per account, and broker-side revocation steps are documented for every adapter. |

## Invariants

These must hold in every release. A change that breaks one is a security bug, regardless of intent.

1. No code path lets model output call a broker adapter directly.
2. No `OrderIntent` originates from retrieved or otherwise untrusted content.
3. Execution is disabled by default for every newly connected account.
4. Every routed order has a stored approval record and the exact user text that produced it.
5. No credential is ever written outside the OS keychain.

## Reporting a vulnerability

The repository is private during the design phase. Once public, report vulnerabilities through GitHub's private vulnerability reporting on this repository — never in a public issue.
