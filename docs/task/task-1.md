# Task List — v1

## Task #1 — Project Setup: FE + BE + DB Scaffolding

Initialize the full project structure.

**Frontend (Vite + React + TypeScript + shadcn/ui):**
- `pnpm create vite frontend -- --template react-ts`
- Install and configure shadcn/ui
- Setup path aliases (`@/components`, `@/lib`, etc.)
- Configure ESLint + Prettier

**Backend (Node.js + TypeScript):**
- Init Node.js project with TypeScript
- Choose framework: Express or Fastify
- Setup project structure: `src/routes`, `src/services`, `src/middleware`, `src/db`
- Configure ESLint + Prettier (shared config with FE if monorepo)

**Database:**
- Install MySQL locally (or Docker for local dev)
- Choose and configure migration tool (Prisma or Knex)
- Create `.env.example` with all required env vars:
  - `DATABASE_URL`
  - `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
  - `JWT_SECRET`
  - `PAT_ENCRYPTION_KEY`
  - `PORT`

**Deliverable:** Both FE and BE run locally; BE connects to MySQL; migration tool runs clean.

---

## Task #2 — Design and Migrate MySQL Database Schema

Define and create all core tables.

**Tables:**

```sql
users (
  id           BIGINT PK AUTO_INCREMENT,
  email        VARCHAR(255) UNIQUE NOT NULL,
  google_id    VARCHAR(255) UNIQUE NOT NULL,
  name         VARCHAR(255),
  avatar_url   TEXT,
  slack_user_id VARCHAR(255),
  created_at   DATETIME DEFAULT NOW()
)

workspaces (
  id                    BIGINT PK AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  github_repo_url       TEXT NOT NULL,
  github_pat_encrypted  TEXT NOT NULL,
  slack_webhook_url     TEXT,
  created_at            DATETIME DEFAULT NOW()
)

workspace_members (
  workspace_id  BIGINT FK → workspaces.id,
  user_id       BIGINT FK → users.id,
  role          ENUM('owner', 'member') NOT NULL,
  joined_at     DATETIME DEFAULT NOW(),
  PRIMARY KEY (workspace_id, user_id)
)

comments (
  id            BIGINT PK AUTO_INCREMENT,
  workspace_id  BIGINT FK → workspaces.id,
  file_path     TEXT NOT NULL,
  scope         ENUM('file', 'line') NOT NULL,
  anchor_text   TEXT,
  anchor_start  INT,
  anchor_end    INT,
  body          TEXT NOT NULL,
  author_id     BIGINT FK → users.id,
  resolved      BOOLEAN DEFAULT FALSE,
  created_at    DATETIME DEFAULT NOW(),
  updated_at    DATETIME DEFAULT NOW()
)

