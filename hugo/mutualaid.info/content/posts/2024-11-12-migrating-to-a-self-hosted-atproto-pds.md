---
title: "Migrating to a self-hosted atproto PDS"
draft: true
---

Convenient way to set up a PDS on Ubuntu here https://github.com/bluesky-social/pds

Wanted to do it on OpenBSD for fun (also run this web server and a Matrix server running Dendrite on here)

Published a role to Ansible Galaxy that installs and sets up an atproto PDS on OpenBSD https://galaxy.ansible.com/ui/repo/published/brianreumere/software/content/role/pds/

A lot of steps from the container installer script https://github.com/bluesky-social/pds/blob/main/installer.sh

All available env vars are here https://github.com/bluesky-social/atproto/blob/main/packages/pds/src/config/env.ts

Most defaults for non-required env vars are here https://github.com/bluesky-social/atproto/blob/main/packages/pds/src/config/config.ts

Only need to set a subset of these...but I don't know exactly which

Uses the index.js from the pds repo as an entrypoint (I don't know js/ts at all so this was an easy place to start)

rc.d file for pds

---

Not sure what the recovery DID key setting is for

I don't think the mod service, AppView, or report variables are required for just PDS hosting https://github.com/bluesky-social/atproto/pull/1198

Learned about PDS Entryway https://docs.bsky.app/docs/advanced-guides/entryway

Redis is not supported by the Ansible role
