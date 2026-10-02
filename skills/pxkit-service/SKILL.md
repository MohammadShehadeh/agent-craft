---
name: pxkit-service
description: Write a service or server boundary the pxkit way — one plain http client, zod-parsed responses and payloads, DTOs parsed to domain types once, a narrowed Result with SCREAMING_SNAKE error keys, and no throws to the UI. Use whenever the user asks to call an API, fetch or submit data, add a server action, route handler, webhook, database query, or integrate a vendor SDK (Stripe, Slack, GitHub, …), and whenever a task would write fetch(), axios, or process.env in feature code, even if they don't say "service".
---

# pxkit Service & Boundary

Every rule about talking to the outside world lives here: the `http` client, `Result`, error keys, parsing, server actions, route handlers, and env. Load `pxkit-conventions` for the rules that apply to all code.

## Gates

- **Present the operation contract (step 2) and wait for confirmation** before creating files.
- **No `fetch`, headers, `try/catch`, or timeout in a service.** The `http` client owns them.
- **No throw crosses into the UI.** Expected failures return `{ ok: false, errorKey }`.
- **No DTO type outside the file that parses it.**
- **No `process.env` outside `env.ts`.**
- **No base service class, generic repository, client factory, or per-feature `http` wrapper.**

## 1. Inspect the repo

Find and reuse what exists: `types/result.ts`, `lib/http.ts`, `env.ts`, `lib/error-copy.ts` (or the i18n `errors` section), and sibling services in the same feature. Match their location and naming. Create a missing shared file once, from the code below.

## 2. Define the operation contract

| Item | Example |
| --- | --- |
| Operation | `fetchInvoice(id)` |
| Caller | server component, query, or client form (decides service vs action, step 3) |
| Success shape | `Invoice` (domain type), or `null` |
| Input schema | zod schema, type via `z.infer` |
| Error keys | `SharedErrorKey \| 'INVOICE_NOT_FOUND' \| 'INVOICE_FETCH_FAILED'` |
| External system | REST endpoint, DB, vendor SDK |

Present it and wait for confirmation.

## 3. Pick the file

- **Reads** (called by a server component or a query): `services/<entity>.ts`.
- **Mutations called from the client**: `actions/<verb>-<entity>.ts` with `'use server'`. The action validates its input and calls `http` itself. It moves the call into `services/` only when a second caller needs it, or when the response needs a parser with its own tests (a `'use server'` file may export only async functions).
- **Route handlers**: guards at the top, then the call (step 7).

## 4. Shared files — once per repo

```ts
// types/result.ts
export type Result<T, K extends string> =
  | { ok: true; data: T }
  | { ok: false; errorKey: K; meta?: Record<string, string | number> };
```

```ts
// lib/http.ts — every fetch in the app goes through here
import type { Result } from '@/types/result';
import { env } from '@/env';
import { getAuthHeaders } from '@/lib/auth'; // the repo's own auth lookup

export type SharedErrorKey = 'NETWORK' | 'TIMEOUT' | 'UNAUTHORIZED' | 'RATE_LIMITED';

interface RequestOptions<K extends string> {
  fallbackKey: K; // any failed status the shared keys don't cover
  notFoundKey?: K; // a 404 gets this reason key instead of the catch-all
  signal?: AbortSignal; // caller cancellation, combined with the timeout
  keepalive?: boolean; // a write that may fire during tab close; never a read
}

const TIMEOUT_MS = 10_000;

export const http = {
  get: <K extends string>(path: string, options: RequestOptions<K>) => request(path, { method: 'GET' }, options),
  post: <K extends string>(path: string, body: unknown, options: RequestOptions<K>) =>
    request(path, { method: 'POST', body: JSON.stringify(body) }, options),
  patch: <K extends string>(path: string, body: unknown, options: RequestOptions<K>) =>
    request(path, { method: 'PATCH', body: JSON.stringify(body) }, options),
  delete: <K extends string>(path: string, options: RequestOptions<K>) => request(path, { method: 'DELETE' }, options),
};

// data is unknown on purpose: the service parses it with its schema
const request = async <K extends string>(
  path: string,
  init: RequestInit,
  options: RequestOptions<K>,
): Promise<Result<unknown, SharedErrorKey | K>> => {
  const timeout = AbortSignal.timeout(TIMEOUT_MS);
  try {
    const response = await fetch(`${env.API_BASE_URL}${path}`, {
      ...init,
      keepalive: options.keepalive,
      headers: { 'Content-Type': 'application/json', ...(await getAuthHeaders()) },
      signal: options.signal ? AbortSignal.any([options.signal, timeout]) : timeout,
    });
    if (!response.ok) return { ok: false, errorKey: errorKeyFromStatus(response.status, options) };
    if (response.status === 204) return { ok: true, data: null };
    return { ok: true, data: await response.json() };
  } catch (error) {
    console.error(`Error in http ${init.method} ${path}::`, error);
    return { ok: false, errorKey: errorKeyFromException(error) };
  }
};

const errorKeyFromStatus = <K extends string>(status: number, options: RequestOptions<K>): SharedErrorKey | K => {
  if (status === 401) return 'UNAUTHORIZED';
  if (status === 404 && options.notFoundKey) return options.notFoundKey;
  if (status === 429) return 'RATE_LIMITED';
  return options.fallbackKey;
};

// exported for services that wrap a non-HTTP SDK.
// "Took too long, try again" and "check your connection" are different instructions to the user.
export const errorKeyFromException = (error: unknown): SharedErrorKey =>
  error instanceof DOMException && error.name === 'TimeoutError' ? 'TIMEOUT' : 'NETWORK';
```