comment_tags (
  comment_id  BIGINT FK → comments.id,
  user_id     BIGINT FK → users.id,
  PRIMARY KEY (comment_id, user_id)
)
```

**Deliverable:** All migrations run cleanly on fresh MySQL instance.

---

## Task #3 — Implement Google OAuth Login + JWT Session

**Backend:**
- Register Google OAuth2 app (Google Cloud Console) — get `CLIENT_ID` and `CLIENT_SECRET`
- Implement OAuth2 flow:
  - `GET /auth/google` → redirect to Google consent screen
  - `GET /auth/google/callback` → exchange code for tokens, get user profile
- On callback:
  - Look up user by `google_id`; if not found → create new user record (first login auto-creates account)
  - Issue JWT containing `{ userId, email }` with expiry (e.g. 7 days)
- Return JWT to frontend (cookie or JSON response)
- `GET /auth/me` → validate JWT, return current user profile
- `POST /auth/logout` → invalidate session (clear cookie / client drops token)
- Auth middleware: validate JWT on all protected routes; return 401 if invalid/expired

**Frontend:**
- Google Sign-In button on login page
- Redirect to `/auth/google`; handle callback and store JWT
- Protected route wrapper: redirect to login if no valid JWT
- Show current user avatar + name in header

**Deliverable:** User can log in with Google, account is created on first login, JWT is issued and validated on subsequent requests.

---

## Task #4 — Workspace CRUD + GitHub Repo Connection + PAT Storage

**Backend:**
- `POST /workspaces` — create workspace; creator becomes owner; encrypt PAT before storing
  - Encryption: AES-256-GCM using `PAT_ENCRYPTION_KEY` from env
- `GET /workspaces` — list workspaces the current user belongs to
- `GET /workspaces/:id` — get workspace detail (do NOT return raw PAT)
- `PATCH /workspaces/:id` — update name, repo URL, PAT, Slack webhook (owner only)
- `DELETE /workspaces/:id` — soft or hard delete (owner only)
- Helper: `decryptPAT(workspace)` — decrypt for internal use in GitHub API calls only

**Frontend:**
- Workspace list / dashboard page
- "Create workspace" dialog: name, GitHub repo URL, GitHub PAT input (masked)
- Workspace settings page (owner only): edit name, update PAT, set Slack webhook URL
- Workspace switcher in sidebar

**Deliverable:** Owner can create workspace, connect GitHub repo, store PAT; PAT is encrypted at rest and never exposed in API responses.

---

## Task #5 — Invite Existing Users to Workspace (Owner Only)

**Backend:**
- `GET /users/search?email=...` — search users by email; return partial matches; only accessible to workspace owners
- `POST /workspaces/:id/members` — add a user to workspace by userId (user must exist in system); owner only
- `DELETE /workspaces/:id/members/:userId` — remove member (owner only; cannot remove self if last owner)
- `GET /workspaces/:id/members` — list all workspace members with roles

**Frontend:**
- Members section in workspace settings
- "Invite user" dialog: type email → live search → select user → add
- Member list: show name, email, role, "Remove" button (owner only)
- Error state: "User not found in system — they need to log in first"

**Deliverable:** Owner can find and add any registered user to the workspace by email.

---

## Task #6 — GitHub API Integration Layer

Backend service module that all file operations go through. Uses workspace owner's decrypted PAT.

**Functions to implement:**
```typescript
getFileTree(repoUrl, pat): Promise<TreeNode[]>           // full folder + file tree
getFileContent(repoUrl, pat, path): Promise<FileContent> // raw content + sha
getFileHistory(repoUrl, pat, path): Promise<Commit[]>   // commits touching file
createFile(repoUrl, pat, path, content, message): Promise<void>
updateFile(repoUrl, pat, path, content, sha, message): Promise<void>
deleteFile(repoUrl, pat, path, sha, message): Promise<void>
```

**Error handling:**
- 401 Unauthorized → PAT invalid or expired → return structured error `{ code: 'PAT_INVALID' }`
- 404 Not Found → file/repo missing → `{ code: 'NOT_FOUND' }`
- 409 Conflict → sha mismatch / merge conflict → `{ code: 'CONFLICT' }`
- 403 Forbidden → no push access → `{ code: 'FORBIDDEN' }`
- All errors surfaced to frontend as clear messages

**Note:** Parse GitHub repo URL to extract `owner/repo` slug for API calls.

**Deliverable:** All GitHub operations work correctly against a real test repo.

---

## Task #7 — Folder Hierarchy View UI

**Frontend:**
- Sidebar tree component: renders `TreeNode[]` from GitHub API
- Folder nodes: expandable/collapsible with chevron icon
- File nodes: click to open file view
- Icons: folder icon (shadcn or lucide), markdown file icon
- "Sync" button at top of tree: calls `GET /workspaces/:id/files/tree` and refreshes tree
- Loading state during fetch
- Error state: show message if PAT invalid or repo inaccessible

**Backend:**
- `GET /workspaces/:id/files/tree` — call `getFileTree`, return as JSON tree structure
- Cache file tree in memory or DB (optional for v1; or re-fetch on every sync button click)

**Deliverable:** User sees full GitHub repo folder structure in sidebar; clicking sync re-fetches from GitHub.

---

## Task #8 — Single File View with Split-Pane Markdown Editor

**Frontend:**
- Split-pane layout: left = CodeMirror or plain `<textarea>` for raw markdown; right = rendered HTML preview (use `marked` or `remark`)
- Load file content on open via `GET /workspaces/:id/files/content?path=...`
- Track dirty state (unsaved changes)
- Commit button: opens dialog asking for commit message → calls `PUT /workspaces/:id/files` → success clears dirty state
- Show inline error if commit fails (e.g. conflict, PAT invalid)
- Sync preview in real time as user types (debounced)

**Backend:**
- `GET /workspaces/:id/files/content?path=...` — return file content + sha
- `PUT /workspaces/:id/files` — body: `{ path, content, sha, message }` → calls `updateFile` GitHub API

**Deliverable:** User can open a markdown file, edit in browser, and commit + push to GitHub with a message.

---

## Task #9 — File Create and Delete from Browser

**Frontend:**
- "New file" button in folder tree: dialog to enter filename (with `.md` enforced) + parent folder
- "Delete file" option on file context menu: confirmation dialog → proceed
- After create/delete: refresh folder tree

**Backend:**
- `POST /workspaces/:id/files` — body: `{ path, content: '', message }` → calls `createFile` GitHub API
- `DELETE /workspaces/:id/files` — body: `{ path, sha, message }` → calls `deleteFile` GitHub API (hard delete; commit attributed to owner PAT)

**Deliverable:** User can create new `.md` files and delete existing files; both changes are committed and pushed to GitHub.

---

## Task #10 — Git History View Per File

**Frontend:**
- History panel/tab in file view: shows list of commits that touched the current file
- Each commit row: message, author name, date, short SHA (link out to GitHub)
- Fetched on demand when panel is opened (not preloaded)
- Loading and empty states

**Backend:**
- `GET /workspaces/:id/files/history?path=...` — calls `getFileHistory` GitHub API; return list of commits

**Deliverable:** User can open git history for any file and see the full commit log from GitHub.

---

## Task #11 — Comment System: Per-File and Per-Line (Text Selection Anchor)

**Backend:**
- `POST /comments` — body: `{ workspaceId, filePath, scope, anchorText, anchorStart, anchorEnd, body }`
- `GET /comments?workspaceId=...&filePath=...` — return all comments for a file (including resolved)
- `PATCH /comments/:id` — edit comment body (author only)
- `DELETE /comments/:id` — delete comment (author only)

**Frontend:**
- **Per-file comments:** comment panel in file view showing file-level thread
- **Per-line comments:** user selects text in rendered preview → popover appears with "Add comment" button → opens inline comment form
  - Store `anchorText` + selection `start`/`end` character offsets
  - Highlight anchored text in preview when comment is visible
- Comment form: textarea + submit
- Comment thread: show author avatar, name, timestamp, body

**Deliverable:** Users can leave comments on a whole file or on selected text within the file.

---

## Task #12 — Comment Resolution

**Backend:**
- `PATCH /comments/:id/resolve` — set `resolved = true` (any workspace member)
- `PATCH /comments/:id/unresolve` — set `resolved = false`

**Frontend:**
- "Resolve" / "Unresolve" button on each comment thread
- Resolved comments show a green "Resolved" badge
- Filter toggle: "Show resolved" / "Hide resolved" (default: show all)
- Resolved comments remain fully visible — not collapsed or hidden

**Deliverable:** Any member can resolve or reopen a comment; resolved comments stay visible.

---

## Task #13 — User Tagging (@mention) in Comments

**Backend:**
- Parse comment body for `@email` or `@name` patterns on save
- Look up matched users in workspace members
- Insert rows into `comment_tags (comment_id, user_id)`
- `GET /workspaces/:id/members` used by FE for autocomplete

**Frontend:**
- Comment editor: detect `@` keystroke → show dropdown of workspace members (filtered by typed text)
- Members shown by name + email
- Selected member inserted as `@email` token in comment body
- After save: tagged users are highlighted in rendered comment

**Deliverable:** Users can type `@` in a comment to tag a workspace member; tags are stored and rendered correctly.

---

## Task #14 — WebSocket Server for Real-Time Live Comment Feed

**Backend:**
- Setup WebSocket server (use `ws` library on Node.js)
- Authenticate connection: client sends JWT in connection query or first message
- Room concept: `workspace_id:file_path` — client subscribes when a file is opened
- Broadcast events to room on:
  - New comment created
  - Comment edited
  - Comment resolved/unresolved
  - Comment deleted
- Event payload: full comment object

**Frontend:**
- Connect WebSocket on file open; disconnect on file close / navigation
- On receiving event: update comment list in real time (add, update, or remove)
- No reconnect replay: on reconnect, do a fresh `GET /comments` to resync state

**Deliverable:** Comments appear in real time for all users viewing the same file without page refresh.

---

## Task #15 — Slack Webhook Notification on @mention

**Backend:**
- Triggered after `comment_tags` rows are saved (Task #13)
- For each tagged user:
  1. Check if workspace has a `slack_webhook_url` → if not, skip silently
  2. Build Slack message: file name, comment excerpt, author name
  3. If tagged user has `slack_user_id` → include `<@slack_user_id>` in message
  4. If tagged user has no `slack_user_id` → include their email as plain text
  5. POST to `slack_webhook_url` with message payload

**User settings:**
- `PATCH /users/me` — allow user to update their `slack_user_id`
- Frontend: "Profile settings" page with Slack user ID field

**Deliverable:** Tagged users receive a Slack message in the workspace's configured channel.

---

## Task #16 — File Content Search (Grep on Disk)

**Backend:**
- Files must exist on disk (either written during sync or on initial workspace setup)
- On manual sync: write all repo files to local disk path `./storage/{workspaceId}/`
- `POST /workspaces/:id/search` — body: `{ query: string }`
  - Run `grep -r --include="*.md" -n "<query>" ./storage/{workspaceId}/`
  - Parse grep output into `[{ filePath, lineNumber, lineContent }]`
  - Return results to frontend

**Frontend:**
- Search bar in workspace sidebar or top navigation
- On submit: call search API
- Results list: file path (clickable → open file), matched line preview

**Note:** Comment search is deferred to v2.

**Deliverable:** User can search across all markdown file contents in the workspace.

---

## Task #17 — Bare Metal Self-Hosted Deployment Setup

**Server setup:**
- Node.js (LTS) installed via nvm
- MySQL installed and configured; run migrations on deploy
- Nginx as reverse proxy: serve Vite static build + proxy `/api` to Node.js backend

**Process management:**
- PM2 to run and auto-restart Node.js backend
- `pm2 startup` to survive server reboots

**Environment:**
- `/etc/environment` or `.env` file with all secrets:
  - `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
  - `JWT_SECRET`
  - `PAT_ENCRYPTION_KEY` (32-byte random key)
  - `DATABASE_URL`
  - `PORT`

**Build & deploy script:**
```bash
# Frontend
cd frontend && pnpm build      # output to dist/
# Copy dist/ to Nginx web root

# Backend
cd backend && pnpm build       # compile TS → JS
pm2 restart mdsot-backend      # or pm2 start
npx knex migrate:latest        # run DB migrations
```

**Deliverable:** Application runs on a bare metal Linux server, survives reboots, accessible via domain or IP.
