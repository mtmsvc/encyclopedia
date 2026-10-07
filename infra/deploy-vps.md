# Deploying to a VPS
How we took `blasto` from a Docker image on a laptop to `https://<domain>`, served over HTTPS from our own server and redeployed automatically on every push to `main`. Written as the story of what we did, why, and what happens on the network at each step, so it can be reused for any app.

Placeholders used below: `<ip>` (the server's public IP), `<user>` (the login user on the server), `<domain>` (e.g. `blasto.me`), `<owner>` (the GitHub account, lowercase).

Parts marked **(Background: not done in blasto)** explain alternatives or next steps we did not perform. Everything else is what we actually did.

---

## 0. The big picture
To put an app on the internet, five problems have to be solved:

1. **A machine** that is always on and reachable: the server.
2. **A way in** to manage it: SSH.
3. **A name** people can type instead of an IP address: DNS.
4. **A way to get the app onto the machine**: an image registry.
5. **A safe front door** that speaks HTTPS and passes requests to the app: a reverse proxy.

Then we automate it all, so a `git push` does the deploy: CI/CD.

The final picture:

```
 phone/browser
      │  1. "where is <domain>?"  ──►  DNS  ──►  "<ip>"
      │  2. HTTPS to <ip>:443
      ▼
 ┌──────────────────────── Azure ───────────────────────────────┐
 │  Network Security Group (firewall): allows 22, 80, 443 only  │
 │  ┌──────────────────── VM (Ubuntu) ────────────────────────┐ │
 │  │  Docker                                                  │ │
 │  │  ┌──────────────┐  internal network  ┌───────────────┐  │ │
 │  │  │ caddy        │ ─── app:8000 ────► │ app (blasto)  │  │ │
 │  │  │ ports 80,443 │    plain HTTP      │ no public port│  │ │
 │  │  └──────────────┘                    └───────────────┘  │ │
 │  └──────────────────────────────────────────────────────────┘ │
 └───────────────────────────────────────────────────────────────┘
      ▲
      │  on push to main: GitHub Actions builds image → pushes to GHCR
      │                   → SSH into VM → pull + restart
```

---

## 1. Networking basics we need
Everything later builds on these.

- **IP address** - the address of a machine on a network, e.g. `203.0.113.7` (IPv4). Like the address of a building.
- **Port** - a number (0-65535) that picks *which program* on that machine gets the traffic. Like the apartment number. Conventions: `22` SSH, `80` HTTP, `443` HTTPS. A program "listens" on a port; traffic to a port nobody listens on is refused.
- **TCP** - the protocol that carries HTTP, HTTPS and SSH: a reliable two-way connection between `client_ip:random_port` and `server_ip:port`.
- **Public vs private IP** - a public IP is reachable from the internet. A private IP (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`) works only inside a local network. Our Azure VM itself only has a private IP (`ip addr` shows `10.x.x.x` on `eth0`); Azure maps the public IP to it in front of the VM (this is NAT - network address translation).
- **`127.0.0.1` vs `0.0.0.0`** - `127.0.0.1` (localhost) means "this machine only". A server listening on `0.0.0.0` accepts connections on all network interfaces. This is why `uvicorn` in the container must use `--host 0.0.0.0`: inside a container, localhost is the container itself, so `127.0.0.1` would be unreachable from outside it.

---

## 2. The machine: what a VPS is
A **VPS** (virtual private server) is a virtual machine: a slice of a big physical server, with its own Linux, its own disk and a root account. We manage everything on it ourselves.

The other options, from "do everything" to "do nothing":

| Type | What you get | You manage | Examples |
|---|---|---|---|
| Dedicated server | A whole physical machine | Everything | Hetzner dedicated |
| **VPS** | A virtual machine with root access | OS, security, Docker, proxy, HTTPS, deploys | Hetzner Cloud, Azure VM, AWS EC2, DigitalOcean |
| Hyperscaler services | VMs plus hundreds of managed services (databases, load balancers, autoscaling) | As much or as little as you choose | AWS, Azure, GCP |
| PaaS | You hand over code or an image; they run it | Almost nothing | Heroku, Render, Fly.io |
| Frontend / serverless | Hosting for websites and short functions | Almost nothing | Vercel, Netlify |

We chose a VPS on purpose: every layer done by hand once makes PaaS and AWS easy to understand later - they just do some of these layers for you.

**Load balancing** (spreading traffic over several copies of the app) is built in on PaaS, is a product on hyperscalers, and does not exist on a single VPS. One server is enough until real traffic says otherwise.

### Creating the VM on Azure for Students
Azure for Students gives $100 credit for 12 months, no credit card, plus free hours of small VMs.

Settings that matter (Virtual machines → Create):

| Field | Value | Why |
|---|---|---|
| Resource group | new, `blasto` | A folder for everything belonging to this project |
| Region | one allowed by the subscription | See the gotcha below |
| Availability options | No infrastructure redundancy required | Zones matter only with several VMs; pinning one VM to a zone can make sizes unavailable |
| Image | Ubuntu Server 24.04 LTS | LTS = long-term support, years of security updates |
| Architecture | x64 | The deploy job builds x64 images; an Arm VM cannot run them without multi-arch builds |
| Size | Standard_B2ats_v2 (fallback: B1s) | Free-tier sizes; "B" = burstable: collects CPU credits while idle, spends them when busy |
| Azure Spot | off | Spot VMs are cheap because Azure can shut them down any time |
| Authentication | SSH public key, existing key | See section 3 |
| Inbound ports | SSH (22) only, at creation | Open 80/443 later, when something listens there |

**Gotchas we hit:**
- `RequestDisallowedByAzure` - student subscriptions may deploy only to a fixed list of regions. Find the list: portal search **Policy** → **Assignments** → the "allowed regions/locations" assignment → **Parameters**. Ours: `italynorth, germanywestcentral, norwayeast, switzerlandnorth, denmarkeast`.
- `NotAvailableForSubscription` on a size - capacity differs per region (and per zone). Try without a zone, then another allowed region.
- "Price unavailable" - student subscriptions often do not show prices. Check **Cost Management** after a day instead.

**(Background: not done in blasto)** Resizing later is possible: stop the VM, change the size, start it.

---

## 3. A way in: SSH
**SSH** (secure shell) gives an encrypted terminal on a remote machine, over TCP port 22.

### Key pairs
SSH uses asymmetric cryptography: a **key pair**.
- The **private key** (`~/.ssh/id_ed25519`) stays on our laptop, never shared.
- The **public key** (`~/.ssh/id_ed25519.pub`) can be given to anyone.

The server keeps a list of allowed public keys in `~/.ssh/authorized_keys`. On login, the server sends a challenge only the matching private key can answer. No password ever crosses the network.

One key per *device* is the norm: the key identifies our laptop, not the service. Every server gets the same public key; the private key stays home.

### Host keys: the other direction
On the first connection SSH asks:
```
The authenticity of host '<ip>' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```
This is the *server* proving its identity to us. Saying `yes` stores its fingerprint in `~/.ssh/known_hosts`. If it ever changes, SSH refuses to connect - someone may be impersonating the server. The deploy job gets this fingerprint in advance (section 11), because it cannot answer the question.

### Commands
```bash
ssh <user>@<ip>                               # log in with the default key
ssh -i ~/.ssh/blasto_deploy <user>@<ip>       # log in with a specific key
ssh-copy-id -i ~/.ssh/<key>.pub <user>@<ip>   # add a public key to the server's authorized_keys
ssh-keyscan <ip>                              # print the server's host keys (for known_hosts)
```
**(Background: not done in blasto)** Check that password login is off on the server (Azure disables it when you choose key authentication):
```bash
sudo sshd -T | grep passwordauthentication    # expect: passwordauthentication no
```

### First thing on a new server: updates
```bash
sudo apt update && sudo apt upgrade -y
ls /var/run/reboot-required && sudo reboot    # reboot if the kernel was updated
```

---

## 4. A name: DNS
Machines find each other by IP. People type names. **DNS** (Domain Name System) is the phonebook that turns `<domain>` into `<ip>`.

### Who is who
- **Registrar** - where we rent the name for a year at a time (Namecheap; GitHub Student Pack gives a free `.me` for a year, renewal is paid).
- **DNS records** - the entries in the phonebook, edited in the registrar's panel (Namecheap: Domain List → Manage → Advanced DNS).
- **Resolver** - the server our phone asks (usually our internet provider's, or `1.1.1.1`/`8.8.8.8`). It looks the name up and caches the answer.

### Records we care about
| Type | Maps | Example |
|---|---|---|
| **A** | name → IPv4 | `@` → `203.0.113.7` |
| AAAA | name → IPv6 | `@` → `2001:db8::7` |
| CNAME | name → another name | `www` → `<domain>` |

`@` means the bare domain itself. **TTL** (time to live) says how long resolvers may cache an answer - why a change takes a while to be seen everywhere. That is why we set DNS *first*, before the server was ready: it spreads while we work on other things.

Namecheap adds parking records by default (a `www` CNAME to its parking page and a URL redirect on `@`). Delete them: a redirect on `@` conflicts with our A record. Then add our own `www` record, or `www.<domain>` stops resolving at all:

| Type | Host | Value |
|---|---|---|
| A Record | `@` | `<ip>` |
| CNAME Record | `www` | `<domain>` |

The CNAME says "`www` is wherever `<domain>` is", so it follows the A record automatically. Caddy also needs to know about `www` (section 8).

**This bit us:** we deleted the parking `www` CNAME without adding our own, so `www.blasto.me` did not work.

### Check it
```bash
dig +short <domain>          # expect: <ip>
nslookup <domain>            # same, if dig is missing
```

---

## 5. The doors: ports and firewalls
A **firewall** decides which traffic may reach the machine at all. We have two possible layers:

1. **Azure Network Security Group (NSG)** - a firewall *outside* the VM, in Azure's network. Traffic it blocks never reaches the VM. We allowed inbound `22`, `80`, `443` (VM → Networking → Create port rule → Inbound; names like `allow-http`, `allow-https`).
2. **(Background: not done in blasto)** **`ufw`** - a firewall *inside* the VM (Linux kernel rules). Off by default on Azure's Ubuntu image (`sudo ufw status`). On providers without an outside firewall (e.g. many plain VPS hosts) this is the one we must configure.

Why exactly these ports:
- `22` - SSH, for us and for the deploy job.
- `80` - plain HTTP. Needed so Let's Encrypt can verify we own the domain, and so Caddy can redirect `http://` to `https://`.
- `443` - HTTPS, the real traffic.

Port `8000` (the app) is **not** opened. Nobody outside should reach the app directly.

### The Docker firewall trap
When a container publishes a port (`ports: - "8000:8000"`), Docker writes its own rules into the kernel's packet filter (iptables), and those rules are checked *before* `ufw`'s. So a published port is reachable from the internet even if `ufw` says it is blocked.

The rule we follow: **publish only the reverse proxy's ports.** That is why `app` in our `compose.yml` (section 9) has no `ports:` at all and is reached through Docker's internal network. On Azure the NSG sits outside the VM and still blocks, but on hosts that rely on `ufw` alone this trap exposes everything.

**(Background: not done in blasto)** Check what listens on the server:
```bash
sudo ss -tlnp          # TCP sockets in LISTEN state, with the owning program
```

---

## 6. Docker on the server
Installed exactly like on the laptop, from Docker's own apt repository (not Ubuntu's `docker.io` package, which is often old): https://docs.docker.com/engine/install/ubuntu/ → "Install using the apt repository". This also installs the Compose plugin (`docker compose`).

```bash
sudo usermod -aG docker $USER      # run docker without sudo; log out and in after
groups                             # check: "docker" is listed
docker run --rm hello-world        # test
```

Membership in the `docker` group is effectively root (a container can mount the whole host filesystem). On a cloud VM where our user already has passwordless `sudo`, this gives nothing new. **(Background: not done in blasto)** On a laptop with password `sudo`, it means programs running as us can get root without the password - a common, accepted trade-off on dev machines.

**This bit us:** the deploy job failed with `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`. The error came from the *server*: our user was not in the `docker` group. The alternative is `sudo docker ...` in the deploy command - same security, more typing.

---

## 7. Getting the image to the server: registries
The image is built on one machine (laptop, later the deploy job) and must run on another (the server). A **registry** sits between them: we **push** an image to it, the server **pulls** it. **GHCR** (`ghcr.io`) is GitHub's registry; Docker Hub is the best known public one; AWS has ECR, Azure has ACR.

### Image names
```
ghcr.io / <owner> / blasto : manual
registry  account   image    tag
```
- The **registry** part tells Docker where to push and pull. The name must be lowercase.
- The **tag** is a label for one version. `latest` is just a tag, not magic. We used `manual` for the first hand-push and the **git commit SHA** for every automated build: the SHA names one exact build of one exact commit, which is what makes rollback possible (redeploy the previous SHA).
- Pushes upload only layers the registry does not have yet - the same layer caching as in the build.

### First push, by hand
1. **Token** - GHCR does not accept the GitHub password. GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)** → scope `write:packages`, short expiry. Shown once; copy it.
2. **Log in, build, push:**
```bash
echo <token> | docker login ghcr.io -u <owner> --password-stdin
docker build -t ghcr.io/<owner>/blasto:manual .
docker push ghcr.io/<owner>/blasto:manual
```
`--password-stdin` keeps the token out of the shell history. The login is saved in `~/.docker/config.json` (per user: with `sudo docker`, root has its own login).

