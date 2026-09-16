# Static React SPA Deployment on a Linux Server

This guide explains how to deploy a **static React SPA** on a Linux server using:

* React / Vite
* Nginx
* NVM + Node.js
* Production environment variables
* DNS
* HTTPS with Certbot
* Swap for low-memory servers

This approach is for frontend applications that produce a static build such as:

```text
dist/
├── index.html
└── assets/
```

Nginx serves those files directly.

It does **not** require:

* Gunicorn
* systemd for the frontend
* a continuously running Node process

Node.js is only needed to install dependencies and build the frontend.

---

## 1. Target Architecture

A typical deployment might look like:

```text
https://app.example.com     -> React frontend
https://api.example.com     -> Django backend
```

On the server:

```text
Internet
   |
   v
Nginx :80/:443
   |
   +----------------------+
   |                      |
   v                      v
React static files     Django API
dist/                  Gunicorn
```

For the frontend specifically:

```text
Browser
   |
   v
Nginx
   |
   v
dist/index.html
dist/assets/*
```

There is no React development server running in production.

---

# 2. Prepare the Server

SSH into the server:

```bash
ssh clinton@YOUR_SERVER_IP
```

Update packages:

```bash
sudo apt update
sudo apt upgrade -y
```

Install useful packages:

```bash
sudo apt install -y nginx git curl
```

Check Nginx:

```bash
sudo systemctl status nginx
```

You should see:

```text
Active: active (running)
```

---

# 3. Clone the Frontend Project

Move to your home directory:

```bash
cd /home/clinton
```

Clone the repository:

```bash
git clone <your-frontend-repository-url> frontend
```

Enter it:

```bash
cd frontend
```

For example:

```text
/home/clinton/frontend/
```

Your project might initially look like:

```text
frontend/
├── src/
├── public/
├── package.json
├── package-lock.json
├── index.html
└── vite.config.ts
```

At this point, `dist/` usually does not exist yet.

---

# 4. Install Node.js With NVM

Install NVM:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

Load NVM:

```bash
source ~/.bashrc
```

Confirm:

```bash
nvm --version
```

Install Node.js:

```bash
nvm install --lts
```

Use it:

```bash
nvm use --lts
```

Confirm:

```bash
node -v
npm -v
```

The relationship is:

```text
NVM
 |
 +--> manages Node.js versions
          |
          +--> includes npm
```

---

# 5. Configure Production Environment Variables

Vite frontend variables normally start with:

```text
VITE_
```

For example, during development:

```env
VITE_API_BASE_URL=http://localhost:8000
```

For production:

```env
VITE_API_BASE_URL=https://api.example.com
```

Create:

```bash
nano .env.production
```

Add:

```env
VITE_API_BASE_URL=https://api.example.com
```

Save the file.

Vite normally reads `.env.production` when running:

```bash
npm run build
```

## Important

Frontend environment variables are **not secret**.

Anything included in a frontend build can eventually be inspected by users.

Do not put values such as these in a frontend env file:

```env
DATABASE_PASSWORD=...
SECRET_KEY=...
PAYSTACK_SECRET_KEY=...
SMTP_PASSWORD=...
```

Frontend environment variables should contain public configuration such as:

```env
VITE_API_BASE_URL=https://api.example.com
```

---

# 6. Add Swap on Low-Memory Servers

Frontend production builds can temporarily require significantly more memory than the deployed frontend itself.

A small server such as:

```text
1 vCPU
512 MB RAM
```

may run out of memory while executing:

```bash
npm run build
```

When that happens, Linux may terminate the Node/Vite build and simply print:

```text
Killed
```

Check memory:

```bash
free -h
```

Check swap:

```bash
swapon --show
```

If you see something like:

```text
Mem:    458Mi
Swap:      0B
```

the server has no emergency memory.

## Create a 2 GB swap file

Run:

```bash
sudo fallocate -l 2G /swapfile
```

