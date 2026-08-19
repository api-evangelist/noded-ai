# Noded SDK — rules for coding agents

You are helping build a front-end feature that embeds **Noded** customer context (people, accounts,
activity, notes, memory) into an app. Use the `@bigfootai/noded-sdk` package. Follow these rules.

## Setup
- Install: `npm i @bigfootai/noded-sdk oidc-client-ts`.
- The `issuer`, `clientId` and `audience` are **provided by Noded** — the developer requests them from
  the Noded team (getnoded.ai/contact-us). Read them from env/config; do NOT invent or hardcode values.
  If they aren't available yet, leave them as clearly-named placeholders and tell the developer to
  request Noded developer access.
- Create ONE client and reuse it:
  ```ts
  import { Noded } from '@bigfootai/noded-sdk';
  const noded = new Noded({
    auth: { mode: 'oidc', issuer: NODED_ISSUER, clientId: NODED_CLIENT_ID, audience: NODED_AUDIENCE },
  });
  ```
- In a browser app, call `await noded.connect()` once (e.g. behind a "Sign in with Noded" button, or
  on mount). It opens the Noded login and then auto-refreshes the token. Never put an `apiKey` in
  browser code.

## The data model (don't invent fields)
- **People** and **Accounts** are both *tags*. People have emails; accounts have domains.
- **Activity** is a timeline of *blocks*: notes, emails, tasks, messages, meeting transcriptions, events.
- **Memory** is what the graph has learned about a person/account (topic → value).

## Preferred API — use these helpers, not raw GraphQL, when they fit
```ts
await noded.people.search('acme')          // Person[]
await noded.people.get(id)                 // Person | null
await noded.accounts.search('acme.com')   // Account[]
await noded.accounts.get(id)              // Account | null
await noded.activity.forTag(tagId, { limit: 20 })  // ActivityItem[]
await noded.notes.create({ title: 'QBR prep' })    // Note
await noded.notes.list()                   // ActivityItem[]
await noded.memory.forTag(tagId)           // MemoryEntry[]
await noded.search('renewal risk')         // { people, accounts }
```
Every search helper accepts `{ limit, select }`. Use `select` only to widen the GraphQL selection set.

## Types (import from '@bigfootai/noded-sdk')
- `Person { id, name, email?, emailDomain?, avatar?, description?, externalId? }`
- `Account { id, name, domain?, avatar?, description?, externalId? }`
- `ActivityItem { id, type, title?, preview?, dateCreated?, dateUpdated? }`
- `MemoryEntry { id, content }`

## Escape hatch (only when a helper truly doesn't cover it)
```ts
await noded.graphql(`query { recentTags { _id alias } }`, {});
```
The backend is the Noded Apollo GraphQL API (`POST /api/v1/graph`). Inputs use the schema's
`*Input` types (e.g. `SearchTagsInput`, `FindBlocksInput`, `FindTagInput`).

## Error handling
- Wrap calls in try/catch. `NodedError` has `.kind` of `'auth' | 'graphql' | 'network'`.
- On `.kind === 'auth'`, prompt the user to sign in again via `noded.connect()`.

## Do / Don't
- DO scope everything to the signed-in user — the API already enforces their tenant + permissions.
- DO render loading and empty states; searches can return `[]`.
- DON'T hardcode ids; resolve them via `search` first.
- DON'T ship secrets to the browser. DON'T guess GraphQL field names — use the helpers or ask.
