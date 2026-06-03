# Formaut Runtime — Agent Handoff

## What is Formaut Runtime?

Formaut Runtime is a lightweight infrastructure layer deployed into your own Cloudflare account. It connects your GitHub repo, a Cloudflare Pages site, and two Supabase projects (formaut_os for runtime memory, site_data for editable site content). An external AI agent — running in Claude.ai, ChatGPT, Cursor, or any other tool — can read this file and use it as context to generate or update the site and admin panel.

## What infrastructure already exists

- **Worker 1 (client worker):** A Cloudflare Worker deployed in your Cloudflare account. This is the runtime. It manages memory, writebacks, agent tokens, and site bindings. It does not use Formaut-paid AI; it runs on your own keys and your own bindings.
- **GitHub repo (this repo):** The source of truth for your generated site files and admin panel HTML. Cloudflare Pages deploys from this repo automatically on push.
- **Cloudflare Pages:** Serves the site from the main branch of this repo.
- **Supabase (site_data):** Stores editable site content (hero, services, SEO, contact, hours, etc.) that the admin panel reads and writes.
- **Supabase (formaut_os):** Stores runtime memory, agent tokens, admin emails, and configuration.

## Files that exist in this repo

- `index.html` — Starter site placeholder with FORMAUT:REPLACE markers.
- `admin/index.html` — Starter admin panel shell with FORMAUT:REPLACE:ADMIN_SECTIONS marker.
- `formaut-meta/generation-brief.json` — Instructions for what to generate next.
- `formaut-meta/admin-manifest.json` — Admin panel section manifest (currently empty).
- `formaut-meta/capability-manifest.json` — Installed capability list.
- `formaut-meta/design-decisions.json` — Design decision log (currently empty).
- `formaut-meta/runtime-blueprint.json` — Current runtime mode, infrastructure status, and approval requirements.
- `formaut-site/site-manifest.json` — Site package identity, entry points, and Pages deployment expectations.
- `formaut-site/editable-fields.schema.json` — Fields exposed to the generated admin panel per section.
- `formaut-site/site-data-contract.json` — site_data tables and fields the site and admin panel depend on.
- `formaut-site/buckets.json` — Expected Supabase Storage buckets.
- `formaut-site/auth-policy.json` — Approved admin/editor auth model. No secrets stored.
- `formaut-site/deployment-state.json` — Current deployment source, branch, Pages project, and bootstrap state.
- `AGENT_HANDOFF.md` — This file.

## What to generate next

1. **Site HTML** — Replace the content inside the `<!-- FORMAUT:REPLACE:HERO -->` and `<!-- FORMAUT:REPLACE:CONTENT -->` markers in `index.html` with real site sections (hero, services, testimonials, contact, etc.) based on the business profile in the Formaut memory context pack. Minimal inline CSS only. No build tooling. No external CDN scripts.

2. **Admin panel HTML** — Replace the `<!-- FORMAUT:REPLACE:ADMIN_SECTIONS -->` marker in `admin/index.html` with real editable section modules. Each section should correspond to a Supabase site_data table (e.g., hero, services, seo, contact, hours).

3. **Wire admin → Supabase** — Each admin section should read from and write to Supabase site_data using the agent token and relay URL from the Formaut Dashboard (Settings → Agent Access).

## How the site and admin panel are related

The site HTML in `index.html` reads its dynamic content from Supabase site_data at build time (via Cloudflare Pages build steps or at page load via the relay). The admin panel in `admin/index.html` writes to those same Supabase site_data fields. Changes made in the admin panel are reflected in the live site after the next Pages deployment or a relay-driven live update.

**Important:** The admin panel must never hardcode Supabase URLs, service-role keys, or any other secret. Configuration is provided at runtime by the Formaut Runtime or by the client's secure session. Generated HTML must contain zero secrets.

## Staging changes through Formaut

When available, all generated file changes should be staged as writeback proposals through the Formaut Dashboard approval tools rather than committed directly. This keeps changes reviewable and reversible. Use the writeback/approval path whenever the Formaut Dashboard is accessible.

## MCP / agent access

This runtime has a dedicated per-client MCP Worker (Worker 2, named `formaut-mcp-{slug}`). Use it to connect external agents via the MCP protocol.

**How to connect:**
1. Issue an agent token from the Formaut Dashboard → Settings → Agent Access. Grant scopes: `contract:read`, `delivery:read`, `delivery:write`, `session:write`.
2. Connect your agent to the Worker 2 MCP endpoint: `https://formaut-mcp-{slug}.workers.dev/mcp`
3. Use `Authorization: Bearer fos_...` with your agent token.
4. Call `initialize` then `get_context` to load the full runtime context before doing any work.

**Available MCP tools:**
- `get_context` / `get_delivery_contract` — full runtime context, identity, infrastructure map, admin panel contract, runtime decision tree, approval rules, writeback contracts
- `stage_writeback` — stage a proposed change for client approval
- `list_pending_writebacks` — list proposals awaiting client approval
- `list_installed_capabilities` — installed capability modules
- `get_site_status` — Worker 1 health check
- `submit_session_intelligence` — record session summary

**Approval path:** All staged writebacks require client approval from the Formaut Dashboard → Agent Writebacks before anything is applied. Agents cannot approve their own proposals.

**Alternative (relay path):** If not using the MCP Worker, use the agent token + relay URL from the Formaut Dashboard. The relay URL supports all runtime operations.

> Do not route agent calls through any legacy shared MCP server or through the Formaut platform. Use the per-client Worker 2 MCP endpoint or the relay URL from your dashboard.
