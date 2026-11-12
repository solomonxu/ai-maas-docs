# Contributing

[English](./CONTRIBUTING.en.md) | [简体中文](./CONTRIBUTING.md)

Thank you for contributing to OpenMaas. The project is in open-source closeout; Issues, documentation improvements, and code contributions are welcome.

## Communication

- Questions and proposals: open an Issue (Bug / Feature / Docs).
- Security vulnerabilities: see [SECURITY.en.md](./SECURITY.en.md); do not disclose them publicly.
- Before the code repository is public, use an Issue or Discussion to align on the approach.

## Branches and Commits

- `main`: stable branch;
- `feat/*`, `fix/*`, `docs/*`: feature, fix, and documentation branches;
- commit format: `<type>(<scope>): <subject>`;
- keep each PR focused and include motivation, changes, impact, and verification.

## Code Contributions

After the code repository is public, follow the service boundaries, reuse the common runtime SDK, prefer in-house frontend components, and run the relevant unit, architecture, typecheck, build, and i18n checks.

## Documentation Contributions

- keep Chinese and English versions aligned;
- prefer Markdown/Mermaid or SVG for diagrams;
- put images under `images/` with semantic names;
- never include internal IPs, domains, credentials, tokens, private keys, or customer information.

## Development Logs

Logs are generated from code commits and reviewed by maintainers. English logs must be rewritten for quality, not produced by glossary substitution. Any sensitive hit stops publication.

## License

Documentation contributions are released under [Apache-2.0](./LICENSE). The code repository license is defined in that repository.
