# Backend restart after a domain change (the reliable runbook)

A **domain change** (new/changed entity, property, reference list, or migration) only takes effect
after the .NET backend is **rebuilt and restarted** — Shesha applies migrations and seeds
`EntityConfig` on startup. Doing this badly is the single biggest cost/failure sink (one harness run
burned **$12.50 / 27 min** on 14 restart attempts and corrupted an existing form). Follow this
runbook instead of improvising.

> **Order of operations:** do ALL domain changes + the restart **first**, then build forms **last**.
> A form built before the entity is ready won't render; and a form pushed *before* a later restart
> can be orphaned by that restart (see "re-verify forms" below).

---

## Rule 0 — never relaunch IIS Express outside Visual Studio

The dev backend is usually hosted by **IIS Express**, launched and managed by Visual Studio. Its
`applicationhost.config` uses `hostingModel="InProcess"` with `processPath="%LAUNCHER_PATH%"` — that
env var is only set by VS, so relaunching `iisexpress.exe` yourself gives **`HTTP Error 500.0 — ANCM
In-Process handler load failure`**. Do **not** try to relaunch IIS Express. Use the Kestrel path below
(headless) or hand back to VS (attended).

---

## Ephemeral / shesha-agent sandbox — check this FIRST

If the `SHESHA_AGENT_API_URL` and `SHESHA_SESSION_ID` environment variables are set, you're running
inside a shesha-agent ephemeral session, not on a developer's machine or in CI. The backend there
already runs under `dotnet watch`, and a separate system handles rebuild/redeploy — **do not** take
over the port or run `dotnet build`/`dotnet run` yourself (the Headless section below does exactly
that, and it will race the live process, producing MSB3027/MSB3021 file-copy errors). Instead,
trigger the restart yourself over shesha-agent's own API, then poll for it to finish. This is two
steps, not one — **do not use a long `--max-time`/`-m` and expect the trigger call itself to wait
for the rebuild**: it only waits for the old process to be confirmably replaced (a matter of
seconds), never the full build, so a short timeout on the trigger call is correct and expected:

```bash
curl -s -X POST --max-time 60 "$SHESHA_AGENT_API_URL/api/app-restart/$SHESHA_SESSION_ID/restart-changed-apps"
```

Then poll `app-statuses` yourself until **`backend`** specifically reports `"running"` — check only
the one app type you actually care about, not whether every app in the session is running (an
admin/public portal can take much longer to cold-start its own `npm install` and is irrelevant to
whether your backend change is live). Use the Bash tool's own `timeout` parameter for the overall
budget (e.g. 400000ms) rather than a curl flag - this is a polling loop, not a single request.

Each app's entry in `app-statuses` is an object — `{"status": "...", "errorMessage": "..."}` — not a
plain string. **`"failed"` is a real, distinct status**: it means the build genuinely broke (a
compile error, most often), and it will never become `"running"` on its own no matter how long you
poll. Treat `"failed"` as a hard stop, not "still in progress":

```bash
for i in $(seq 1 40); do
  RESPONSE=$(curl -s "$SHESHA_AGENT_API_URL/api/app-restart/$SHESHA_SESSION_ID/app-statuses")
  STATUS=$(echo "$RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin).get('data',{}).get('backend',{}).get('status','unknown'))")
  echo "[$i] backend: $STATUS"
  [ "$STATUS" = "running" ] && break
  if [ "$STATUS" = "failed" ]; then
    ERROR=$(echo "$RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin).get('data',{}).get('backend',{}).get('errorMessage','no detail'))")
    echo "Backend build FAILED: $ERROR"
    break
  fi
  sleep 10
done
```

**If you hit the `"failed"` branch**: stop polling immediately — do not loop, retry the restart, or
re-poll hoping it resolves itself. Read the reported error, go fix the actual source file it points
to, then trigger `restart-changed-apps` again. Cap yourself at **2 targeted fix attempts** for the
same error; if it still fails after that, stop and report the exact error to the user instead of
continuing to retry — an unbounded fix-and-retry loop against an error you can't actually resolve is
exactly the pattern that burns API credits for no progress.

Then verify against `$SHESHA_BACKEND_URL` (never `localhost`) — including the 2-boot lag described
below for a brand-new entity: poll `Crud/GetAll`; a 404 means the entity's `EntityConfig` was only
just seeded and its dynamic controller needs one more boot. Nothing changed on disk since the first
restart, so `restart-changed-apps` won't detect a reason to run again — force the second boot
instead (same two-step pattern: trigger, then poll):

```bash
curl -s -X POST --max-time 60 "$SHESHA_AGENT_API_URL/api/app-restart/$SHESHA_SESSION_ID/force-restart" \
  -H "Content-Type: application/json" -d '{"appTypes":["backend"]}'
# then poll app-statuses for backend == "running" exactly as above
```

