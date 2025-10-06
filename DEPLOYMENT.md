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
  # Stop current containers
  `docker compose down`

5. Optional - If previous build was corrupted or failed, can do a full docker cleanup before rebuilding
  - docker builder prune -af
  - docker image prune -a
  - docker system prune -a --volumes
  - sudo systemctl restart docker
  - sudo systemctl status docker

5. Pull in the changes to the VM, rebuild the image, and deploy
  - `git pull origin impersonation-poc`
  - `docker compose build lightdash`  # Can use --no-cache flag to build from scratch
  - `docker compose up --detach`
  # Check logs (optional)
  - `docker compose logs -f lightdash`


