---
name: project_rls_public_policy_hole
description: "🚨CRITICAL 2026-07-30: 31 tables carry a policy NAMED 'Service role full access' that is actually scoped TO PUBLIC with qual=true — the publishable anon key has full INSERT/UPDATE/DELETE on staff_users, fleet_operators, dispatch_commands, engine_config, api_integrations and 26 more. Proved empirically. NOT fixed — 25 of 31 have no other policy, so restricting it would cut all anon reads too."
metadata: 
  node_type: memory
  type: project
  originSessionId: 9b5e02f2-79ac-4e19-b3aa-031fd558224e
  modified: 2026-07-30T18:08:52.145Z
---

**🚨 THE FINDING (measured + empirically proved on the live DB, 2026-07-30).**

**31 tables** in `public` have a RLS policy named **"Service role full access"** whose actual
definition is:

```
cmd = ALL · roles = {public} · qual = true · with_check = true
```

The **name says service_role; the grant says PUBLIC.** A permissive `ALL/true/true` policy for
`PUBLIC` matches **every** role including `anon`, and `anon` also holds the table-level
INSERT/UPDATE/DELETE grants. So RLS provides **zero** protection on these tables.

**This is the RULE -1(a) trap in POLICY form** — same failure shape as the function-grant trap in
[[reference_db_surgery_lessons]]: a statement that *reads* as service-role-scoped but actually
resolves to PUBLIC. Because the policy is *named* correctly, it passes review by eye. **Always read
`pg_policies.roles`, never the policyname.**

**PROVED, not inferred.** `SET LOCAL ROLE anon` then `DELETE FROM <t> WHERE false` (permission is
checked before rows are matched, so this touches no data), whole block rolled back:
```
engine_config=WRITABLE_BY_ANON  dispatch_commands=WRITABLE_BY_ANON  staff_users=WRITABLE_BY_ANON
fleet_operators=WRITABLE_BY_ANON  api_integrations=WRITABLE_BY_ANON  schedule_tasks=WRITABLE_BY_ANON
```

**Worst of the 31:** `staff_users`, `fleet_operators`, `retail_members` (identity/tenant),
`dispatch_commands` (vehicle dispatch), `engine_config` (orchestration config), `api_integrations`
(integration endpoints/creds), `maintenance_schedules`, `ocpp_sessions`, `vehicle_telemetry`,
`notifications`, `tariff_schedules`, `waves`, `ai_decision_log`.

---

## ✅ FIXED 2026-07-30 — migrations `rescope_service_role_policies_off_public` + `revoke_record_event_from_anon_authenticated`

All 31 policies rescoped `TO service_role` in one `DO` loop over `pg_policies`
(`ALTER POLICY … TO service_role` — no drop/recreate needed). Added
`"Staff read own record"` on `staff_users` (`FOR SELECT TO authenticated USING (auth_user_id = auth.uid())`)
to preserve the one legitimate client path. Also revoked `ottoq_record_event`,
`ottoq_resolve_signing_secret` and `ottoq_sign_event` from PUBLIC/anon/authenticated.

**ACCEPTANCE TEST (the real one):**
```
owner sees  ocpp=59,453  energy=19,846  vsl=539,110      <- engine unaffected
anon  sees  ocpp=0       energy=0       vsl=0            <- RLS now enforcing
anon_exec ottoq_record_event / resolve_signing_secret / sign_event = false
engine write path: signed=t  verify_valid=t  prev_state_null=t
```
Rollback: `ottoq_schema_snapshots` label `pre_rls_public_policy_rescope_2026_07_30`
holds a ready `ALTER POLICY … TO {public}` line per table.

**⚠️⚠️ METHOD CORRECTION — my first "proof" used the wrong instrument.**
I proved the hole with `SET LOCAL ROLE anon; DELETE FROM <t> WHERE false;` and read
`insufficient_privilege` as "RLS blocks". **It does not. That tests the table-level GRANT only.**
RLS *filters rows*; it does not raise on DELETE/UPDATE — a blocked DELETE simply affects 0 rows.
So the probe returned `STILL_OPEN` on all 12 tables *after* a fix that had actually worked, and
would have sent me chasing a non-existent failure.
**Correct instruments:** `SELECT count(*)` as the role (RLS filters reads → 0 rows), or an
`INSERT` (checked by `with_check`, raises `42501`). **Never infer RLS state from DELETE/UPDATE.**

