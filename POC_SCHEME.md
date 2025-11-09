# Proof-of-Concept Scheme — Soundness CLI + Walrus Quests

This note documents the high-level workflow we use to keep Sui Quest Platform secure while letting users generate zero-knowledge proofs through the Soundness CLI. It is intentionally descriptive (not low-level) so product, ops, and engineering stakeholders share the same mental model.

MY WEB PROJECT : 
- https://connect-sphere.xyz/
- https://connect-sphere.xyz/login
- https://connect-sphere.xyz/register

---

## 1. Actors & Components

| Actor / Component | Responsibility | Key Notes |
|-------------------|----------------|-----------|
| Learner (end user) | Solves quiz questions, pays the proof fee, runs Soundness CLI, submits digest. | Needs a linked Sui wallet + hCaptcha challenge. |
| Web Frontend | Guides quiz attempts, fetches proof fee status, invokes payment & proof job APIs, displays release/use prompts. | Manages local cache (`pending-fee:<slug>`) and session storage. |
| Backend API | Issues signed nonces, validates proof fees, tracks reservations, queues prover jobs, verifies digests. | Single source of truth in Postgres (tables: `proof_payments`, `proof_jobs`, `pending_digests`). |
| Soundness CLI | Executes the quest payload with the proof token + nonce and uploads proof blobs via Walrus. | Runs locally; reuses same nonce/token in digest submission. |
| Walrus Network | Stores compiled wasm payloads and proof blobs referenced by on-chain transactions. | Provides blob IDs used in CLI command templates. |

---

## 2. Flow Timeline (Happy Path)

1. **Preparation**
   - User links wallet (signed nonce) and passes quiz (≤3 guided retries).
   - Quest detail page loads queue metrics and proof-fee status.

2. **Fee Reservation**
   - Frontend fetches `/proof-fee/status`. No reservation → Pay CTA enabled.
   - User hits “Pay & generate”; backend returns preflight (balance) and message (nonce + text to sign).
   - User signs the authorization message, broadcasts a SUI transfer to the operator address, and submits the resulting digest + signature + nonce token.
   - Backend verifies the digest (amount, sender, recipient), stores a `proof_payments` row with `consumed_at = NULL`, and blocks other quests from receiving new payments until this one is consumed or released.

3. **Proof Job Creation**
   - Immediately after fee confirmation, frontend invokes `/quests/:slug/proof-jobs`.
   - Backend enqueues a Ligero WebGPU prover job tied to the quest’s Walrus payload template and a freshly issued proof token + command nonce.
   - Worker builds the proof artifacts, uploads them to Walrus, and exposes a Soundness CLI command.

4. **Soundness CLI Run**
   - CLI command includes: proof blob ID, ELF blob ID, proof token, command nonce, wallet public key, XP reward, etc.
   - Quest payload (wasm) enforces the provided nonce/token; tampered payloads cannot pass verification.
   - CLI outputs a digest once on-chain submission is completed.

5. **Digest Submission**
   - User submits the digest via `/api/quests/submit/digest` along with the signature + nonce token from the CLI output.
   - Pending digest worker validates the signature, confirms nonce consumption, and finalizes XP.

6. **Post-Verification**
   - `proof_payments.consumed_at` is set, allowing the user to pay for another quest if desired.
   - UI clears pending XP banners and surfaces the proof session link as “Completed”.

---

## 3. Data & State Overview

| Table | Purpose | Notable Fields |
|-------|---------|----------------|
| `proof_payments` | Tracks every verified proof fee. | `quest_slug`, `tx_digest`, `amount_mist`, `verified_at`, `consumed_at`. |
| `proof_jobs` | Queue of prover workloads. | `status`, `session_token`, `payload_path`, `command_nonce`. |
| `pending_digests` | Digests awaiting chain verification or auto-submission. | `transaction_digest`, `submitted_signature`, `command_nonce`. |
| `signed_nonces` | Nonces for wallet link, proof fees, commands, and digest submissions. | `purpose`, `quest_slug`, `wallet_address`, `consumed_at`. |

State transitions for a proof fee:

1. **New** — row inserted with `consumed_at = NULL`.
2. **Bound to quest** — user sees “Use reserved fee” CTA; backend prevents other quests from requesting proof fees.
3. **Consumed** — after job queueing + digest verification, `consumed_at` is set.
4. **Released (optional)** — user manually releases via `/proof-fee/release`; row deleted; no refund occurs.

