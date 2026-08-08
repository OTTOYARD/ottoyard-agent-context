---
name: OTTOYARD Tech Stack
description: Supabase backend, TypeScript engine, Lovable dashboards, OCPP for chargers, per-OEM AV fleet adapters
type: reference
---

- Database: Supabase (PostgreSQL 15 + PostGIS)
- API Layer: Supabase Edge Functions (Deno)
- Engine: TypeScript (pure business logic)
- Dashboards: Lovable (Depot Ops App, OrchestraAV, OrchestraEV)
- Charger Protocol: OCPP 2.0.1
- AV Integration: Per-platform adapters (Waymo, Tesla, Motional, Zoox, etc.)
- Notifications: Twilio (SMS), Resend (email), Supabase (push)
- AI/Optimization: Anthropic API (Claude) for predictive engine
- Tow Integration: OTTOW dispatch system