- A second backend with different auth is a second file (`lib/billing-http.ts`), copied from this one. Two copies of 60 readable lines beat a configurable factory.
- No automatic retries. If one is ever needed: idempotent methods only, with a max attempt count.
- JSON only. Blob, CSV, and streaming responses get their own function.

## 5. Write the service

```ts
// services/invoice.ts — everything about invoice I/O, top to bottom
import { z } from 'zod';
import { http, type SharedErrorKey } from '@/lib/http';
import type { Result } from '@/types/result';

export type InvoiceErrorKey = SharedErrorKey | 'INVOICE_NOT_FOUND' | 'INVOICE_FETCH_FAILED' | 'INVOICE_RESPONSE_INVALID';

export type InvoiceStatus = 'paid' | 'open';

export interface Invoice {
  id: string;
  amount: number;
  issuedAt: Date;
  status: InvoiceStatus;
}

export const fetchInvoice = async (id: string): Promise<Result<Invoice, InvoiceErrorKey>> => {
  const res = await http.get(`/invoices/${id}`, {
    fallbackKey: 'INVOICE_FETCH_FAILED',
    notFoundKey: 'INVOICE_NOT_FOUND',
  });
  if (!res.ok) return res;

  const dto = invoiceDtoSchema.safeParse(res.data);
  if (!dto.success) {
    console.error('Error in fetchInvoice::', dto.error);
    return { ok: false, errorKey: 'INVOICE_RESPONSE_INVALID' };
  }
  return { ok: true, data: parseInvoice(dto.data) };
};

// the vendor's shape — it never leaves this file
const invoiceDtoSchema = z.object({
  invoice_id: z.string(),
  amount_cents: z.number(),
  issued_at: z.string(),
  state: z.enum(['PAID', 'OPEN']),
});

type InvoiceDto = z.infer<typeof invoiceDtoSchema>;

export const parseInvoice = (dto: InvoiceDto): Invoice => ({
  id: dto.invoice_id,
  amount: dto.amount_cents / 100,
  issuedAt: new Date(dto.issued_at),
  status: dto.state === 'PAID' ? 'paid' : 'open',
});
```

- **The schema is the DTO type.** No hand-written DTO interface beside it.
- **Parse once.** The parser is the only place a wire value becomes a domain value (date strings to `Date`, cents to `number`, vendor enums to a union). Nothing downstream re-parses; an `Invalid Date` at a read site means fix the parser. Display formatting is the view's job (`pxkit-conventions` → `state.md`).
- **Name parsers for what they do**: `parseInvoice` inbound, `formatInvoiceRow` outbound. Plain functions, no mapper classes.
- **A service wrapping a non-HTTP SDK** keeps its own `try/catch`, logs with `console.error('Error in <fn>::', error)`, and returns `errorKeyFromException(error)`.

## 6. Error keys

