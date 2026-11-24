# 💚 Develop Customer Statements (dunning) on your Mac

# 💚 Develop Customer Statements (dunning) on your Mac
## 💛 What you get
- `make dev`: the web app on http://localhost:8080 with hot reload, and the API on http://localhost:8081.
- `make up`: the exact production image on http://localhost:8080.
- The same checks the PR runs, in about 30 seconds.
- A merge to `main` is live on dunning.chartmetric.com within about 15 minutes, with no infra ticket.
## 💛 Before you start: the checklist
An **Apple Silicon** Mac. The lockfile only carries native binaries for Apple Silicon and Linux, so Intel Macs and Windows can't install.
Access to **chartmetric/dunning** on GitHub, with `gh` signed in.
**Docker Desktop**, only if you want `make up`.
An **AWS SSO profile the documents bucket admits**: Kaylee's `product-engineer` session, or `infra-admin`. Anyone else gets AccessDenied on statement PDFs, so ask infra to add you to the bucket's `dev_users` list.
The app's **DATABASE_URL** (login `customer_statements_app` on prod2) and the pod's **restricted Stripe key** (`rk_live_...`), from infra by DM. Never paste either one in a channel.
A network path to prod2. Today prod2 answers from the internet; once it's closed off, you'll need the VPN.
## 💛 Step 1: Install the exact toolchain
The repo pins **Node 24.22.1** and **pnpm 10.34.9**, and `pnpm install` refuses anything else. Homebrew's `node` (24.1.0) and `node@24` (24.22.3) are both refused. You also need `flock`, which macOS doesn't ship.
```bash
brew install fnm flock
fnm install 24.22.1
fnm default 24.22.1
echo 'eval "$(fnm env)"' >> ~/.zshrc && eval "$(fnm env)"
npm i -g pnpm@10.34.9
node -v && pnpm -v && which flock   # v24.22.1, 10.34.9, and a path to flock
```
No fnm? The route we ran end to end
## 💛 Step 2: Clone and install
```bash
gh repo clone chartmetric/dunning
cd dunning
pnpm install --frozen-lockfile   # about 15 seconds with a warm cache
```
You should see `Supported toolchain: Node 24.22.1, pnpm 10.34.9`. Ignore the "Update available ... 12.x" banner: the pin stays.
## 💛 Step 3: Fill in .env
```bash
cp .env.example .env
```
Then edit `.env`:
Then sign in, and make sure no static AWS keys are around:
```bash
aws sso login --profile <your-profile>
make env-check       # refuses a .env that holds AWS keys
env | grep '^AWS_'   # should print nothing: no keys exported in your shell
```
No AWS keys, ever: not in `.env` and not exported in your shell. Static keys are the one credential SES accepts besides the pod's own role. Email is also locked in code, but keep both layers. `env-check` catches `AWS_ACCESS_KEY_ID=` lines, not an `export` line or keys already in your shell, hence the `env` check.
## 💛 Step 4: Run it
```bash
make dev
```
About 7 seconds later you have:
- **Web** on http://localhost:8080 (Vite, with hot reload)
- **API** on http://localhost:8081 (no hot reload: stop and rerun `make dev` after a server change)
The first run also downloads the RDS certificate bundle into `certs/`. Check it from another terminal:
```bash
curl -s localhost:8080/api/healthz   # {"status":"ok"}
curl -s localhost:8080/api/access    # {"id":"you@chartmetric.com","role":"super_admin"}
```
### 🤍 The production image: make up
```bash
aws sso login --profile <your-profile>   # the container reuses your ~/.aws, read-only
make up     # builds the Dockerfile (about 75 seconds), then serves http://localhost:8080
make down
```
Wait about 10 seconds after it says it's listening before you curl it.
## 💛 Step 5: Run the same checks as the PR
The PR check runs exactly these. The tests mock the database, but the code needs `DATABASE_URL` set to something, so use a placeholder and never your real URL:
```bash
export DATABASE_URL=postgres://nobody:x@127.0.0.1:1/none
pnpm run typecheck:libs                                      # first, or the next two fail with TS6305
pnpm --filter @workspace/api-server run typecheck
pnpm --filter @workspace/customer-statements run typecheck
pnpm --filter @workspace/api-server run test:unit            # 116 tests, about 7 seconds
pnpm --filter @workspace/customer-statements run test:unit   # 56 tests, about 6 seconds
```
One test file, the full build and the root test script
## 💛 Step 6: Ship a change
1. Branch, commit, push and open a PR. The **PR check** runs the typechecks, both unit suites and an amd64 Docker build.
1. Get a review and merge to `main`. Always go through a PR, even though the ruleset that enforces it isn't switched on yet.
1. The **Deploy** workflow runs by itself: it builds the image, applies new migrations to prod2, points the `live` tag at the new image and restarts the two pods. Watch it under Actions, Deploy.
1. It's live on https://dunning.chartmetric.com within about 15 minutes. App alerts go to #infra-alarms.
To roll back, run Deploy again with an earlier commit's **full** SHA:
```bash
gh workflow run deploy.yml -R chartmetric/dunning -f sha=<full 40-character sha>
```
### 🤍 Adding a database migration
- Add `lib/db/migrations/NNNN_short_name.sql`. Each file runs once, in filename order, on the next deploy.
- Use `IF NOT EXISTS` everywhere, and wrap a normal file in `BEGIN;` and `COMMIT;`.
- A file containing the word "concurrently" runs one statement at a time: no `BEGIN`, and no `;` inside strings.
- Migrations run while the old pods are still serving, so keep them backward compatible: add first, rename or drop later.
- Mirror the change in `lib/db/src/schema/statements.ts`. Never run `drizzle-kit push` or `pnpm --filter @workspace/db run push`.
- Tables belong to the role `customer_statements_owner`. Anything beyond the app's own tables (roles, grants, other schemas) is the Data team's call, so ask in #team-data-engineering.
### 🤍 Changing the API contract
Edit `lib/api-spec/openapi.yaml`, run `pnpm --filter @workspace/api-spec run codegen` and commit the generated files. A response that no longer matches the schema comes back as an HTTP 400.
## 💛 Safety rules
**Your laptop is part of production while make dev runs.**
Never run these while the real DATABASE_URL is in your shell or `.env`:
- migrations (`dist/migrate.mjs`): they run on deploy
- `pnpm --filter @workspace/db run push` or `scripts/post-merge.sh`
- the full API suite, `pnpm --filter @workspace/api-server run test`: it inserts and deletes rows
- the performance suites, `reconcile`, or `apply-chartmetric-hq.ts --apply`
Customer emails can't go out: sending stays locked in code until the billing owner unlocks it in a reviewed PR.
The Replit-era files are wrong in places. `replit.md`, `docs/SETUP.md`, `threat_model.md` and `.replit` still describe Clerk, `db push` and port 5000, so trust this page and the Makefile instead. `.agents/` holds Replit agent notes that include customer billing exports: don't share it.
