# The /api/ticket timeout: a Compose service-name mismatch

**Date:** 2026-11-04  
**Repo:** distributed-ticket-service  
**Stack:** React (Vite), nginx, Node.js, Apache Kafka, MongoDB, Docker Compose

## The problem

I ran the repo’s `load-test.js` against the frontend on port 5173 and got:

- Total requests: 1000
- Accepted: 0
- Failed: 1000
- Average latency: ~10s (timeout)

## The evidence

- Backend-only throughput via nginx on port 80: **338 req/s, 81ms avg, 1000/1000 accepted**.
- `POST http://localhost/api/ticket` returned `202` via nginx.
- The same request via the frontend timed out, with Vite logs showing:  
  `Error: getaddrinfo ENOTFOUND gateway-service`

## The root cause

The frontend’s Vite dev server proxied `/api` and `/socket.io` to `gateway-service:3000` — but the Compose service was defined as `gateway`. Inside the Compose bridge network, DNS resolves service names, not image names. `gateway-service` did not exist, so the browser’s request never made it past the proxy.

## The fix

In `frontend/vite.config.js`:

```js
target: 'http://gateway:3000',
```

Or remove the duplication and let the frontend talk to nginx at `http://localhost:80` like the backend does.

## What this changed for me

I stopped trusting the browser as the final source of truth and started verifying each hop: browser → proxy → gateway → Kafka → worker → MongoDB. That staging is also what let me measure backend throughput through nginx without depending on the broken frontend.