Secure it:

```bash
sudo chmod 600 /swapfile
```

Format it:

```bash
sudo mkswap /swapfile
```

Enable it:

```bash
sudo swapon /swapfile
```

Verify:

```bash
free -h
```

You should now see something similar to:

```text
               total
Mem:           458Mi
Swap:          2.0Gi
```

Make swap persistent across reboots:

```bash
grep -q '^/swapfile ' /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Reduce swap aggressiveness

Check:

```bash
cat /proc/sys/vm/swappiness
```

Set it to `10`:

```bash
sudo sysctl vm.swappiness=10
```

Make it persistent:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
```

Verify:

```bash
cat /proc/sys/vm/swappiness
```

Expected:

```text
10
```

Swap is not a replacement for RAM.

Think of it as:

```text
RAM
 |
 | memory pressure
 v
Swap
```

Your application still prefers RAM. Swap provides a slower emergency buffer instead of Linux immediately killing a process.

---

# 7. Install Frontend Dependencies

From the project directory:

```bash
cd /home/clinton/frontend
```

Install dependencies:

```bash
npm install
```

For deployments with an existing `package-lock.json`, you can also use:

```bash
npm ci
```

`npm ci` is particularly useful for repeatable deployments because it installs the exact dependency versions recorded in the lock file.

---

# 8. Build the Frontend

Run:

```bash
npm run build
```

For a Vite application, this usually runs something similar to:

```text
TypeScript
   ↓
Vite
   ↓
Production build
   ↓
dist/
```

A successful build normally creates:

```text
dist/
├── index.html
└── assets/
    ├── index-xxxxx.js
    └── index-xxxxx.css
```

Verify:

```bash
ls -lah dist
```

You can also locate the generated HTML:

```bash
find . -maxdepth 3 -type f -name index.html
```

---

# 9. If `npm run build` Says `Killed`

If you get:

```text
Killed
```

check the Linux kernel log:

```bash
sudo journalctl -k -n 100 | grep -Ei 'out of memory|oom|killed process'
```

If you see:

```text
Out of memory: Killed process ...
```

then Linux's OOM killer terminated the build because the machine ran out of memory.

Check:

```bash
free -h
swapon --show
```

If necessary, add swap as described earlier.

Then retry:

```bash
npm run build
```

---

# 10. Understand What Nginx Will Serve

Do not point Nginx at the React source directory just because it contains:

```text
index.html
```

For example, this:

```text
/home/clinton/frontend/index.html
```

is normally part of the Vite source project.

The production build is:

```text
/home/clinton/frontend/dist/index.html
```

The flow is:

```text
src/
index.html
package.json
    |
    | npm run build
    v
dist/
├── index.html
└── assets/
    |
    v
Nginx
```

Therefore Nginx should normally point to:

```text
/home/clinton/frontend/dist
```

---

# 11. Create the Nginx Configuration

Create:

```bash
sudo nano /etc/nginx/sites-available/frontend
```

Example:

```nginx
server {
    listen 80;
    server_name app.example.com www.app.example.com;

    root /home/clinton/frontend/dist;
    index index.html;

    location /assets/ {
        try_files $uri =404;
        access_log off;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Replace:

```text
app.example.com
```

with your real frontend domain.

Replace:

```text
/home/clinton/frontend/dist
```

with your actual frontend build path.

---

# 12. Why `try_files` Is Important for React

A React SPA might have routes such as:

```text
/
 /events
 /events/123
 /dashboard
 /profile
```

The server does not actually contain:

```text
dist/events/
dist/dashboard/
dist/profile/
```

React Router handles those routes in the browser.

Therefore:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

means:

```text
Request /assets/logo.png
        |
        +--> file exists
             serve it

Request /dashboard
        |
        +--> no physical file
             |
             v
          index.html
             |
             v
          React Router
