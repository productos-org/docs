# Request: allow-list ProductOS redirect URIs for AWS MCP Server OAuth

**Status:** ready to send · blocks the ProductOS AWS connector
**Owner:** Heemang (heemang@oneinfinitylabs.com)
**Date raised:** 2026-08-04

---

## The ask, in one line

Add ProductOS's OAuth callback URIs to the **Supported Redirect URIs for DCR**
table for the AWS MCP Server, so ProductOS is listed alongside Lovable, Replit
and Vercel v0.

> **There is no self-service path.** That table is maintained by AWS and
> published at
> <https://docs.aws.amazon.com/signin/latest/userguide/aws-mcp-server.html>.
> It cannot be edited from the AWS console, an API, or IAM. The
> `signin:OAuthRedirectUri` condition key only lets an *organization narrow*
> which of the already-approved URIs its own users may use — it cannot widen
> the list. Getting added is a request to AWS.

## Precedent — peer platforms are already approved

The published table (retrieved 2026-08-04) already includes hosted AI
app-builder platforms whose redirect URIs live on their own domains:

| OAuth client | Redirect URI(s) |
| --- | --- |
| Localhost | `localhost`, `127.0.0.1` |
| Claude | `https://claude.ai/*` |
| Cursor Desktop | `cursor://anysphere.cursor-mcp/oauth/callback` |
| Cursor Web | `https://www.cursor.com/agents/mcp/oauth/callback` |
| Visual Studio Code | `vscode://*` |
| Visual Studio Code (Web) | `https://vscode.dev/*` |
| ChatGPT | `https://chatgpt.com/*` |
| ChatGPT Connectors | `https://chatgpt.com/connector_platform_oauth_redirect` |
| **Replit** | `https://replit.com/` |
| **Lovable** | `https://lovable.app/*` |
| **Lovable Developer** | `https://lovable.dev/*` |
| **Vercel v0** | `https://api.v0.dev/v1/mcp-servers/oauth/callback` |

**Lovable, Replit and Vercel v0 are the closest analogues to ProductOS** — hosted
platforms where an AI agent builds and deploys an application into the user's
own cloud account. This is not a novel category for AWS to approve; it is an
existing one that ProductOS belongs to. Lead the request with this.

**URIs to allow-list**

```
https://productos.dev/api/auth/mcp/callback
https://develop.productos.dev/api/auth/mcp/callback
https://design.productos.dev/api/auth/mcp/callback
```

(Confirm the full surface list before sending — one per deployed surface that
exposes the Connectors UI.)

---

## Evidence: what actually happens today

Dynamic Client Registration against `https://us-east-1.oauth.signin.aws/v1/register`
succeeds for loopback and is refused for every public host. Reproduced directly:

```
POST /v1/register  { redirect_uris: [ … ], token_endpoint_auth_method: "none" }

https://develop.productos.dev/api/auth/mcp/callback  →  400 invalid_redirect_uri
    "The redirection URI https://develop.productos.dev/api/auth/mcp/callback
     is not allowed by this server."
https://productos.dev/api/auth/mcp/callback          →  400 invalid_redirect_uri
http://localhost:3000/api/auth/mcp/callback          →  200  client_id=arn:aws:signin…
http://127.0.0.1:8080/callback                       →  200  client_id=arn:aws:signin…
```

This matches the documented model: AWS Sign-in supports discovery + DCR "for
agents running on local workstations **and supported hosted environments**",
with the supported set enumerated in *Supported redirect URIs for the AWS MCP
Server*. ProductOS is a hosted environment that is not on that list.

Reference: <https://docs.aws.amazon.com/agent-toolkit/latest/userguide/oauth-authentication.html>

## What ProductOS is

An AI-native product-development platform. Its Deployment Agent (built on Claude
Code) provisions and operates AWS infrastructure on the user's behalf — Amplify
Hosting, S3 + CloudFront, App Runner, ECS Fargate — from the user's own AWS
account. The AWS MCP Server is the execution layer.

## Why this is a reasonable client to allow-list

