# SaverTrack Backend

Backend service for linking real UK bank accounts (Open Banking) to the Saver app.

Separate from the Saver mobile app (React + Capacitor). The mobile app talks to this
backend over HTTPS only. Bank access tokens live here, never on the device.

## Status

Early setup. Full implementation plan and milestones are in [BACKEND_PLAN.md](./BACKEND_PLAN.md).

## Provider

Plaid (Sandbox first). A provider abstraction layer keeps the aggregator swappable.

## Getting started

```bash
npm install
cp .env.example .env   # then fill in real values (never commit .env)
npm run dev
```

## Security

- Secrets live in `.env` (gitignored) only.
- Bank access tokens are stored encrypted at rest.
- No tokens are ever sent to the mobile client.