---

## 4. Reserved Fee Behaviour & Edge Cases

- **Single reservation per user**: Backend rejects `/proof-fee/preflight` and `/proof-fee/message` if any `proof_payments` row is still unconsumed. Frontend mirrors this by disabling the Pay button and displaying a warning (“You already reserved fee for quest X”).
- **Same quest reuse**: If the reservation matches the current slug, the primary CTA switches to “Use reserved proof fee”. Clicking it queues a new proof job without another wallet transaction.
- **Release confirmation**: The release modal requires typing the quest slug and explicitly states that the SUI transfer is not refunded. Only after confirmation is the reservation removed, freeing the user to pay for another quest.
- **Unexpected reloads**: If the browser closes during verification, `pending-fee:<slug>` ensures the saved digest + nonce are retried when the page reopens.
- **RPC delays**: Backend polls the Sui RPC (10 attempts with backoff) before declaring the payment invalid. If all retries fail, the UI keeps the pending state and prompts the user to refresh later.

---

## 5. Security & Abuse Controls

1. **Signed nonce enforcement** across wallet link, proof fee, CLI command, and digest submission to prevent replay attacks.
2. **Proof token handshake** ensuring only backend-issued tokens (HMAC, quest-scoped) can unlock Walrus uploads and digest submissions.
3. **Per-quest locking** in `proof_payments`, blocking double-spend attempts across quests.
4. **CLI payload integrity** via command nonce embedded in wasm.
5. **Quiz throttling** (three check attempts) and mandatory hCaptcha to deter scripted brute force.
6. **Auditability** through unique tx digests and explorer URLs stored per payment.

---

## 6. Operational Checklist

1. **Bootstrap**
   - Run migrations (`sqlx migrate run`).
   - Upload the latest `quest_payload.wasm` to Walrus and update `quest_proof_configs`.
   - Keep `.env` secrets (`PROOF_TOKEN_SECRET`, `SIGNED_NONCE_SECRET`) long and environment-specific.

2. **Routine Validation**
   - Reset dev database, register a new account, and perform the full flow end-to-end.
   - Confirm reserved-fee banner, release modal, CLI output, and digest verification.
   - Run `npm run lint` and `cargo check` before deployment.

3. **Monitoring**
   - Watch logs for `Proof fee required`, `Proof job completed successfully`, and `Pending digest verified`.
   - Track queue metrics (pending/running jobs, average wait) surfaced on the quest detail page.
   - Investigate any repeated “Payment digest tied to a different quest” warnings—they signal users racing quests simultaneously.

---

## 7. Future Enhancements

1. **Wallet Extension Detection**
   - Detect when MetaMask, OKX, or other extensions override `window.ethereum`.
   - Provide a single-click fallback to Slush wallet or a Web3Modal-style selector so users know which extension controls Sui signing.

2. **Finance & Ops Reporting**
   - Export `proof_payments` and `xp_events` to CSV/Snowflake for finance reconciliation.
   - Include per-quest WAL/SUI cost, XP awarded, and success rate to inform sponsorship budgets.

3. **Self-Service CLI Endpoint**
   - Allow power users to request `proof_token` + `command_nonce` directly (with JWT auth) so they can script proof generation without going through the UI builder.
   - Enforce the same nonce/token guards to keep parity with the web flow.

4. **Concurrency & Queue UX**
   - Add a global banner when the user has an active reservation for another quest (“Release fee for walrus-storage-basics to continue here”).
   - Surface average queue wait times in the dashboard header so learners can plan their CLI runs.

5. **Web3 Social Integration**
   - Let users share proof completion badges directly to Farcaster, Lens, or Warpcast with deep links to the Suiscan transaction.
   - Offer optional on-chain attestations (e.g., Sui Kiosk NFT) so achievements propagate to third-party reputation systems.

6. **Smart Notifications**
   - Push notifications (email, Telegram, or WalletConnect ping) when a proof job finishes or when pending fee is about to expire.
   - Auto-remind users to release unused fees before TTL to avoid manual support requests.

These upgrades keep the flow secure while making it more social, transparent, and automation-friendly, paving the way for future partner quests.

This POC scheme keeps the payment-to-proof pipeline auditable while offering a predictable experience for learners. Treat it as the blueprint for future Walrus quests or partner programs. Adjust the numbers (fee amount, TTLs, queue capacity) per environment, but preserve the reservation + nonce guarantees described above.
