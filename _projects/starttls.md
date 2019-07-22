---
title: STARTTLS Everywhere
links:
  - <a href='https://starttls-everywhere.org/' target='_blank'>Live site</a>
  - <a href='https://github.com/EFForg/starttls-backend' target='_blank'>Code on GitHub</a>
thumb:
  filename: starttls
  alt: A hopping cartoon kangaroo with an @ symbol on its belly
description: A tool in Go for verifying the security of an e-mail server and building a list of MTAs that support the MTA-STS standard.
weight: 100
---

The goal of EFF's STARTTLS Everywhere project is to improve the security of communcation between mailservers. It has a few parts:

1. It provides a tool to run security checks against email domains. It checks for STARTTLS support, a secure version of TLS, a valid TLS certificate.

2. It checks for support of MTA-STS, a standard designed to prevent downgrade attacks between mail servers, and provides feedback on the configuration.

3. It allows mailserver admins to add their domain to a preload list for MTA-STS.

4. It tracks MTA-STS adoption among the top million domains by regularly conducting scans.

The STARTTLS Everywhere API and checker tool are written in Go.
