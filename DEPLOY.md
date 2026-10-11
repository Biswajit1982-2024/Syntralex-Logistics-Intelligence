# Deploying Freight Control Tower — push to live

This app is deliberately zero-dependency (Node built-ins only), so there's
no build step and nothing to `npm install`. That makes the first deploy
fast. Render is the path documented here because its free tier includes a
persistent disk (needed for the SQLite database and uploaded files) and
reads `render.yaml` automatically — Railway/Fly.io work too with the
equivalent of steps 2–4 done in their own dashboard instead.

## 1. Push the code to GitHub

```
cd freight-app
git init
git add .
git commit -m "Freight Control Tower — initial deploy"
```
Create an empty repo on GitHub, then:
```
git remote add origin <your-repo-url>
git push -u origin main
```
`.gitignore` already excludes `server/data/`, `server/uploads/`, and `.env`
— your local test data and secrets won't be pushed.

## 2. Create the Render service

1. Go to [render.com](https://render.com) → **New +** → **Blueprint**.
2. Connect the GitHub repo you just pushed. Render reads `render.yaml` and
   provisions:
   - A **web service** running `node server/server.js`, no build step.
   - A **1 GB persistent disk** mounted at `/var/data` (holds the SQLite DB
     and uploaded POD files — without this, a redeploy wipes both).
   - `NODE_ENV=production`, `STORAGE_DIR=/var/data`, and a **random
     generated** `SEED_DEMO_PASSWORD` (so demo accounts don't ship with the
     well-known `Freight@123` password — the login screen's one-click demo
     buttons automatically turn themselves off in production).
3. Click **Apply**. First deploy takes a minute or two.

## 3. Verify it's live

- Open the URL Render gives you (`https://freight-control-tower-xxxx.onrender.com`).
- You'll land on a plain login form (no demo buttons — this is correct in
  production). To get in:
  - Render dashboard → your service → **Shell**, then:
    ```
    node -e "console.log(process.env.SEED_DEMO_PASSWORD)"
    ```
    and log in as any seeded email (e.g. `ravi.kumar@company.example`) with
    that password. Or set `SEED_DEMO_PASSWORD` yourself in the dashboard's
    **Environment** tab before the first deploy if you'd rather pick it.
- `GET /api/health` should return `{"ok":true,"data":{"status":"ok",...}}` —
  this is what Render's own health check polls.

## 4. What's still a pilot, not a finished production system

Everything above gets you a real public HTTPS URL with working auth,
sessions, and persistent data — enough for a pilot with a real team. Before
wider rollout, work through the rest of the production checklist in
`README.md` → **Going further**:

- **Database**: SQLite is single-process. `server/postgres-schema.sql` is
  already written and validated — swap it in once you can `npm install pg`
  (Render's build environment has full npm access, unlike this sandbox).
- **File storage**: move `Uploaded_File.Stored_Path` to S3/Azure Blob —
  Render's disk is reliable but not a CDN.
- **Identity**: swap `modules/auth.js`'s local login for your real SSO
  (Entra ID etc.) if this needs to match company identity requirements.
- **Custom domain**: Render → Settings → **Custom Domain**.

## Local development

```
node server/server.js
```
No env vars needed — defaults to `server/data/`, `server/uploads/`,
password `Freight@123`, plain HTTP on port 4000. See `.env.example` for
every variable the app reads.
