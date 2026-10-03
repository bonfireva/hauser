# Maintaining the custom Hauser build

The `custom` branch starts at upstream `v0.33.0`, matching the deployed release.
`main` is reserved for upstream code. The fork is `bonfireva/hauser`; the
`upstream` remote is `ralleur/hauser`.

## Custom commits

- Daytime light glows: placed lamps follow device state during daylight too.
- Custom image workflow and this maintenance guide.

Keep behavior changes in separate commits so they can be reapplied or removed
when an upstream release changes the same feature. Household images, runtime
configuration, backups and credentials are not committed.

## Build a release

Push `custom` to run tests, typechecking, production/demo builds, dependency
audit, and metadata/license checks. To publish an image, push a numbered tag
whose upstream version matches `app/package.json`:

```bash
git tag custom-v0.33.0-1 custom
git push origin custom-v0.33.0-1
```

The custom workflow publishes `ghcr.io/bonfireva/hauser:custom-v0.33.0-1` for
`linux/amd64`. It also runs the container contract and records the image digest
in its summary. Use that digest for deployment after every job succeeds.
No workflow connects to Proxmox or deploys automatically. Upstream version
metadata stays unchanged; the custom tag and image labels identify our build.
Do not push upstream `v*` tags to the fork: they trigger upstream release jobs.

GHCR packages may initially be private. To pull without credentials on the
deployment host, make only this source-only image package public in GitHub's
package settings. Never put cluster keys or household data in GitHub Actions.

## Adopt an upstream release

```bash
git fetch upstream --tags
git log --oneline v0.33.0..custom
git switch -c update/v0.39.0 v0.39.0
```

Start the update branch from the exact fetched release tag rather than an
untagged main commit. Cherry-pick only the custom commits in order (not upstream
commits or merge commits), fix conflicts, and update
this guide's base version. Do not cherry-pick an old customization if the
upstream release already implements its behavior.

Push the update branch for review. After review, move `custom` to the tested
update branch with a regular merge, preserving history; do not force-push.
Choose a fresh custom tag, such as `custom-v0.39.0-1`, and wait for the full
custom workflow to pass before deploying. The branch quality job runs when the
merge reaches `custom`.

## Deploy and roll back

The existing deployment is managed by Docker Compose inside a Proxmox LXC.
Inspect the active Compose file and mounts before changing it; preserve the
existing Compose project name, environment file, ports, and persistent volumes.

1. Retain the current image digest and copy the active Compose/environment files
   into a local backup directory outside this source repository.
2. Take a consistent backup of `/config`, `/data`, and `/assets`; stop Hauser
   briefly while copying them. Keep credentials in those backups private.
3. Pull the verified custom image by digest before stopping the old container.
4. Change only the image reference in the active deployment and recreate the
   Hauser service with the existing Compose project and volumes.
5. Check health, Home Assistant connection, saved room images, and daytime
   light glows. Reload clients when idle so the PWA picks up the new frontend.

To roll back, restore the previous image reference and recreate the service.
For updates that migrate persistent data, restore the matching pre-update
data backup as well. Do not remove volumes with `docker compose down -v`.