**This bit us:** `denied` on push - we logged in as our user but pushed with `sudo`, i.e. as root, who was not logged in. Use the same mode for both.

### Public or private image
GHCR packages start **private**. The image appears under the GitHub profile → **Packages**.

- **Public** (our choice): Package settings → Danger Zone → Change visibility. Anyone can pull it, and the server needs no login. Fine when the repo is public anyway - the image contains nothing the repo does not already show.
- **(Background: not done in blasto)** **Private** (for closed-source code): keep it private and log the server in once with a separate token that has only `read:packages`:
  ```bash
  echo <read-token> | docker login ghcr.io -u <owner> --password-stdin
  ```
  The login is saved, so later pulls (including automatic deploys) work. **Catch:** when the token expires, deploys fail with an auth error until we create a new token and log in again. Use a long expiry and a calendar reminder.

**Actions access:** a package created with a personal token is not yet linked to the repo, so the workflow's built-in token may not push to it. Fix: Package settings → **Manage Actions access** → add the repo → role **Write**.

---

## 8. The front door: proxies, HTTPS and Caddy
### Proxies
A **proxy** is a middleman that receives traffic and passes it on. Which side it stands on gives it its name.

| Kind | Stands in front of | Purpose | Example |
|---|---|---|---|
| **Forward proxy** | Clients | Clients reach the internet through it; websites see the proxy, not the client (privacy, filtering, the "hacker proxy" from movies) | Corporate proxy, VPN-like services |
| **Reverse proxy** | Servers | Clients talk only to it; it forwards to the real app behind it | **Caddy**, Nginx, Traefik |
| Load balancer | Several copies of a server | A reverse proxy that spreads requests over many app copies | AWS ALB, Hetzner LB, Nginx |
| API gateway | Many services | A reverse proxy plus auth, rate limits, routing per API | Kong, AWS API Gateway |
| CDN | Servers, worldwide | Reverse proxies near users that cache responses | Cloudflare |

