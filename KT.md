# Knowledge Transfer — Nirdesh Platform

**Prepared by:** [Your Name]
**Date:** April 2026
**Session type:** Handover KT
**Audience:** Incoming developer(s) taking ownership of this project

---

## 1. What is Nirdesh?

Nirdesh is an internal project management platform for iCloudLogic. It lets teams create sprints, tasks, and projects — and now includes a real-time forums/messaging module for team communication.

The platform has two frontend apps and one backend:

```
nirdesh_frontend        ← Main host app (sprints, tasks, projects, file upload)
nirdesh_micro_frontend  ← Forums module (channels, messaging, chatbot, real-time)
nirdesh_backend         ← Spring Boot REST API (port 8080)
Keycloak                ← Authentication server (port 8180)
```

---

## 2. What was built (my contribution)

### Forums Micro-Frontend — built from scratch

The entire forums module is new. Before this, there was no messaging or collaboration feature. I built:

- **Sidebar** with channel list, user profile, home navigation
- **Forum list** with search, filter by level (org/subscription/private), sort
- **Forum detail** with Slack-style threading — click Reply on any message to open a thread panel
- **Real-time messaging** via SSE (Server-Sent Events)
- **AI Chatbot** floating assistant wired to `/api/chatbot/query`
- **Dark/light theme** with gradient styling, persisted to localStorage
- **Microsoft Entra ID login** — users can sign in with their Office 365 accounts via Keycloak

### Host App — improvements

- Dark theme toggle (🌙/☀️) added to all pages
- Smooth transition between host app and forums
- Login page redesigned with gradient branding
- Loading/error states improved

### Architecture decisions

- **Micro-frontend via Module Federation** — forums loads at runtime from port 3001, zero coupling to host app
- **Tailwind CSS** — migrated forums from MUI to pure Tailwind (build time dropped from 20s to 2s, bundle size reduced by ~6MB)
- **SSE over WebSockets** — simpler, works through all proxies, built-in reconnection
- **Clean 4-layer architecture** — Presentation → Application → Domain → Repository
- **Mock data fallback** — app works fully without a backend, falls back to sample data automatically

---

## 3. Current state

### What works today

| Feature | Status |
|---------|--------|
| Login via Keycloak (username/password) | ✅ Working |
| Login via Microsoft Office 365 | ✅ Working (configured) |
| Forum list, create forum, filter/search | ✅ Working (mock data) |
| Forum messaging with threading | ✅ Working (mock data) |
| Dark/light theme | ✅ Working |
| AI Chatbot (mock responses) | ✅ Working (mock) |
| Sprint/Task/Project creation | ✅ Working (real backend) |
| Real-time SSE updates | ⏳ Needs backend channels API |

### What the backend still needs to build

The forums backend endpoints don't exist yet. The frontend falls back to mock data until they're built:

| Endpoint | Priority |
|----------|----------|
| `GET /api/v1/channels` | #1 |
| `POST /api/v1/channels` | #2 |
| `GET /api/v1/channels/:id/messages` | #3 |
| `POST /api/v1/channels/:id/messages` | #4 |
| `GET /api/events/stream` (SSE) | #5 |
| `POST /api/chatbot/query` | #6 |

Full spec in `docs/backend-integration-spec.md`.

---

## 4. How to run the project

### Prerequisites
- Node.js 18+
- Docker Desktop
- Java 17+ (for backend)

### Start everything

```bash
# 1. Start Keycloak
docker-compose up -d

# 2. Wait ~30 seconds, then setup realm
powershell -ExecutionPolicy Bypass -File scripts/setup-keycloak.ps1

# 3. Build and serve forums micro-frontend (Terminal 1)
cd nirdesh_micro_frontend
npm install
npm run build
npm run preview        # serves on :3001

# 4. Start host app (Terminal 2)
cd nirdesh_frontend
npm install
npm run dev            # serves on :5173

# 5. Start backend (Terminal 3)
cd nirdesh_backend
./mvnw spring-boot:run  # serves on :8080
```

Open `http://localhost:5173` — login with `alice / password123`

### Test users

| User | Password | Role |
|------|----------|------|
| alice | password123 | Developer |
| bob | password123 | Manager |
| charlie | password123 | Designer |

---

## 5. Key files to know

