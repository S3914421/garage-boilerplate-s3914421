# Notes feature design

## Scope

Implement the tutorial Notes feature without the optional backend HTTP API. Authenticated users can create notes and receive live updates for notes they own.

## Architecture

- `/notes` is a protected App Router page.
- A Server Action authenticates the request, validates input with Zod, and writes to Firestore.
- A client component subscribes to the authenticated user's notes.
- Firestore rules enforce owner-only access.
- Shared Firestore types, collection helpers, schema documentation, and sidebar navigation stay synchronized.

## Validation

- Title: required, 1-200 characters.
- Body: optional content, up to 10,000 characters.
- Run typecheck, lint, all tests, and the production build.

## Delivery

Work is isolated on `feature/notes`. Firebase rule deployment, remote push, pull request creation, and merge are handled after local verification.
