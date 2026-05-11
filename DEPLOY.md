# ST Crane Hire — Deployment Guide

This is a standard **Vite + React + TypeScript PWA** with a **Supabase** backend (database, auth, storage, edge functions, RLS). Nothing in this repo is locked to Lovable — you can host it on any web server and any Supabase project.

---

## 1. What you're deploying

| Piece | What it is | Where it runs |
|---|---|---|
| Frontend | Static `dist/` folder produced by `npm run build` | Any static host (Nginx, Apache, Caddy, Cloudflare Pages, S3+CloudFront, Vercel, Netlify, Docker) |
| Backend | Supabase project (Postgres + Auth + Storage + Edge Functions) | supabase.com OR self-hosted Supabase |
| PWA | Service worker (`public/sw.js`) for offline use | Served as static files alongside the frontend |

---

## 2. Backend setup (one-time)

### Option A — Hosted Supabase (recommended)
1. Create a project at https://supabase.com.
2. From **Project Settings → API**, copy:
   - `Project URL` → becomes `VITE_SUPABASE_URL`
   - `anon` key → becomes `VITE_SUPABASE_PUBLISHABLE_KEY`
   - `Project ID` (the `xxxx` from `xxxx.supabase.co`) → becomes `VITE_SUPABASE_PROJECT_ID`
3. Apply the schema:
   ```bash
   npx supabase link --project-ref <your-project-ref>
   npx supabase db push
   ```
   This runs every file in `supabase/migrations/` against your new project.
4. Deploy the edge functions:
   ```bash
   npx supabase functions deploy create-admin
   npx supabase functions deploy revoke-admin
   ```
5. In the Supabase dashboard, create a **Storage bucket** named `module-media` and mark it **public**.

### Option B — Self-hosted Supabase
Follow https://supabase.com/docs/guides/self-hosting then perform the same steps as Option A against your own instance URL.

### Seed the super administrator
After deploying, sign up once via the app (this becomes a normal trainee), then in the Supabase SQL editor run:
```sql
insert into public.user_roles (user_id, role)
values ('<your-auth-user-id>', 'super_admin');
```
That account can now sign in at `/admin/login` and create additional admins from the **Admins** tab.

---

## 3. Frontend build

```bash
# 1. Install dependencies
npm install

# 2. Configure backend connection
cp .env.example .env   # then edit with your values
# OR create .env directly with:
# VITE_SUPABASE_URL=https://xxxx.supabase.co
# VITE_SUPABASE_PUBLISHABLE_KEY=eyJ...
# VITE_SUPABASE_PROJECT_ID=xxxx

# 3. Build
npm run build

# Output: dist/  ← upload this to your server
```

---

## 4. Hosting `dist/`

### Nginx
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    root /var/www/stcrane/dist;
    index index.html;

    # SPA fallback — required for React Router deep links
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Long cache for hashed assets
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Never cache the service worker
    location = /sw.js {
        add_header Cache-Control "no-cache";
    }
}
```

### Caddy (auto-HTTPS)
```
yourdomain.com {
    root * /var/www/stcrane/dist
    try_files {path} /index.html
    file_server
}
```

### Apache (`.htaccess` in `dist/`)
```apache
RewriteEngine On
RewriteBase /
RewriteRule ^index\.html$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.html [L]
```

---

## 5. Docker (optional one-shot)

`Dockerfile`:
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_SUPABASE_URL
ARG VITE_SUPABASE_PUBLISHABLE_KEY
ARG VITE_SUPABASE_PROJECT_ID
ENV VITE_SUPABASE_URL=$VITE_SUPABASE_URL
ENV VITE_SUPABASE_PUBLISHABLE_KEY=$VITE_SUPABASE_PUBLISHABLE_KEY
ENV VITE_SUPABASE_PROJECT_ID=$VITE_SUPABASE_PROJECT_ID
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

Build & run:
```bash
docker build \
  --build-arg VITE_SUPABASE_URL=https://xxxx.supabase.co \
  --build-arg VITE_SUPABASE_PUBLISHABLE_KEY=eyJ... \
  --build-arg VITE_SUPABASE_PROJECT_ID=xxxx \
  -t stcrane .

docker run -p 8080:80 stcrane
```

---

## 6. Going live checklist

- [ ] Backend deployed, migrations applied, `module-media` bucket created
- [ ] Edge functions `create-admin` and `revoke-admin` deployed
- [ ] One user promoted to `super_admin` in `user_roles`
- [ ] `.env` filled with your project's URL + anon key
- [ ] `npm run build` produces a clean `dist/`
- [ ] Static host serves `index.html` for unknown routes (SPA fallback)
- [ ] HTTPS enabled (required for the PWA service worker + install prompt)
- [ ] Custom domain pointed at your host

---

## 7. Day-to-day operation

- **Super admin** signs in at `/admin/login`, opens the **Admins** tab to create/revoke regular admins.
- **Admins** manage modules, questions and trainee data. They cannot create or remove other admins.
- **Trainees** sign up at `/auth` (employee number required) and complete the induction.

That's it — fully self-contained, your data, your server.
