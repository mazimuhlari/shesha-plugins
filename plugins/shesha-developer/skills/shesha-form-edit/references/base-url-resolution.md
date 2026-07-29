# Resolving app URLs across environments

Any skill that needs to reach a running backend or admin portal should resolve the URL this same
way, wherever the project happens to be running — a developer's machine, CI, or a shesha-agent
ephemeral sandbox — rather than hardcoding `localhost` or re-deriving its own detection. One place
to update this logic if it ever needs to change.

## Backend URL

Check in this order, stopping at the first that applies:
1. **Task-supplied context block** (headless/CI runs, or a `claude -p` invocation that supplied
   one) — always wins.
2. **`$SHESHA_BACKEND_URL` environment variable** — set automatically in a shesha-agent ephemeral
   session, where the backend runs in a separate process and is never reachable at `localhost`.
   Use it directly and skip the steps below.
3. `src/*.Web.Host/Properties/launchSettings.json` (`profiles.Project.applicationUrl`).
4. `src/*.Web.Host/appsettings.json` (`Kestrel:Endpoints:Http:Url` or `ServerRootAddress`).
5. Fallback: `http://localhost:21021`.

Strip any trailing slash and store as `{BASE_URL}`. Ping `{BASE_URL}/swagger/index.html` to confirm
it's actually reachable before relying on it for anything else.

If your task involves code you (or another skill in this same turn) just wrote — not just reading
existing data — the backend may need rebuilding before `{BASE_URL}` reflects it. See
`shesha-developer/skills/shesha-form-edit/references/backend-restart.md` for how to handle that
across environments, including the ephemeral sandbox's self-serve restart API.

## Admin portal URL (browser-driven checks only)

1. Task-supplied context block, if present — always wins.
2. **`$SHESHA_ADMIN_PORTAL_URL` environment variable** — set the same way, for the same reason, in
   a shesha-agent ephemeral session.
3. The dev server's own default (usually `http://localhost:3000` for `npm run dev`) — check
   `package.json`/`next.config.js` if unsure.

If neither environment variable is set and no local dev server can be found, say so and stop rather
than guessing a port.