What a reverse proxy gives us:
- **One public entry point.** Only it is exposed; apps stay internal.
- **TLS termination.** HTTPS is handled in one place; the apps speak plain HTTP on the internal network.
- **Routing.** One server can host several apps: `api.<domain>` → backend, `<domain>` → frontend.
- Later: load balancing, compression, rate limits, access logs.

### HTTPS and certificates
HTTPS = HTTP inside **TLS**. TLS gives two things:
1. **Encryption** - nobody between phone and server can read or change the traffic.
2. **Identity** - the server proves it really is `<domain>`, using a **certificate**.

A certificate says "this public key belongs to `<domain>`", signed by a **certificate authority (CA)** that browsers trust. **Let's Encrypt** is a free CA. Before signing, it checks we control the domain - the **ACME challenge**: it asks for a special file at `http://<domain>/.well-known/acme-challenge/...` (over port 80), or does an equivalent check over 443. That is why DNS must point at the server and ports 80/443 must be open *before* Caddy can get a certificate.

Certificates are short-lived on purpose and must be renewed regularly. Doing that by hand is how sites go down; Caddy does it automatically. (This is also why the free "SSL certificate" from the Student Pack is useless here: one year, installed and renewed by hand.)

### Caddy
Caddy is a reverse proxy with **automatic HTTPS**: given a domain name, it gets the certificate, renews it, and redirects HTTP to HTTPS - no extra config. The whole config:

