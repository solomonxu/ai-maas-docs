# Changelog

OpenMaas follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and semantic versioning.

## [Unreleased]

Open-source closeout. The public repository and documentation site are coming soon.

### Added

- Self-hosted platform for base models, datasets, training, artifacts, inference, evaluation, compression, image management, and resource governance.
- Training-to-inference pipeline with preflight snapshots, stage state machines, database leases, retries, cancellation, cost attribution, templates, and triggers.
- WebSocket runtime control plane for Python pullers, trainers, inference workers, compressors, scanners, and node agents.
- Tenant-aware platform services, audit, notifications, approvals, and budget-aware cost governance.
- Vue 3 web console with goal-first pipeline flows and operational resource views.

### Changed

- Public documentation now targets OpenMaas rather than the earlier AI MaaS name.
- Historical `PLAN.md` content is archived under `docs/archive/`.
- Internal engineering notes and UI review checklists are no longer part of the public documentation tree.

### Fixed

- Documentation links, deployment guidance, and public security/contribution pages have been aligned with the current project state.
