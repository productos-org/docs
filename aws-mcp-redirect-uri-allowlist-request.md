# Request: allow-list ProductOS redirect URIs for AWS MCP Server OAuth

**Status:** ready to send · blocks the ProductOS AWS connector
**Owner:** Heemang (heemang@oneinfinitylabs.com)
**Date raised:** 2026-08-04

---

## The ask, in one line

Add ProductOS's OAuth callback URIs to the AWS Sign-in allow-list for the
`aws-mcp.amazonaws.com` service principal, so ProductOS can be listed alongside
Claude Code, Cursor, Devin, Kiro and the other supported MCP clients.

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

- AWS account team / TAM, if one is assigned to account `300207323857`.
- Otherwise the AWS MCP Server feedback channel referenced in the Agent Toolkit
  user guide, or the `awslabs` GitHub repo for the MCP server.
- Include: the reproduction above, the three URIs, and section "Why this is a
  reasonable client to allow-list" verbatim.