```

Without the SPA fallback, refreshing `/dashboard` could result in a `404`.

---

# 13. Enable the Nginx Site

Create a symbolic link:

```bash
sudo ln -sf /etc/nginx/sites-available/frontend /etc/nginx/sites-enabled/frontend
```

The relationship becomes:

```text
sites-available/frontend
           ^
           |
       symlink
           |
sites-enabled/frontend
```

Check:

```bash
ls -l /etc/nginx/sites-enabled/
```

You should see:

```text
frontend -> /etc/nginx/sites-available/frontend
```

---

# 14. Remove the Default Nginx Site

If you no longer need it:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

This removes only the enabled symbolic link, not the original configuration file.

---

# 15. Test Nginx Before Reloading

Always run:

```bash
sudo nginx -t
```

A successful result looks like:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Then reload:

```bash
sudo systemctl reload nginx
```

Do not routinely reload Nginx before testing the configuration.

The safe sequence is:

```text
Edit config
   ↓
sudo nginx -t
   ↓
successful?
   ↓
sudo systemctl reload nginx
```

---

# 16. Check Linux Permissions

Nginx normally runs as:

```text
www-data
```

If your build lives under:

```text
/home/clinton/frontend/dist
```

Nginx must be able to traverse every directory leading to the file.

Check:

```bash
namei -l /home/clinton/frontend/dist/index.html
```

You may see something like:

```text
f: /home/clinton/frontend/dist/index.html
drwxr-xr-x root    root    /
drwxr-xr-x root    root    home
drwx--x--x clinton clinton clinton
drwxrwxr-x clinton clinton frontend
drwxrwxr-x clinton clinton dist
-rw-rw-r-- clinton clinton index.html
```

Directories need the execute/traverse permission required for Nginx to reach the file.

Do not blindly use:

```bash
chmod -R 777 ...
```

That creates unnecessary security problems.

---

# 17. Configure DNS

In your DNS provider, create an `A` record.

For example:

```text
Type: A
Name: app
Value: YOUR_SERVER_IP
```

Result:

```text
app.example.com
       |
       v
YOUR_SERVER_IP
```

If using:

```text
www.app.example.com
```

you may need another record.

Check DNS:

```bash
dig app.example.com
```

or:

```bash
nslookup app.example.com
```

---

# 18. Test the HTTP Deployment

Run:

```bash
curl -I http://app.example.com
```

A successful response should resemble:

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

You can also visit:

```text
http://app.example.com
```

in the browser.

---

# 19. Install HTTPS With Certbot

Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Issue a certificate:

```bash
sudo certbot --nginx -d app.example.com
```

If using both:

```text
app.example.com
www.app.example.com
```

run:

```bash
sudo certbot --nginx \
  -d app.example.com \
  -d www.app.example.com
```

Certbot will normally modify the Nginx configuration and enable HTTPS.

Test:

```bash
curl -I https://app.example.com
```

---

# 20. Test Certificate Renewal

Run:

```bash
sudo certbot renew --dry-run
```

This verifies that automatic certificate renewal should work.

---

# 21. Configure the Django Backend for the Frontend

If your React application communicates with a Django backend at:

```text
https://api.example.com
```

the backend may need to trust the frontend origin.

Example:

```env
CORS_ALLOWED_ORIGINS=https://app.example.com
CSRF_TRUSTED_ORIGINS=https://app.example.com,https://api.example.com
FRONTEND_BASE_URL=https://app.example.com
```

Exact settings depend on how your Django application handles authentication, cookies, CORS and CSRF.

Restart the backend after configuration changes:

```bash
sudo systemctl restart gunicorn
```

---

# 22. Verify the API URL Used in the Frontend

Because Vite injects `VITE_*` variables during the build, check:

```bash
cat .env.production
```

For example:

```env
VITE_API_BASE_URL=https://api.example.com
```

Then rebuild:

```bash
npm run build
```

You can search the generated files:

```bash
grep -R "api.example.com" dist | head
```

This helps confirm that the production API URL was included.

---

# 23. Normal Deployment Update Flow

When you make frontend changes and push them to GitHub, deployment can be:

```bash
cd /home/clinton/frontend
git pull
npm ci
npm run build
```

Since Nginx already points at:

```text
/home/clinton/frontend/dist
```

the new build replaces the old generated files.

You generally do **not** need to restart Nginx just because frontend files changed.

Nginx reads the new static files automatically.

So your normal deployment can simply be:

```bash
cd /home/clinton/frontend
git pull
npm ci
npm run build
```

Then test:

```bash
curl -I https://app.example.com
```

If you changed the Nginx configuration itself, then run:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

# 24. Why `dist/` Should Usually Stay Out of Git

A Vite `.gitignore` often contains:

```text
dist/
```

This is normal.

Your GitHub repository stores:

```text
src/
package.json
package-lock.json
vite.config.ts
...
```

The production server generates:

```text
dist/
```

during deployment.

The flow is:

```text
Developer PC
    |
    | git push
    v
