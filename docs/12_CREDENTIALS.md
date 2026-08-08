# 12 — Credentials and access

**No secret values appear in this repository, and none ever should.** This file tells you what
exists, what it is for, and **the exact words to use when asking Chase for it.**

---

## 1. Non-secret identifiers (safe, use freely)

These are public identifiers, not secrets.

| Thing | Value |
|---|---|
| Supabase org | `gdqqjpxbgpperhgyfkes` ("OTTOYARD") |
| **Core project ref** | **`gxdrcyphqjzjsuhxuqtg`** |
| Core project URL | `https://gxdrcyphqjzjsuhxuqtg.supabase.co` |
| Core DB host | `db.gxdrcyphqjzjsuhxuqtg.supabase.co` (Postgres 17.6, us-east-1) |
| OrchestrAV legacy project ref | `ycsisvozzgmisboumfqc` ("OTTOYARD MVP", us-east-2) |
| Abandoned project ref | `sovyxwtrqfmizelrammm` ("Fleet Dashboard", INACTIVE) |
| ⚠️ Dead reference | `hfjaofyfxsyniohdfacg` — **does not exist.** If you find it in a config, it is stale. |
| GitHub org | `https://github.com/OTTOYARD` |
| Edge function base | `https://gxdrcyphqjzjsuhxuqtg.supabase.co/functions/v1/` |
| API gateway | `/functions/v1/otto-q-api/api/v1/...` |
| Flagship depot id | `11111111-1111-1111-1111-111111111111` |
| Benchmark depot id | `22222222-2222-2222-2222-222222222222` |
| NVIDIA cuOpt endpoint | `https://optimize.api.nvidia.com/v1/nvidia/cuopt` |
| Nemotron model id | `nvidia/nemotron-3-ultra-550b-a55b` |
| NVIDIA npm registry | `https://edge.urm.nvidia.com:443/artifactory/api/npm/omniverse-client-npm/` (public, no auth) |
| Isaac Sim image | `nvcr.io/nvidia/isaac-sim:6.0.1` |

---

## 2. What you will need, and how to get it

### Already granted (per Chase, 2026-08-08)
> *"Give it full access to view and read through everything, including private repos… It should have
> the same authorization as well."*

So you should have:
- **Read access to all 9 OTTOYARD GitHub repos**, including the private ones.
- **Read and query access to the Supabase projects.**
- **Branch and PR rights.**

If any of that is missing, that is the first thing to ask for.

### Environment variables the system uses

From `~/Desktop/OTTO-Q V1/.env.example` and the edge-function environment. **Names only.**

```
# Supabase
SUPABASE_URL
SUPABASE_ANON_KEY                # publishable; safe in a client bundle
SUPABASE_SERVICE_ROLE_KEY        # SECRET — server only, never in a client

# NVIDIA
NVIDIA_API_KEY_CUOPT             # cuOpt hosted API
NVIDIA_API_KEY / NIM key         # Nemotron NIM
NGC token                        # only for pulling nvcr.io containers

# AI
ANTHROPIC_API_KEY                # used by ottoq-ottocommand

# Frontier bridge
OTTOQ_INTELLIGENCE_URL           # the EC2 FastAPI service
OTTOQ_BRIDGE_TOKEN               # x-bridge-token header
                                 # ⚠️ currently HARDCODED in edge fn ottoq-energy-mpc
                                 #    and the replan defaults — see P2-6

# AV fleet APIs (not yet live)
TESLA_FLEET_CLIENT_ID / _SECRET / _API_BASE
WAYMO_FLEET_API_KEY / _API_BASE
ZOOX_FLEET_API_KEY / _API_BASE

# Chargers
OCPP_CSMS_BASE_URL / _API_KEY / _BASIC_AUTH

# Energy
BESS_API_BASE_URL / BESS_API_KEY

# Notifications
TWILIO_ACCOUNT_SID / _AUTH_TOKEN / _FROM_NUMBER
RESEND_API_KEY

# Signing
WEBHOOK_SIGNING_SECRET           # HMAC for OEM webhook signatures

# Config
DEFAULT_DEPOT_ID
DEFAULT_DEPOT_TIMEZONE           # America/Chicago
ENABLE_AI_ENGINE / ENABLE_BESS_INTEGRATION / ENABLE_EARLY_RECALL / ENABLE_DEMAND_SHAPING
```

