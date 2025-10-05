**Production Deployment**
To be compeleted if we decide to adopt LightDash.


**Rapid Local Testing Deployment Flow**
1. Make changes in locally in the lightdash fork. 
2. Verify that the changes still yield a successful build:

    Individual Module Build Commands
    `pnpm -F @lightdash/common build`
    `pnpm -F @lightdash/warehouses build`
    `pnpm -F backend build`
    `pnpm -F frontend build`

    Full Build Command (All Modules)
    `pnpm build`
    This runs all packages sequentially in dependency order: common → warehouses → backend → frontend.

3. Push the changes to the remote github repo
