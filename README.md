# ProductOS Docs

User-facing documentation for ProductOS (productos.dev), built for [Mintlify](https://mintlify.com).

Structure:

- `docs.json` defines navigation (tabs: Documentation, Agents, Integrations, Resources), theme, and branding.
- Pages are MDX files organized by tab: `getting-started/`, `build/`, `code/`, `design/`, `platform/`, `agents/`, `integrations/`, `resources/`.
- Content is grounded in the internal wiki (`../wiki/`) and the verified agent roster (`../productos-website/lib/agents.ts`). When the product changes, update the wiki first, then mirror here.

## Local preview

```bash
npm i -g mint
cd productos-docs
mint dev
```

Preview at http://localhost:3000.

## Publishing (one-time setup)

1. Push this folder as its own GitHub repo (e.g. `productos-org/productos-docs`).
2. In the [Mintlify dashboard](https://dashboard.mintlify.com), create a project and install the Mintlify GitHub app on that repo.
3. Set the custom domain to `docs.productos.dev` (dashboard: Settings, Custom domain) and add the CNAME in the DNS provider.
4. Every push to main auto-deploys.

**DNS warning:** never point productos.dev DNS at Mintlify (their domain-setup
flow suggests a CNAME to cname.mintlify.builders). The site uses the
reverse-proxy option instead: productos.dev stays on Vercel, and the website's
next.config rewrites /docs/* to productos.mintlify.dev. An apex CNAME to
Mintlify put the whole domain behind their edge and intermittently served the
docs at the homepage (incident: 2026-07-16).

## Conventions

- No em dashes in copy (house rule).
- Never document UI click-paths that the wiki or code does not confirm; describe workflows at the level of stages and capabilities.
- Every page has frontmatter `title` + `description`.