Re-check `Crud/GetAll` once more — it should be live now. Then run `/test-entity-crud-api` yourself
as the final step, exactly as the calling skill's "MANDATORY... THEN test" instruction requires —
don't stop short and ask the user to run it themselves; the point of this whole mechanism is that you
have everything needed to complete the loop within this turn.

Skip the rest of this doc in this case — the Headless and Attended sections below both assume you can
freely stop/rebuild/relaunch the backend process yourself, which you must never do here.

---

## Headless / CI / harness — take over the port with Kestrel

You're headless when the task supplied a context block (Backend URL / Module / Working directory) or
you're in `claude -p`. Run this **once** as a single combined sequence (don't probe step-by-step):

```bash
WH="<workingDir>/backend/src/<App>.Web.Host"          # e.g. .../boxfusion.test/backend/src/boxfusion.test.Web.Host
DLL="$WH/bin/Debug/net8.0/<App>.Web.Host.dll"          # e.g. boxfusion.test.Web.Host.dll
BASE="http://localhost:21021"

# 1. Stop whatever holds the port (IIS Express + its tray, or the port owner)
powershell -NoProfile -Command "Get-Process iisexpress,iisexpresstray -ErrorAction SilentlyContinue | Stop-Process -Force"
# (fallback: kill the PID returned by `Get-NetTCPConnection -LocalPort 21021 -State Listen`)

# 2. Build (Web.Host build compiles the Domain project + migration too)
dotnet build "$WH/<App>.Web.Host.csproj" -c Debug --nologo -v m

# 3. Launch Kestrel in the BACKGROUND on :21021 (NOT `dotnet run` — run the built DLL; it's faster and
#    avoids a rebuild). ASPNETCORE_ENVIRONMENT=Development is required (Production 500s here).
ASPNETCORE_ENVIRONMENT=Development ASPNETCORE_URLS="$BASE" dotnet "$DLL"   # run via run_in_background

# 4. Poll until Shesha finishes booting (migrations + bootstrappers ~10–30s)
#    until: curl -s -o /dev/null -w '%{http_code}' "$BASE/swagger/index.html"  == 200
```

Launch step 3 with `run_in_background: true` (it's a long-lived server) and then poll in a separate
call. Write any scratch scripts into `<workingDir>`, **not `/tmp`** (git-bash `/tmp` ≠ Windows paths).

### The 2-boot lag (new entities only) — handle it deterministically

A **newly added** entity's dynamic CRUD controller registers only on the boot *after* its
`EntityConfig` is seeded. So the first boot brings the app up but the new entity's endpoint 404s. After
step 4 succeeds, verify the entity and restart **once more** if needed — don't flail:

```bash
# dynamic CRUD endpoint format:  /api/dynamic/<module>/<Entity>/Crud/GetAll
code=$(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/dynamic/<module>/<Entity>/Crud/GetAll?MaxResultCount=1")
# 200 → ready.  404/500 → repeat steps 1–4 ONCE; the controller registers on the second boot.
```

For a brand-new entity, just **plan for two boots** up front (build once, then boot → boot) rather than
discovering the 404 and reacting.

---

## After ANY restart — re-verify the forms you'll touch

Startup re-runs the configuration bootstrappers (`ConfigurableModuleBootstrapper` /
`ImportConfigurationAsync`), which can leave a previously-edited form without its "live" revision:
`FormConfiguration/GetByName` (and the `/dynamic/<mod>/<name>` route) return **404**, while
`GetJson?id=<id>` still returns the markup. To restore name-resolution, **re-push the markup**:

```bash
# if GetByName 404s but you have the id + markup:
curl -s -X PUT "$BASE/api/services/Shesha/FormConfiguration/UpdateMarkup" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"id":"<id>","markup":"<stringified markup>"}'
```

Because you build forms **after** the restart, a clean single run normally won't hit this — it bites
when an earlier form exists and a later domain change forces a restart. Re-verify defensively.

---

## Attended / real-world (Visual Studio is running the app)

Do **NOT** kill VS's IIS Express or take over the port — that breaks the developer's session. Instead:

1. Tell the developer: *"I created/changed an entity + migration — rebuild and restart the app in
   Visual Studio (Stop ▸ Build ▸ Run), then I'll continue."* For a **new** entity, ask them to restart
   **twice** (the 2-boot lag).
2. Poll the entity's `Crud/GetAll` until it returns 200, then resume form work.
3. Only offer the headless Kestrel takeover above if the developer explicitly prefers it.

This keeps the skill usable in normal development, not just the test harness.
