# Security Policy

[English](./SECURITY.en.md) | [简体中文](./SECURITY.md)

## Supported scope

AI MaaS is in **M0 (under development)** and is **not committed to be production-ready**. Security fixes will be addressed on a best-effort basis, **without any SLA**.

## Reporting a vulnerability

We follow **responsible disclosure**. Please **do not** disclose unfixed security issues in public issues.

- Preferred: report privately via **GitHub Security Advisories** (repository → Security → Report a vulnerability).
- Please include: impact, reproduction steps, proof-of-concept (if any), affected version/commit, and your contact.
- We will acknowledge and follow up within a reasonable time, and coordinate public disclosure after a fix.

## Red lines (do not commit)

To protect deployers, it is **strictly forbidden** to commit to this public repository:

- Details of unfixed vulnerabilities, exploit code (PoC), or raw security scan reports;
- Internal IPs/domains/ports, database/object-storage/message-queue credentials, keys or tokens;
- Customer information, contracts or trade secrets, internal meeting notes.

## Notes for deployers

- This platform targets **private/on-prem deployment**. In production: rotate default credentials/secrets, enable network isolation and TLS, and run with least privilege.
- When integrating third-party **base models**, you are responsible for evaluating and complying with their licenses and regulations.