GitHub
    |
    | git pull
    v
Production server
    |
    | npm ci
    | npm run build
    v
dist/
    |
    v
Nginx
```

`dist/` does not need to be committed to GitHub.

---

# 25. Alternative: Build With GitHub Actions

As the deployment becomes more mature, you may decide not to compile the frontend on the production server.

Instead:

```text
GitHub repository
       |
       v
GitHub Actions
       |
       +--> npm ci
       +--> npm run build
       |
       v
dist/
       |
       | rsync / scp
       v
Production server
       |
       v
Nginx
```

The generated `dist/` still does not need to be committed.

GitHub Actions can build it temporarily and copy it directly to the server.

This is particularly useful for very small servers because TypeScript/Vite compilation happens elsewhere.

---

# 26. Troubleshooting: `500 Internal Server Error`

Check:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

If you see:

```text
rewrite or internal redirection cycle while internally redirecting to "/index.html"
```

one common cause is that Nginx is configured with:

```nginx
try_files $uri $uri/ /index.html;
```

but the configured:

```text
dist/index.html
```

does not exist.

Check:

```bash
ls -lah /home/clinton/frontend/dist
```

If:

```text
dist/
```

does not exist, the build probably hasn't completed successfully.

Run:

```bash
npm run build
```

---

# 27. Troubleshooting: `403 Forbidden`

Check the Nginx error log:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

Check the file:

```bash
ls -lah /home/clinton/frontend/dist/index.html
```

Check the entire path:

```bash
namei -l /home/clinton/frontend/dist/index.html
```

Common causes:

* Nginx cannot access `/home/clinton`.
* Nginx cannot traverse one of the directories.
* `index.html` isn't readable.
* The configured `root` is incorrect.

---

# 28. Troubleshooting: `404` on React Routes

Suppose:

```text
https://app.example.com/
```

works, but:

```text
https://app.example.com/dashboard
```

returns `404` after refreshing.

Check that your Nginx configuration contains:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

This fallback is necessary for client-side routing.

---

# 29. Troubleshooting: Wrong Backend API

Suppose the frontend loads but requests go to:

```text
http://localhost:8000
```

instead of:

```text
https://api.example.com
```

Check:

```bash
cat .env.production
```

Make sure:

```env
VITE_API_BASE_URL=https://api.example.com
```

Then rebuild:

```bash
npm run build
```

Remember:

> Changing `.env.production` does not modify an already-built frontend.

You must rebuild because Vite embeds the environment values into the generated JavaScript.

---

# 30. Troubleshooting: Nginx Configuration Errors

Always run:

```bash
sudo nginx -t
```

If you get:

```text
open() "/etc/nginx/sites-enabled/..." failed
```

check your symbolic links:

```bash
ls -l /etc/nginx/sites-enabled/
```

For example, this is correct:

```text
frontend -> /etc/nginx/sites-available/frontend
```

A broken link might look valid in `sites-enabled` but point to a file that doesn't exist.

Remove the broken link:

```bash
sudo rm /etc/nginx/sites-enabled/frontend
```

Then recreate it:

```bash
sudo ln -s \
  /etc/nginx/sites-available/frontend \
  /etc/nginx/sites-enabled/frontend