```
<domain> {
    reverse_proxy app:8000
}

www.<domain> {
    redir https://<domain>{uri}
}
```
- First block: "For requests to `<domain>`: handle HTTPS, forward everything to `app` on port 8000." `app` is the Compose service name (section 9).
- Second block: "For `www.<domain>`: get a certificate too, and send visitors to the bare domain, keeping the path (`{uri}`)." One canonical address, so links and search engines see a single site.

After editing the Caddyfile: `docker compose restart caddy`.

**(Background: not done in blasto)** `.dev` and `.app` domains are HTTPS-only in browsers (HSTS preload). Before the certificate exists, a browser shows nothing at all; test with `curl` instead.

---

## 9. Running it together: Docker Compose
**Compose** runs several containers from one file and connects them. On the server, in `~/blasto`:

`compose.yml`:
```yaml
services:
  app:
    image: ghcr.io/<owner>/blasto:${IMAGE_TAG}
    env_file: app.env
    restart: unless-stopped

  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data

volumes:
  caddy_data:
```

`.env` (in the same folder; Compose reads it automatically and fills `${...}` in `compose.yml`):
```
IMAGE_TAG=manual
```

`app.env` (in the same folder; the app's own settings, e.g. `LOG_LEVEL`, later passwords):
```
LOG_LEVEL=INFO
```
```bash
chmod 600 app.env      # only our user can read it
```

The two files have different jobs:
- **`.env`** is read by Compose itself, to fill the blanks in `compose.yml`. Its values are **not** passed into the containers.
- **`app.env`** is passed **into the app container** as environment variables (`env_file: app.env`). This is what the app reads.

Keeping them separate also avoids a trap: the deploy job rewrites `.env` on every deploy (section 11), so any app setting stored in `.env` would be wiped.

`app.env` has no leading dot, so a plain `ls` shows it. That doesn't matter: a dot only hides a file from `ls`, it doesn't protect it. What protects it is that it exists only on the server, and `chmod 600`.

What each part does:
- **`app` has no `ports:`** - it is reachable only from other containers (the firewall trap, section 5).
- **`env_file: app.env`** - passes the variables in `app.env` into the app container.
- **`caddy` publishes 80 and 443** - the only doors to the outside.
- **`restart: unless-stopped`** - containers come back after a crash or a server reboot, unless we stopped them on purpose.
- **`./Caddyfile:/etc/caddy/Caddyfile`** - a *bind mount*. A container can't see the server's files, but Caddy needs our `Caddyfile`. This line takes `Caddyfile` from this folder on the server and makes it appear inside the container at `/etc/caddy/Caddyfile`, where Caddy looks for its config. The container does not get its own copy; it reads the file that sits on the server. So when we change the `Caddyfile` on the server, we only need to restart Caddy (`docker compose restart caddy`) for it to read the new version. We don't need to build or download a new image.
- **`caddy_data`** - a *named volume*: storage managed by Docker that survives container restarts and recreation. Caddy keeps its certificates there. Without it, every restart would request new certificates, and Let's Encrypt rate-limits how often that is allowed.
- **`${IMAGE_TAG}`** - a blank filled from `.env`, so a deploy can say which build to run by rewriting one line.

### The internal network
Compose connects all the containers in the project to their own private network. On that network, each container can reach the others by their service name. That's why Caddy can send requests to `app:8000`, even though the app isn't open to the outside.

### Commands
```bash
docker compose up -d             # create/update and start everything, in the background
docker compose pull              # download newer images for the current tags
docker compose ps                # what is running
docker compose logs -f caddy     # follow a service's logs
docker compose down              # stop and remove containers (volumes stay)
docker image prune -a            # (Background: not done in blasto) delete unused images (old SHA tags pile up)
```

---

## 10. The journey of one request
What happens when the phone opens `https://<domain>/health`:

1. **DNS.** The phone asks its resolver for `<domain>`. The resolver answers `<ip>` (from cache, or by asking the domain's DNS servers).
2. **TCP connect.** The phone opens a connection from `phone_ip:random_port` to `<ip>:443`.
3. **Azure.** The packet reaches Azure's network. The NSG checks it: port 443 allowed. Azure translates the public IP to the VM's private `10.x.x.x` address (NAT) and delivers it.
4. **Into the VM, into Docker.** The kernel sees traffic for port 443. Docker's published-port rule forwards it to the Caddy container's internal IP, port 443.
5. **TLS handshake.** The phone says which name it wants (SNI (Server Name Indication): `<domain>`). Caddy answers with the certificate for that name; the phone checks it was signed by a trusted CA and matches the name. Both sides agree on encryption keys.
6. **HTTP request, decrypted.** Caddy decrypts `GET /health`. Its config says: forward to `app:8000`.
7. **Internal hop.** Caddy asks Docker's internal DNS for `app`, gets the app container's internal IP, and sends plain HTTP to it on port 8000.
8. **The app.** `uvicorn` (listening on `0.0.0.0:8000` inside the container) passes the request to FastAPI, which runs the `/health` function and returns `{"status": "ok"}`.
9. **Back.** The response travels back to Caddy, which encrypts it and sends it to the phone over the same TLS connection.

A plain `http://<domain>` request takes the same path to port 80, where Caddy answers with a redirect to `https://`.

---

## 11. Automating: CI/CD
**CI** (continuous integration) checks every change. **CD** (continuous deployment) ships every change that passed. Ours: every push to `main` runs the checks, then builds, pushes and deploys.

### Pieces the deploy job needs
When we push to `main`, GitHub starts a temporary computer, the **runner**, which runs our workflow. The checks job (CI) needs nothing extra: it only tests the code on the runner. The deploy job (CD) is different. It has to log in to our server with SSH, the same way we do from the laptop. For that it needs three things: a key to log in with, the details of where to log in, and permission to upload the image.

1. **A deploy key.**
   The runner logs in with a key, like we do. We give it a **new** key, not our personal one. If the deploy key ever leaks, we delete only that key from the server, and our own access keeps working.

   Create the key on the laptop:
   ```bash
   ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/blasto_deploy -N ""
   ```
   - `-t ed25519` - the key type, the modern default.
   - `-C "github-actions-deploy"` - a label, so we can tell later which key this is.
   - `-f ~/.ssh/blasto_deploy` - save it under its own name, so it doesn't overwrite our personal key.
   - `-N ""` - no passphrase. The runner can't type one, so the key must work without it.

   This creates two files: `blasto_deploy` (the private key, which goes to GitHub) and `blasto_deploy.pub` (the public key, which goes to the server).

   Give the server the public key:
   ```bash
   ssh-copy-id -i ~/.ssh/blasto_deploy.pub <user>@<ip>
   ```
   This adds the key to the server's list of allowed keys (`~/.ssh/authorized_keys`). It logs in with our personal key to do that.

   Test that the new key works on its own:
   ```bash
   ssh -i ~/.ssh/blasto_deploy <user>@<ip>
   ```
   `-i` means "log in with this specific key". If we get the server's prompt, the key works.

2. **Secrets.**
   The deploy job needs the private key, the server's address, and the username. Our repo is public, so we can't write these into a file: anyone could read them and log in to our server.

   GitHub Secrets is a locked box in the repo settings. We save each value there once. The workflow can use it, but nobody can see it again, not even us, and GitHub hides it in the logs.

   Repo → Settings → Secrets and variables → Actions → New repository secret:

   | Secret | Value | Why |
   |---|---|---|
   | `DEPLOY_HOST` | `<ip>` | Where to log in |
   | `DEPLOY_USER` | `<user>` | Which user to log in as |
   | `DEPLOY_SSH_KEY` | the whole private key file (`cat ~/.ssh/blasto_deploy`), including the `-----BEGIN` and `-----END` lines | The key to log in with |
   | `DEPLOY_KNOWN_HOSTS` | the output of `ssh-keyscan <ip>` | The server's fingerprint (see below) |

   **Why the fingerprint:** the first time we connected to the server, SSH asked "Are you sure you want to continue connecting?" and we typed `yes`. SSH then remembered the server's fingerprint, its unique ID. The runner can't answer that question, so we give it the fingerprint in advance. This also protects it: if someone puts a fake server at our IP, the fingerprint won't match, and the runner refuses to connect.

3. **Permission to upload the image to GHCR.**
   The deploy job builds the image and uploads it to GHCR, so it needs permission to do that.

   - **Every run gets a temporary token, `GITHUB_TOKEN`,** created by GitHub for that run only. We don't create it or save it anywhere.
   - **`permissions` decides what this token may do.** In the deploy job, `packages: write` lets it upload images. Without that line, the upload is refused.
   - **The package must also allow our repo.** We first created the package with our personal token, so GHCR doesn't know the repo yet. Package settings → **Manage Actions access** → add the repo → role **Write** (section 7).

### The deploy job
Added under `jobs:`, at the same level as the checks job:
```yaml
  deploy:
    needs: checks
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v...

      - name: Log in to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Build and push image
        run: |
          docker build -t ghcr.io/<owner>/blasto:${{ github.sha }} .
          docker push ghcr.io/<owner>/blasto:${{ github.sha }}

      - name: Deploy to server
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          echo "${{ secrets.DEPLOY_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} \
            "cd ~/blasto && echo IMAGE_TAG=${{ github.sha }} > .env && docker compose pull && docker compose up -d"
```
- **`needs: checks`** - deploy only after lint, types and tests pass. Must match the checks job's name.
- **`if: ...`** - only on a push to `main`; on pull requests the job shows as *skipped*, which is correct.
- **`permissions`** - the built-in token may read the repo and push packages, nothing more.
- **Build and push** - like the manual push, but tagged with the commit SHA.
- **Deploy** - writes the key and fingerprint from secrets into files on the runner (`chmod 600`: SSH refuses private keys others can read), logs into the server, rewrites `.env` with the new tag, pulls that image, and lets Compose recreate the changed container. Caddy keeps running; only `app` restarts.

### The flow
```
branch → PR → checks run (deploy skipped) → merge → push to main
       → checks → deploy: build, push ghcr.io/<owner>/blasto:<sha>
       → ssh: IMAGE_TAG=<sha> → pull → up -d → new version live
```

Verify a deploy:
```bash
cat ~/blasto/.env            # on the server: IMAGE_TAG=<long sha>
docker compose ps
curl -I https://<domain>/health
```

**(Background: not done in blasto yet - S8)** **Rollback** follows from the SHA tags: set `IMAGE_TAG` back to the previous SHA and `docker compose up -d`. (Database migrations are not undone by this.)

---

## 12. What broke, and why
| Symptom | Cause | Fix |
|---|---|---|
| `RequestDisallowedByAzure` | Region not allowed for the student subscription | Read the allowed list in Policy → Assignments |
| Size `NotAvailableForSubscription` | No capacity for that size in that region/zone | Drop the zone, try another allowed region |
| `docker push` → `denied` | Logged in as user, pushed with `sudo` (root) | Same mode for login and push, or join the `docker` group |
| Deploy job: `permission denied ... docker.sock` | Server user not in the `docker` group | `usermod -aG docker`, re-login, or `sudo` in the deploy command |
| Deploy job "skipped" on a PR | The `if:` limits deploy to pushes to `main` | Expected; merge to deploy |
| `www.<domain>` does not load | Parking `www` record deleted, none added; Caddyfile had no `www` block | CNAME `www` → `<domain>`; `www` redirect block in the Caddyfile |
| (Avoided) app reachable on `:8000` despite firewall | Docker-published ports bypass `ufw` | Publish only the proxy's ports |
| (Avoided) browser shows nothing on `.dev` | HTTPS-only TLD, no certificate yet | Test with `curl` until Caddy has the certificate |

**(Background: not done in blasto)** Debugging order when the site is down: `dig +short <domain>` (DNS) → `nc -vz <ip> 443` from the laptop (firewall) → `docker compose ps` (containers) → `docker compose logs caddy` (certificate, proxy) → `docker compose logs app` (the app).

---

## 13. Checklist: deploying anything to a fresh VPS
1. Create the VM: Ubuntu LTS, x64, SSH key auth, only port 22 open.
2. `ssh` in; `apt update && apt upgrade`; reboot if needed. **(Background: not done in blasto)** Confirm password login is off.
3. Point DNS: remove parking records; A record `@` → server IP; CNAME `www` → the domain. Do this early.
4. Install Docker from Docker's repo; add the user to the `docker` group.
5. Open 80 and 443 in the provider's firewall (or `ufw` if there is no outside firewall).
6. Push the image to a registry; make it public, or log the server in with a read-only token.
7. On the server: folder with `compose.yml` (app without ports, proxy with 80/443, volume for certificates), `Caddyfile` (domain + `www` redirect), `.env` with the tag, `app.env` with the app's settings (`chmod 600`).
8. `docker compose up -d`; check `https://<domain>` from a phone.
9. CD: separate deploy key, secrets (host, user, key, known_hosts), registry permission, deploy job tagged by commit SHA.
10. Push a visible change; confirm it appears without touching the server.

---

## 14. Next levels
**(Background: not done in blasto)** Not needed now, but this is where the setup grows:

- **A limited deploy user.** Today the deploy key logs in as our admin user, so whoever holds it controls the server. Stricter: a separate user that owns only `~/blasto` and is in the `docker` group, and an `authorized_keys` entry restricted to a single command (`command="..."` before the key), so the key can only run the deploy script.
- **Health-checked deploys.** After `up -d`, have the deploy job `curl` the health endpoint and roll back automatically if it fails.
- **Backups.** Once Postgres arrives, the data lives in a volume on this one VM; it needs regular dumps stored elsewhere.
- **Monitoring.** Something outside the server that alerts when `https://<domain>/health` stops answering.
- **Zero-downtime deploys.** Today `app` restarts, so requests during that moment fail. Fixed by running two copies and switching between them (blue-green), which is what orchestrators do.
- **Scaling.** Several app copies behind a load balancer, then several servers, then an orchestrator (Kubernetes) and infrastructure defined in code (Terraform). Every one of those automates a step from this guide.

---

## Glossary
- **ACME** - the protocol Let's Encrypt uses to verify domain control and issue certificates.
- **CA** - certificate authority; signs certificates browsers trust.
- **CI/CD** - automatically checking (CI) and shipping (CD) every change.
- **DNS** - turns names into IP addresses.
- **NAT** - translating one address to another at a network boundary (public IP → VM's private IP).
- **NSG** - Azure's network security group; a firewall outside the VM.
- **Port** - number selecting which program on a machine receives traffic.
- **Registry** - storage for images (GHCR, Docker Hub).
- **Reverse proxy** - server in front of apps; clients talk only to it.
- **SNI** - the part of the TLS handshake where the client says which domain it wants.
- **TLS** - encryption + identity layer under HTTPS.
- **TTL** - how long a DNS answer may be cached.
- **VPS** - virtual private server; a VM with root access.