Worth stating explicitly, because it is the same bar the existing supported
clients meet:

1. **The credential never reaches user-controlled compute.** ProductOS runs
   agents in per-user sandboxes. The AWS connector is deliberately *proxy-only*
   (`PROXY_ONLY_MCP_SERVER_NAMES`): the OAuth token is held host-side and is
   never written into the sandbox's `.mcp.json`, precisely so a sandboxed agent
   with shell access cannot exfiltrate it and call AWS directly.
2. **Every mutation is classified before it runs.** A static classifier
   (`src/lib/deploy/aws-destructive.ts`) inspects each script host-side.
   Destructive operations (delete/terminate/revoke, IAM writes, lifecycle and
   policy replacement, `desiredCount: 0`) and costly creates (NAT Gateway, RDS,
   ElastiCache, CloudFront) are refused and re-routed through an explicit
   in-product user confirmation card. The classifier **fails closed**: anything
   that hides an operation name behind `getattr`, f-strings, concatenation or
   `eval` is treated as destructive.
3. **Standards-compliant client.** OAuth 2.1 authorization-code + PKCE (S256),
   RFC 9728 → RFC 8414 discovery, RFC 7591 dynamic registration, public client
   (`token_endpoint_auth_method: "none"` — we already negotiate this correctly
   against your metadata).
4. **Tokens are stored encrypted** (AES-256-GCM) and scoped to a single project
   connection, with automatic refresh.

## Two smaller findings AWS may want anyway

Raised as feedback, not as blockers:

1. **`code_challenge_methods_supported` is absent** from
   `https://us-east-1.oauth.signin.aws/.well-known/oauth-authorization-server`.
   RFC 8414 §2 marks it OPTIONAL, so AWS is compliant — but several MCP clients
   (including `@ai-sdk/mcp`, whose `OAuthMetadataSchema` requires it) fail
   discovery outright with a confusing `invalid_type` error. Publishing
   `"code_challenge_methods_supported": ["S256"]` would remove a whole class of
   client-side breakage. We work around it locally.
2. **`invalid_redirect_uri` is returned at registration time** with no
   indication that an allow-list exists or how to join it. Surfacing that in the
   `error_description` would save integrators the reverse-engineering.

## What we do until this lands

The connector is blocked for browser sign-in, so ProductOS uses a cross-account
IAM role (`sts:AssumeRole` + external ID, provisioned by a CloudFormation stack
the user runs) as the credential, rather than a stored long-lived key. See
`Docs/` for the connector design.

## Where to send it

There is no form or console page for this. In rough order of likely success:

1. **AWS account team / Technical Account Manager** for account `300207323857`,
   if one is assigned. This is the fastest route when it exists — ask them to
   route it to the AWS Sign-in / AWS MCP Server service team.
2. **AWS Support case** (Account & Billing → Service limit / feature request),
   category AWS Sign-in. Title it *"Add redirect URI to Supported Redirect URIs
   for DCR — AWS MCP Server"* and cite the published table.
3. **`awslabs` GitHub** — open an issue on the AWS MCP Server / Agent Toolkit
   repo. The peer platforms in the table were plausibly onboarded through
   partner conversations, so a public issue also surfaces the ask.
4. **AWS Partner Network**, if ProductOS is (or becomes) a partner — this is the
   channel Lovable/Replit/v0 most likely used.

Include, in this order: the precedent table with Lovable/Replit/v0 highlighted,
the three URIs, the reproduction, and the "Why this is a reasonable client"
section verbatim.

## Note if you speak to AWS directly

Two things are worth asking in the same conversation, because they affect
whether the connector needs the allow-list at all:

- Is the **client-credentials flow** (`signin create-oauth2-token-with-iam`,
  `--resource aws-mcp.amazonaws.com`) supported for a third-party hosted service
  that holds a customer's cross-account role? If yes, ProductOS can ship without
  waiting for the allow-list, since that flow has no redirect URI.
- Is there a **timeline or criteria** for allow-list additions? That decides
  whether the role-based path is a stopgap or the permanent design.