```

Test again:

```bash
sudo nginx -t
```

---

# 31. Monitor Memory

On a small server, periodically check:

```bash
free -h
```

You can also use:

```bash
htop
```

Install it if necessary:

```bash
sudo apt install -y htop
```

For more detailed memory and swap activity:

```bash
vmstat 1
```

You do not want the server continuously depending heavily on swap.

For example:

```text
RAM:   458 MiB
Swap:   30 MiB used
```

may be acceptable.

Something like:

```text
RAM:   458 MiB
Swap:  900 MiB used
```

during normal operation suggests the machine needs more RAM.

---

# 32. Recommended Directory Layout

A simple setup could be:

```text
/home/clinton/
├── backend/
│   ├── manage.py
│   └── ...
│
└── frontend/
    ├── src/
    ├── public/
    ├── package.json
    ├── package-lock.json
    ├── .env.production
    │
    └── dist/
        ├── index.html
        └── assets/
```

Nginx then serves:

```text
/home/clinton/frontend/dist
```

while Gunicorn serves the backend separately.

---

# 33. Complete Deployment Flow

For a new server:

```text
1. Provision Linux server
        ↓
2. Install Nginx, Git and curl
        ↓
3. Clone frontend repository
        ↓
4. Install NVM
        ↓
5. Install Node.js
        ↓
6. Configure .env.production
        ↓
7. Add swap if server has little RAM
        ↓
8. npm ci
        ↓
9. npm run build
        ↓
10. Confirm dist/index.html
        ↓
11. Configure Nginx
        ↓
12. Enable Nginx site
        ↓
13. Configure DNS
        ↓
14. Test HTTP
        ↓
15. Install SSL with Certbot
        ↓
16. Test HTTPS
```

For later deployments:

```text
git pull
   ↓
npm ci
   ↓
npm run build
   ↓
Nginx immediately serves new dist/
```

---

# 34. Verification Checklist

### Node

```bash
node -v
npm -v
```

### Memory

```bash
free -h
swapon --show
```

### Build

```bash
ls -lah dist
```

Make sure:

```text
dist/index.html
```

exists.

### Nginx

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
```

### HTTP

```bash
curl -I http://app.example.com
```

### HTTPS

```bash
curl -I https://app.example.com
```

### Error logs

```bash
sudo tail -n 100 /var/log/nginx/error.log
```

### Production API

```bash
grep -R "api.example.com" dist | head
```

---

# 35. Best Practices

* Keep `dist/` out of Git unless there is a specific reason to commit generated artifacts.
* Use `npm ci` for predictable production installations when you have a lock file.
* Keep production API URLs in `.env.production`.
* Never put backend secrets into `VITE_*` variables.
* Always run `sudo nginx -t` before reloading Nginx.
* Use HTTPS for production.
* Keep frontend and backend on clear domains or subdomains.
* Configure SPA fallback with `try_files`.
* Cache hashed `/assets/` files aggressively.
* Keep swap available on very small servers as an emergency buffer.
* Do not treat swap as a replacement for adequate RAM.
* Monitor RAM, swap and disk usage.
* Upgrade server resources if swap is regularly active during normal traffic.
* Consider moving builds to GitHub Actions when your deployment workflow matures.
* Do not run `vite dev` or `npm run dev` as your production frontend server for a static SPA.

The core production model is simply:

```text
React source code
      ↓
npm run build
      ↓
dist/
      ↓
Nginx
      ↓
Browser
```

For a static React/Vite SPA, that is the main architecture to keep in mind.
