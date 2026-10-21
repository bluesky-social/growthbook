# Bluesky GrowthBook Node security patch, 2026-10-21

`Dockerfile.node-security` derives from the exact image deployed on Bluesky's
`kubes-1` cluster. It replaces Node 24.22.0 with the official Node 24.22.1
Debian 12 binary. Both inputs use multi-platform digest pins. Application
code, Python, native modules, the non-root user, and the inherited startup
command stay intact. The old Node executable is removed from the resulting
filesystem, and the build checks the replacement and native module loading.

This is a Bluesky derivative, not a new DHI release. Inherited DHI metadata
and base-image attestations describe the original base, not the patched
filesystem. The October 21 Node advisory requires the runtime update;
changing TypeScript or updating OS CA packages alone does not fix it.

Build the production architecture locally:

```sh
docker buildx build --platform linux/amd64 --load \
  -f Dockerfile.node-security \
  -t growthbook-node-security:d5d5772-node24.22.1 .
```

Publish only after runtime/native-module and isolated application startup
checks pass. Use a dedicated Bluesky ECR repository, not the Docker Hub
pull-through cache. Record the resulting image digest in
`bluesky-social/deploy:apps/growthbook/deployment.yaml` after reviewing
`kubectl --context kubes-1 diff`. Argo Application `eks1-growthbook` watches
that directory on `main`, with automated sync and self-heal enabled and
pruning disabled. There is no release-triggered manifest update for this app.

The deployment has no readiness probe, so pod readiness alone is not a
sufficient rollout check. Verify the API `/healthcheck`, frontend, SDK feature
serving, Node version, and image digest after the rollout. Keep the existing
CA distrust workaround. Returning to the previous image would reintroduce
the Node vulnerabilities; treat that only as an emergency availability
rollback.

Retire this derivative once an upstream GrowthBook image with Node 24.22.1
or a later patched Node 24 release is verified and deployed.

Advisory: https://nodejs.org/en/blog/vulnerability/october-2026-security-releases

## Validation before publication

The amd64 build passed on 2026-10-21. A network-isolated runtime test verified
Node 24.22.1, OpenSSL 3.5.9, UID 1000, removal of the original Node executable,
a Brotli round trip, Kerberos loading, an actual `isolated-vm` evaluation,
and Python `gbstats`/NumPy/SciPy imports. Against disposable local MongoDB
8.2.2, the unchanged application startup command served HTTP 200 from both
port 3100 `/healthcheck` and port 3000 `/`.

Roast's independent finder/verifier review reported no confirmed findings.
Production SSO, warehouse integrations, and existing SDK feature payloads
still require post-deployment checks; local smoke tests do not cover them.
