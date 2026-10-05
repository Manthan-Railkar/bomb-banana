# Deploying Bomb Squad with Vercel + Render

This guide walks you through deploying **Bomb Squad** as a split stack:
- **Frontend ([`apps/client`](../apps/client))**: Deployed to **Vercel** as a high-performance React Single Page Application (SPA).
- **Backend ([`apps/server`](../apps/server))**: Deployed to **Render** as a persistent Node.js Web Service (handling Socket.IO WebSocket connections and real-time game timers).

---

## 1. Deploying the Backend to Render

Because Bomb Squad uses persistent WebSocket connections (`Socket.IO`) and real-time interval timer schedulers, the backend needs a long-running Node.js process (Render Web Service).

### Option A: 1-Click Blueprint (Recommended)

1. Log in to [Render Dashboard](https://dashboard.render.com/).
2. Click **New +** > **Blueprint**.
3. Connect your GitHub repository (`bomb-banana`).
4. Render will detect [`render.yaml`](../render.yaml) automatically.
5. Click **Apply**.
6. Render will build and deploy `bomb-squad-server`.
7. Once deployed, copy your Render service URL (e.g., `https://bomb-squad-server.onrender.com`).

---

### Option B: Manual Web Service Setup

If you prefer configuring the Web Service manually on Render:

1. Click **New +** > **Web Service**.
2. Select your repository.
3. Configure the following settings:
   - **Name**: `bomb-squad-server`
   - **Language**: `Node`
   - **Region**: Any (e.g. *Oregon* or *Frankfurt*)
   - **Branch**: `main`
   - **Root Directory**: *(leave blank)*
   - **Build Command**:
     ```bash
     pnpm install --frozen-lockfile && pnpm --filter @bomb-squad/shared build && pnpm --filter @bomb-squad/server build
     ```
   - **Start Command**:
     ```bash
     node apps/server/dist/index.js
     ```
   - **Plan**: `Free` or `Starter`
   - **Health Check Path**: `/health`

4. Add the following **Environment Variables**:

| Variable | Value | Description |
|---|---|---|
| `NODE_VERSION` | `22` | Node runtime version |
| `REDIS_URL` | `memory` | In-memory game store (or your Upstash/Render Redis URL) |
| `DATABASE_URL` | `memory` | No-op archive (or your PostgreSQL connection string) |
| `LIVEKIT_URL` | `ws://localhost:7880` | Browser LiveKit URL (or LiveKit Cloud URL) |
| `LIVEKIT_SERVER_URL` | `http://localhost:7880` | Server-to-server LiveKit URL |
| `LIVEKIT_API_KEY` | `devkey` | LiveKit API Key |
| `LIVEKIT_API_SECRET` | `devsecret` | LiveKit API Secret |
| `TURN_SECRET` | `changeme` | TURN HMAC secret |
| `TURN_TTL` | `86400` | TURN credential TTL |

> [!TIP]
> **Zero-Database Mode (`REDIS_URL=memory`)**:
> With `REDIS_URL=memory` and `DATABASE_URL=memory`, the server runs entirely in-memory with zero external database dependencies. All 11 puzzle modules, timers, multi-round relays, and strikes work out of the box on Render's free tier!
>
> If you want persistent session archiving or multi-instance clustering, you can create a free Redis instance on [Upstash](https://upstash.com/) and a free PostgreSQL database on [Neon](https://neon.tech/) or [Supabase](https://supabase.com/) and paste their connection URLs.

---

## 2. Deploying the Frontend to Vercel

The frontend is a Vite + React SPA that talks to your Render backend via WebSockets.

### Step-by-Step Vercel Setup

1. Log in to [Vercel Dashboard](https://vercel.com/) and click **Add New...** > **Project**.
2. Import your GitHub repository (`bomb-banana`).
3. In the project configuration:
   - **Framework Preset**: `Vite`
   - **Root Directory**: Click *Edit* and select **`apps/client`** (or leave `.` with root `vercel.json`).
   - If using `apps/client` as Root Directory:
     - Enable **Include source files outside of the Root Directory in the Build Step**.
     - **Build Command**: `pnpm --filter @bomb-squad/shared build && pnpm --filter @bomb-squad/client build`
     - **Output Directory**: `dist`
     - **Install Command**: `pnpm install`
4. Expand the **Environment Variables** section and add:

| Name | Value |
|---|---|
| `VITE_SERVER_URL` | `https://your-render-service.onrender.com` *(use your actual Render backend URL)* |

5. Click **Deploy**.

> [!NOTE]
> Vercel SPA routing is already configured in [`apps/client/vercel.json`](../apps/client/vercel.json) and [`vercel.json`](../vercel.json) to rewrite all routes to `/index.html`, ensuring URLs like `/session/XXXX` or `/sandbox` work on refresh without 404s.

---

## 3. WebRTC Voice (LiveKit) in Production

Bomb Squad features an independent voice subsystem:
- The game is **100% playable** even if voice is disabled or unconfigured (the audio subsystem gracefully falls back).
- For voice communication in production without managing complex WebRTC/TURN infrastructure on Docker:
  1. Sign up for a free project at [LiveKit Cloud](https://cloud.livekit.io/).
  2. Create a project and get your WebSocket URL (`wss://<your-project>.livekit.cloud`), API Key, and Secret.
  3. In Render environment variables, update:
     - `LIVEKIT_URL`: `wss://<your-project>.livekit.cloud`
     - `LIVEKIT_SERVER_URL`: `https://<your-project>.livekit.cloud`
     - `LIVEKIT_API_KEY`: `<your-key>`
     - `LIVEKIT_API_SECRET`: `<your-secret>`

---

## 4. Verification Checklist

- [ ] Render backend `/health` returns `200` with `{"status":"ok"}`.
- [ ] Vercel frontend loads the Landing page without any console network errors.
- [ ] Clicking "Host Session" on the Vercel frontend creates a game room and generates a 4-character join code.
- [ ] Experts and Defusers can connect from separate browsers/devices using the join code.
