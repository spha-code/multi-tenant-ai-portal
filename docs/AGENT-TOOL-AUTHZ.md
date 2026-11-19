# Agent tool authorization — design note

How `/agent/chat` decides **which tools a tenant may call**, and how a write is
proven to be human-approved. Data isolation (RLS) is handled elsewhere; this
note is only about *tool* authorization.

> Status: draft — first step of [#40](https://github.com/spha-code/multi-tenant-ai-portal/issues/40).
> Related: [PLAN.md](../PLAN.md) (agent loop), [AUTHENTICATION.md](./AUTHENTICATION.md) (tenant identity).

## Problem

`tenant_session(principal.tenant_id)` makes the *data* safe, but every tenant's
agent can still call every tool. A tenant that should only read orders would
still get `create_order`, and a client that ignores the confirm card can trigger
writes directly. There is also no record of which tool ran for which tenant.

## Decisions

1. **Allowlist per tenant.** Add `tenant_settings.allowed_tools` (JSON array).
   Default to read tools only (`query_customers`, `query_orders`); writes such
   as `create_order` must be explicitly granted.

2. **Enforce server-side, in the dispatcher.** Resolve the allowlist from the
   principal's tenant on every request. A tool that is not in the list is
   rejected before execution and surfaced as a typed SSE `error` event.

3. **Writes need a server-issued confirmation token.** The model proposes an
   action; the backend stores it as a pending action and returns a token. The
   confirm card POSTs that token back. The client's claim that "a human
   approved" is never trusted on its own.

4. **Audit every call.** Emit
   `{tenant_id, user_id, tool, args_hash, outcome}` — a hash of the arguments,
   never the raw arguments, to avoid logging PII.

## Proposed request flow

1. Token is verified → `principal` (user + tenant) is built.
2. Load `tenant_settings.allowed_tools` for `principal.tenant_id`.
3. Model requests a tool → dispatcher checks the allowlist.
4. Read tool → execute inside `tenant_session(principal.tenant_id)`.
5. Write tool → create pending action, return confirmation token to the client.
6. Client confirms → backend validates the token, executes, and records the
   audit event.

## Acceptance criteria (from #40)

- [ ] Tool not in `allowed_tools` is rejected server-side, never executed
- [ ] `create_order` cannot run without a valid confirmation token
- [ ] Two tenants with different allowlists behave differently (test)
- [ ] Audit event written per tool call
- [ ] Reflected in `PLAN.md`

## Open questions

- Token TTL and single-use semantics for pending write actions.
- Where pending actions live (Postgres table vs. short-lived cache).
- Whether the allowlist is editable by tenant admins or only by operators.
