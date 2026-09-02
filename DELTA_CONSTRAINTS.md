# Project DELTA — Standing Constraints

Rules that apply to every session. Read this first.

## Spend
- No provisioning of any infrastructure, and no spend of any kind, without 
  Venkatesh's explicit approval. No exceptions.
- Current spend: $0/month. Nothing has been deployed. Linear free plan, 
  Supabase free tier, no Azure resources exist.
- Projected: ~$13/month for the Azure production footprint once approved. 
  Development is local and free.

## Technical
- All-Azure for deployment: Azure Functions v4 (Node 20), App Service, 
  Front Door + WAF, Service Bus, Blob Storage, Key Vault.
- Supabase Postgres for data (free tier, RLS). Azure Database for PostgreSQL 
  is migration target only — not currently deployed.
- Portability contract: no Supabase-proprietary features outside 
  lib/shared/supabase_client and one RLS claim-helper (current_tenant_id()).
- TypeScript everywhere. Node 20 Functions, Next.js 14, shared Zod schemas.
- Multi-tenant: tenant_id NOT NULL + CHECK + RLS on every table, no exceptions.
- Composite FKs on every child table: UNIQUE(tenant_id, id) on parent, 
  FOREIGN KEY(tenant_id, col) on child. Never single-column FK to id alone.
- No shortcut FKs between two document tables — use match_links.
- Development happens on Venkatesh's laptop. Only deployment touches the cloud.

## Documentation
- No vendor platform names (SAP, Oracle, NetSuite) in customer-facing docs.
- Naming follows OASIS UBL 2.1 conventions — vendor-neutral.
- Consulting-grade output only. McKinsey/Deloitte standard. No bullet walls.
- One document, updated in place. Never spawn a new doc per feedback round.

## Working style
- Be direct. Flag holes upfront. Push back rather than capitulate.
- Ask targeted questions rather than assume.
- Verify claims before stating them — never fabricate a count, figure, or source.
- Repo docs (docs/DELTA_*.md) are the single source of truth. Update them 
  when decisions are made. Claude project docs are deprecated.
