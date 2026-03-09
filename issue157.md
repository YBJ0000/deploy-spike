## Goal

Research how to deploy `medical-server` with the open-source, self-hosted version of [Dokploy](https://dokploy.com/), and deliver a workable setup.

This research is specifically for a self-hosted Dokploy deployment, not a paid hosted solution.

## Scope

- Review the app's deployment/runtime requirements.
- Determine the Dokploy deployment approach that fits this repo.
- Define the required Dokploy configuration, infrastructure dependencies, and app settings.
- Identify any blockers or follow-up work needed before rollout.

## Deliverable

- A short write-up describing the recommended self-hosted Dokploy setup for `medical-server`
- The concrete configuration needed to run it in Dokploy
- Required environment variables, networking, storage, domains/TLS, and dependent services
- Any known risks, gaps, or follow-up tasks

## Done When

- There is one clear self-hosted Dokploy deployment approach for `medical-server`.
- The setup is specific enough to implement without further discovery.
- Risks or blockers are documented.

## References

- Dokploy: https://dokploy.com/
- Current release workflow: `.github/workflows/release-main.yml`
- Current runtime notes: `app/README.md`