- An error key is a SCREAMING_SNAKE string literal. Each operation declares its union in the file that produces it (`InvoiceErrorKey` above). The transport keys (`SharedErrorKey`) live in `lib/http.ts`.
- **Name the reason when you know it** (`INVOICE_NOT_FOUND` on a 404). The operation catch-all (`INVOICE_FETCH_FAILED`) is only for unknown reasons. No `_FAILED` suffix on a reason key.
- Keys are a wire contract: reorganizing translation files never renames a key.
- `meta` carries interpolation values only (`{ maxSizeMb: 10 }`), never messages or vendor payloads.
- **Two kinds of failure, two channels.** A programmer error (a missing required argument, a broken invariant) throws with a developer-facing message. An expected failure returns `{ ok: false, errorKey }`. Keys are only for the second kind.
- **Copy resolves from the key at render time**, in one place. i18n apps use a flat `errors` section: ``t(`errors.${errorKey}`, meta)``. Single-language apps use one exhaustive record, so a key without copy is a compile error:

```ts
// lib/error-copy.ts — the one place keys become words
import type { InvoiceErrorKey } from '@/features/invoices/services/invoice';
import type { ContactErrorKey } from '@/features/contact/actions/submit-contact';

export type ErrorKey = InvoiceErrorKey | ContactErrorKey;

export const errorCopy: Record<ErrorKey, string> = {
  NETWORK: 'Connection failed. Check your internet and try again.',
  TIMEOUT: 'That took too long. Try again.',
  UNAUTHORIZED: 'You need to sign in to do that.',
  RATE_LIMITED: 'Too many requests. Wait a moment and retry.',
  INVOICE_NOT_FOUND: 'That invoice does not exist.',
  INVOICE_FETCH_FAILED: 'Could not load the invoice.',
  INVOICE_RESPONSE_INVALID: 'Could not load the invoice.',
  CONTACT_INVALID: 'Check the highlighted fields.',
  CONTACT_SUBMIT_FAILED: 'Could not send your message.',
};
```

- No user-facing error string anywhere else.

## 7. Server actions and route handlers

```ts
// actions/submit-contact.ts
'use server';

import { http, type SharedErrorKey } from '@/lib/http';
import type { Result } from '@/types/result';
import { contactSchema, type ContactValues } from '@/features/contact/lib/contact-schema';

export type ContactErrorKey = SharedErrorKey | 'CONTACT_INVALID' | 'CONTACT_SUBMIT_FAILED';

export const submitContact = async (values: ContactValues): Promise<Result<null, ContactErrorKey>> => {
  const parsed = contactSchema.safeParse(values); // the client is untrusted, even our own form
  if (!parsed.success) return { ok: false, errorKey: 'CONTACT_INVALID' };

  const res = await http.post('/contact', parsed.data, { fallbackKey: 'CONTACT_SUBMIT_FAILED' });
  return res.ok ? { ok: true, data: null } : res;
};
```

The schema lives in the feature's `lib/` because the form and the action both import it.

A route handler guards at the top in plain view, then delegates. No `withAuth(handler)` wrapper:

```ts
// app/api/orders/route.ts
export const POST = async (request: Request) => {
  const session = await getSession(request);
  if (!session) return Response.json({ ok: false, errorKey: 'UNAUTHORIZED' }, { status: 401 });

  const body = orderSchema.safeParse(await request.json().catch(() => null));
  if (!body.success) return Response.json({ ok: false, errorKey: 'ORDER_INVALID' }, { status: 400 });

  return Response.json(await createOrder(session.userId, body.data));
};
```

## 8. Env

One typed, zod-validated `env.ts` (`@t3-oss/env-nextjs` or the repo's equivalent). `process.env` appears nowhere else.

## 9. Tests

Parser tests sit next to the service (`services/invoice.test.ts`), fed with real-shaped DTOs. Don't unit-test the `http` wiring.

## 10. Verify

- `rg "fetch\(|AbortSignal|process\.env" <feature>` returns nothing (non-HTTP SDK services excepted).
- `rg "<DtoSchemaName>|<DtoTypeName>" <feature>` matches only the service file and its test.
- `rg "throw " <feature>/services <feature>/actions` returns only programmer-error guards.
- Every `Result` from this operation is checked with `if (!res.ok)` before `res.data` is read.
- A reader follows call site → action or service → `http` without a third file.
- The repo's `typecheck`, `lint`, and `test` scripts pass.
