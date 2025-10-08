**Production Deployment**
To be compeleted if we decide to adopt LightDash.


**Rapid Local Testing Deployment Flow**
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
  # Stop current containers
  - `docker compose down`

5. Optional - If previous build was corrupted or failed, can do a full docker cleanup before rebuilding
  - `docker builder prune -af`
  - `docker image prune -a`
  - `docker system prune -a --volumes`
  - `sudo systemctl restart docker`
  - `sudo systemctl status docker`

6A - Brand new image option

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

6B - Build new image with code change but use cache

