# Security Policy

[English](./SECURITY.en.md) | [简体中文](./SECURITY.md)

## Supported Scope

OpenMaas is in open-source closeout and is pre-1.0. Security fixes are handled on a best-effort basis; no production SLA is provided.

## Reporting a Vulnerability

Please follow responsible disclosure and do not report unfixed vulnerabilities in public Issues. Prefer GitHub Security Advisories and include impact, reproduction steps, proof of concept if available, affected versions/commits, and contact information.

## Red Lines

Do not submit to the public repository:

- details or exploits for unfixed vulnerabilities;
- internal IPs, domains, ports, database/object-storage/message-queue credentials, keys, or tokens;
- customer information, contracts, trade secrets, or internal meeting notes;
- personal or otherwise non-public data.

## Deployment Guidance

OpenMaas is designed for self-hosted deployment. In production, replace default credentials, enable TLS and network isolation, and run with least privilege. External models, datasets, images, and runtimes remain subject to their own licenses and compliance requirements.