**Where they live:** Supabase edge-function secrets (Dashboard → Project Settings → Edge Functions →
Secrets), and the `ottoq_signing_keys` / vault path for signing material.
⚠️ **No live API keys are stored in the vault today** except `ottoq_anon_key`. The calibration
corpora were ingested offline from public sources; **live pollers would need keys Chase must
provision.**

### AWS

| Resource | Notes |
|---|---|
| **EC2 `g6e.2xlarge`** — NVIDIA L40S 48 GB | Runs the Isaac Sim container. SSH `ssh -i <key>.pem ubuntu@<ip>` (add `-o ServerAliveInterval=30`; sessions idle-drop). Relaunch: `bash ~/ottoq/launch_isaac.sh`. **Stop when idle (~$2–4/hr).** |
| **EC2 hosting `ottoq-intelligence`** | FastAPI on :8080. |
| Security group | Must have 49100/TCP, 47998/UDP, 8210/TCP open for the Isaac stream. |
| Credits | ~$100k NVIDIA Inception (partially used). ~$10k separate AWS credits **reserved for Phase 2: IoT Core MQTT + command API.** |

⚠️ **Claude has never had AWS console access** and neither should you by default. Chase performs
console actions (start/stop instances, security groups, Elastic IPs). **If you need one, ask with
the exact steps.**

---

## 3. How to ask for a credential

Use this shape. Be specific about *what*, *why*, *where it goes*, and *what happens without it*.

> **Access request — <thing>**
>
> **What I need:** <exact name, e.g. `NVIDIA_API_KEY_CUOPT`>
> **Why:** <the specific task it unblocks>
> **Where it goes:** <e.g. "Supabase → Project Settings → Edge Functions → Secrets, as
> `NVIDIA_API_KEY_CUOPT`" or "my agent secret store, never written to a file">
> **Exact steps for you:**
> 1. <step>
> 2. <step>
> **If I don't get it:** <what I will do instead, or what stays blocked>

**Worked example:**

> **Access request — AWS security group ingress**
>
> **What I need:** ports 49100/TCP and 47998/UDP open on the `g6e` instance's security group.
> **Why:** the Isaac Sim WebRTC stream cannot reach the cockpit without them; the RTX tab shows a
> blank viewport.
> **Where it goes:** AWS console only — I have no AWS access and am not asking for any.
> **Exact steps for you:**
> 1. EC2 → Instances → select the `g6e` box → Security tab → click its security group.
> 2. Inbound rules → Edit → Add rule: Custom TCP, port 49100, source `0.0.0.0/0`.
> 3. Add rule: Custom UDP, port 47998, source `0.0.0.0/0`. Save.
> **If I don't get it:** Tier-B photoreal work is blocked; I will continue on Tier-A motion instead.

---

## 4. Hard rules about credentials

1. **Never print a secret** into a log, a PR description, a commit message, a test fixture, or a
   chat message.
2. **Never commit a secret**, not even in a file you intend to delete. Git history is forever.
3. **Never rotate a credential** on your own. Ask.
4. **Never move a secret from a secret store into a file** "temporarily."
5. **Flag hardcoded secrets you find** rather than fixing them silently — the fix usually needs a
   coordinated deploy. (Known: `OTTOQ_BRIDGE_TOKEN` and the AWS URL in `ottoq-energy-mpc`.)
6. **If a key is exposed, say so immediately.** It is not embarrassing; it is urgent.

---

## 5. The Mapbox incident — the reason rule 6 exists

**A leaked Mapbox token cost $1–2k.**

**Standing policy, binding:**
- **MapLibre with free tiles.** Never a metered raster provider.
- **Never a committed token**, in any repo, in any form.

If any map work happens in the cockpits, this policy governs it.
