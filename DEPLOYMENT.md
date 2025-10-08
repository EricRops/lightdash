**Production Deployment**
To be compeleted if we decide to adopt LightDash.


**Local Testing Flow**
# Terminal 1: Backend (watches for changes)
cd ../Repos/lightdash-fork
export PATH="/Users/ericrops/Documents/Repos/lightdash-fork/venv/bin:$PATH"
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGPASSWORD=password
export PGDATABASE=postgres
export LIGHTDASH_SECRET='not very secret'
export BIGQUERY_TENANT_DATA_PROJECT=combined-tenant-data-project
export BIGQUERY_DATASET_PREFIX=tenant
export NARVAR_TENANT_ID=narvar
pnpm -F backend dev

# Terminal 2: Frontend (watches for changes)  
pnpm -F frontend dev

# Local postgres:
psql -h localhost -p 5432 -U postgres -d postgres



**Deployment to Compute Engine, Docker Compose**
1. Make changes locally in the lightdash fork. 
2. Local test to verify that the changes still yield a successful build:

    Individual Module Build Commands
    `pnpm -F @lightdash/common build`
    `pnpm -F @lightdash/warehouses build`
    `pnpm -F backend build`
    `pnpm -F frontend build`

    OR, Full Build Command (All Modules at once)
    `pnpm build`
    This runs all packages sequentially in dependency order: common → warehouses → backend → frontend.

    Linting check: run linter across all packages, auto-fix issues where possible:
    `pnpm run fix-lint`

3. Push the changes to the remote github repo

4. SSH into the VM

  - `git pull origin impersonation-poc`
  - `docker compose down`

**6A - Build new image with code change but use cache**
  - `docker compose build lightdash`
  - `docker compose up --detach`

**6B - Brand new image option**

  # Do a full docker cleanup before (optional)
  - `docker builder prune -af`
  - `docker image prune -a`
  - `docker system prune -a --volumes`
  - `sudo systemctl restart docker`
  - `sudo systemctl status docker`

  # Create/edit Docker daemon config:
  `sudo nano /etc/docker/daemon.json`

  # Add this configuration:
  {
    "max-concurrent-downloads": 3,
    "max-concurrent-uploads": 3,
    "builder": {
      "gc": {
        "enabled": true,
        "defaultKeepStorage": "20GB"
      }
    }
  }

  ```
  DOCKER_BUILDKIT=1 \
  BUILDKIT_STEP_LOG_MAX_SIZE=50000000 \
  BUILDKIT_STEP_LOG_MAX_SPEED=10000000 \
  docker compose build --no-cache lightdash
  ```
  
  - `docker compose up --detach`
  - `docker compose logs -f lightdash`


