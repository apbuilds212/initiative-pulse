# Deploying Initiative Pulse to Netlify

Deploy the live dashboard at `initiative-pulse-live.netlify.app` so it reads directly from your Notion database. Three steps: create a Notion integration, set one environment variable, re-deploy the zip.

---

## Step 1 — Create a Notion integration

1. Go to **https://www.notion.so/profile/integrations** and click **New integration**.
2. Name it `Initiative Pulse` (or anything you like), select your workspace, leave type as **Internal**.
3. Click **Save**. Copy the **Internal Integration Secret** — it starts with `secret_...`. You'll need it in Step 2.
4. Open your Initiative Pulse database in Notion (`https://app.notion.com/p/23e2491f018a455f96f6067c05399835`).
5. Click **···** (top-right menu) → **Connections** → search for `Initiative Pulse` → **Confirm**.

> The integration must be connected to the database or the API will return 404.

---

## Step 2 — Add the token to Netlify

1. Open **https://app.netlify.com** → your `initiative-pulse-live` site.
2. Go to **Site configuration → Environment variables → Add a variable**.
3. Add:
   - Key: `NOTION_TOKEN`  Value: `secret_...` (your token from Step 1)
4. Save. No other env vars are needed — the database ID is already baked in as a default.

> If you ever need to point to a different database, add `NOTION_DATABASE_ID` with its ID (32-char hex string from the Notion URL).

---

## Step 3 — Deploy the zip

1. Open the `initiative-pulse-netlify.zip` file in this folder.
2. Go to **https://app.netlify.com** → your site → **Deploys** tab.
3. Drag and drop the zip onto the **"Drag and drop your site folder here"** area.
4. Wait ~30 seconds for the deploy to complete.
5. Visit **https://initiative-pulse-live.netlify.app** — it should load your live Notion data.

---

## What's inside the zip

```
initiative-pulse-netlify.zip
├── index.html                        ← the dashboard (calls /.netlify/functions/notion-proxy)
├── netlify.toml                      ← tells Netlify where the functions live
└── netlify/
    └── functions/
        └── notion-proxy.js           ← server-side proxy; your NOTION_TOKEN never hits the browser
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "NOTION_TOKEN environment variable is not set" | Env var missing or not yet deployed | Add `NOTION_TOKEN` in Netlify UI, then re-deploy |
| HTTP 404 from Notion | Integration not connected to the database | Re-do Step 1 → Connections |
| HTTP 401 from Notion | Token copied wrong (extra space, truncated) | Re-paste from notion.so/profile/integrations |
| "No initiatives found" | Database empty or wrong database ID | Check the DB has rows; add `NOTION_DATABASE_ID` env var if using a different DB |
| Page loads but spinner never stops | Function not deployed (missing `netlify.toml` or `netlify/functions/`) | Re-deploy the zip (not just `index.html`) |
