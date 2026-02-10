# Epoch Zone UI

Svelte frontend for the Epoch Zone timezone API. Provides timezone lookup and conversion between timezones.

## Prerequisites

- [Node.js](https://nodejs.org/) (18+)

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `VITE_API_KEY` | No | - | API key for the Epoch Zone backend |

## Setup & Run

```bash
# Install dependencies
npm install

# Start dev server (port 5173, proxies /api to the backend)
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Serve production build
npm run start
```

In development, API requests are proxied to the production backend via Vite's dev server proxy. To point to a local backend instead, update the `proxy.target` in `vite.config.js`.

## Deploy

Deployed on [Railway](https://railway.app). Set `VITE_API_KEY` in your Railway service settings. Railway runs `npm run build` then `npm run start` to serve the static files.