**Why the fix was safe (verified before acting, not assumed):** all 31 tables are owned by
`postgres` and **none** has `relforcerowsecurity`, so the owner — and every SECURITY DEFINER
function and pg_cron job — bypasses RLS entirely; 20 of 31 had never had a row inserted; and a
repo-wide frontend scan found only `staff_users` queried by client code.

**⚠️ A SECOND METHOD TRAP, same session:** my first frontend scan used
`for t in $TABLES` in **zsh**, which does **not** word-split unquoted variables — the loop ran once
with the whole string as one name and returned a confident **zero**. Use an array
(`tables=(a b c); for t in "${tables[@]}"`). A false "nothing uses these tables" would have led me
straight into breaking the cockpit.

**Still open (defence in depth, not urgent):** `anon` retains the table-level INSERT/UPDATE/DELETE
**grants** on these tables. RLS now blocks it, but a future permissive policy would be instantly
exploitable again. Consider `REVOKE` on the grants too.

---

**⚠️ ORIGINAL ASSESSMENT (kept for the record) — why it looked hard before measuring:**
**25 of the 31 tables have NO other policy at all.** The blanket policy is the *only* thing granting
access, so re-scoping it to `service_role` removes **all** anon/authenticated access including
`SELECT` — which would very likely take the cockpit down. The other 6 (`dispatch_commands`,
`site_energy_snapshots`, `exceptions`, `schedule_tasks`, `vehicle_schedules`, `notifications`) have
extra **SELECT-only** tenant-scoped policies, so reads there would survive but writes would stop.

**Staged fix (needs Chase's go — it can break the demo):**
1. Determine what the cockpit actually authenticates as (`anon` vs `authenticated`) and which of the
   31 tables it touches. The frontend repos are `~/Desktop/OTTOYARD/*` — see [[project_cards_integration_map]].
2. Add proper tenant-scoped `SELECT` policies for the tables it legitimately reads.
3. Only then re-scope the blanket policy: `ALTER POLICY "Service role full access" ON <t> TO service_role;`
   (`ALTER POLICY … TO` exists — no need to drop/recreate.)
4. Re-run the `SET ROLE anon` + `DELETE … WHERE false` probe as the acceptance test, per table.
5. Consider also revoking the table-level grants from `anon`, so a future policy mistake is not
   immediately exploitable — defence in depth.

**Related and compounding:** `ottoq_record_event` is **also** `anon`-EXECUTE-able (SECURITY DEFINER),
so the anon key can write arbitrary rows into the append-only audit journal — and since signing went
live on 2026-07-30 those forgeries now carry **VALID** signatures. Blind-revoking it breaks legitimate
paths: 7 non-SECURITY-DEFINER callers run as the invoking role, including the three state-change
trigger functions. `vehicles`/`stalls` are NOT anon-writable so those trigger paths are unreachable by
anon, but `schedule_tasks` IS. See task #11 and [[project_db_capacity_ceiling]] (ninth pass).

**Context on priority:** [[project_deployment_bar]] says security is deferred until post-funding, and
[[project_ottoq_logic_completeness]] already names RLS as *the* deploy gate needing Chase's sign-off.
This finding is a level beyond "hardening" though — it is not a missing control, it is a control that
**appears present and is not**. Anyone with the publishable key (it ships in the frontend by design)
can rewrite dispatch and staff records today.

Links: [[reference_db_surgery_lessons]] (RULE -1(a)), [[project_db_capacity_ceiling]],
[[project_ottoq_twin_boundary]] (48 SECURITY DEFINER writers, no role separation),
[[project_ottoq_logic_completeness]], [[project_deployment_bar]].