```
nirdesh_micro_frontend/src/
├── components/forums/
│   ├── ForumShell.jsx          ← Entry point (exposed via Module Federation)
│   ├── Sidebar/Sidebar.jsx     ← Channel list + user profile
│   ├── ForumList/              ← Forum cards, filter, create dialog
│   ├── ForumDetail/            ← Messages + thread panel
│   └── Chatbot/Chatbot.jsx     ← AI assistant
├── contexts/
│   ├── ForumContext.jsx        ← All forum state + actions (start here)
│   └── ThemeContext.jsx        ← Dark/light mode
├── services/
│   ├── forumAPI.js             ← All API calls (repository pattern)
│   ├── apiClient.js            ← Axios + auth interceptor
│   └── sseService.js           ← Real-time SSE connection
└── data/mockForums.js          ← Sample data (used when backend is down)

nirdesh_frontend/src/
├── App.jsx                     ← Routes + auth gate + theme provider
├── contexts/ThemeContext.jsx   ← Host app dark mode
└── components/
    ├── VerticalMenuBar.jsx     ← Navigation drawer
    └── RemoteForumShell.jsx    ← Loads forums module via Module Federation
```

---

## 6. Architecture in 2 minutes

```
Browser
  ├── Host App (:5173)
  │     └── loads forums at runtime via Module Federation
  └── Forums Module (:3001)
        ├── Sidebar (channels, user)
        ├── ForumList (browse channels)
        ├── ForumDetail (messages + threads)
        └── Chatbot (AI assistant)

Both apps share:
  - React (singleton)
  - react-oidc-context (one login, works everywhere)

Auth flow:
  User → Keycloak login page → JWT token → stored in localStorage
  Every API call → Bearer token in Authorization header
  Microsoft login → Keycloak brokers Azure AD → same JWT flow

Real-time:
  Browser ←── SSE stream ──── Backend (pushes new messages)
  Browser ────── REST POST ──► Backend (sends messages)
  No database needed for SSE — it's in-memory on the backend
```

---

## 7. Known issues and gotchas

| Issue | Status | Notes |
|-------|--------|-------|
| Forums API returns 404 | Expected | Backend channels endpoints not built yet |
| `window.location.reload()` loop | Fixed | Was in `apiClient.js` handleAuthError — removed |
| Micro-frontend needs rebuild after changes | Known | `npm run build && npm run preview` — dev mode doesn't work with Module Federation |
| Port 3001 conflict | Occasional | `netstat -ano \| findstr :3001` then `taskkill /PID <pid> /F` |
| Microsoft login "Account already exists" | Normal | Click "Add to existing account" on first Microsoft login |

---

## 8. Documentation index

All docs are in the `docs/` folder:

| Doc | What it covers |
|-----|---------------|
| `NIRDESH-COMPLETE-GUIDE.md` | Master reference — start here |
| `backend-integration-spec.md` | Exact API contracts + DB mapping + SSE implementation |
| `clean-architecture.md` | 4-layer architecture with real code examples |
| `microsoft-entra-oauth-setup.md` | Azure AD + Keycloak setup step by step |
| `keycloak-setup.md` | Keycloak realm, users, roles, OIDC flow |
| `sse-communication.md` | SSE vs WebSocket, event types, reconnection |
| `security.md` | Auth, XSS prevention, token management |
| `sync-analysis.md` | DB schema vs frontend field names — what maps to what |
| `backend-api-guide.md` | API reference for backend developer |
| `messaging-app-guide.md` | How to make messaging fully functional |

---

## 9. What I'd do next if I were staying

1. **Build the channels API** — `GET/POST /api/v1/channels` and `GET/POST /api/v1/channels/:id/messages`. The frontend is ready, just needs the backend.
2. **Wire up SSE** — implement `SseService.java` (full code in `docs/backend-integration-spec.md`). Real-time messages will work immediately.
3. **Connect chatbot to AI model** — just implement `POST /api/chatbot/query` returning `{ response, action, confidence }`.
4. **Add member management UI** — the DB has `channels_users` table, the API is documented, just needs a UI component.
5. **Unread count tracking** — add a `user_channel_read` table to track last read message per user per channel.

---

## 10. Questions to ask the backend team

- What's the exact URL structure for channels? Is it `/api/v1/channels` or something else?
- What does the `visibility` table contain? Need to confirm the values map to `organization`/`subscription`/`private`.
- Is there a users table or does the app rely entirely on Keycloak for user data?
- What's the plan for the chatbot — which AI model/provider?

---

Good luck to whoever picks this up. The codebase is well-documented, the architecture is clean, and the mock data fallback means you can develop the UI without a running backend. Reach out if anything is unclear.
