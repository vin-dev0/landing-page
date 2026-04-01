+++
title = "Lock Down Your Stack: Automating NPM Security with Trivy"
date = "2026-03-29"
draft = false
description = "A comprehensive guide on securing Node.js applications by scanning dependencies, secrets, and infrastructure misconfigurations using Trivy."
+++

In the modern JavaScript ecosystem, security isn't just about your code—it’s about the massive web of dependencies you pull in every time you run `npm install`. 

**Trivy**, developed by Aqua Security, has emerged as the "Swiss Army Knife" of security scanning. While famous for container scanning, its ability to scan local filesystems, Node.js packages, and Infrastructure as Code (IaC) makes it indispensable for securing the entire software development lifecycle.

---

## 1. Getting Started with Trivy
Before scanning, you need the CLI. You can install it via most package managers or run it via Docker to keep your local environment clean.

### Installation
* **macOS:** `brew install trivy`
* **Ubuntu/Debian:** `sudo apt-get install trivy`

### Running via Docker (No Install)
```bash
docker run --rm -v $(pwd):/src aquasec/trivy fs /src
```

## 2. Scanning NPM Packages for Vulnerabilities
The most common entry point for attackers in Node.js is a compromised or outdated dependency. Trivy scans your `package-lock.json` or `yarn.lock` to identify known CVEs (Common Vulnerabilities and Exposures).

### Basic Filesystem Scan
To scan your current project directory for vulnerable packages:
```bash
trivy fs .
```

### Filtering by Severity
In a large project, you might be overwhelmed by "Low" severity warnings. You can filter to focus on what actually matters:
```bash
trivy fs --severity HIGH,CRITICAL .
```

### Including Development Dependencies
By default, Trivy focuses on production dependencies. If you want to check your build tools and test suites, use the following flag:
```bash
trivy fs --include-dev-deps .
```

## 3. Beyond Packages: Scanning for Secrets and Misconfigs
Vulnerabilities aren't just in your `node_modules`. Hardcoded API keys and loose Dockerfile permissions are just as dangerous. Trivy can look for these simultaneously.

```bash
trivy fs --scanners vuln,secret,misconfig .
```

*   **vuln**: Checks for CVEs in packages.
*   **secret**: Looks for AWS keys, GitHub tokens, and certificates.
*   **misconfig**: Checks your Dockerfile or kubernetes.yaml for security best practices (e.g., "Do not run as root").

## 4. Generating a Software Bill of Materials (SBOM)
Transparency is a modern development requirement. An SBOM is a formal record of every library and version used in your app. Trivy can generate these in industry-standard formats like CycloneDX.

```bash
trivy fs --format cyclonedx --output sbom.json .
```

You can then "scan" this SBOM later or share it with security teams to prove your application's health.

## 5. Integrating into CI/CD
Security is a "fail-fast" game. You can configure Trivy to return a non-zero exit code if it finds a Critical vulnerability, effectively breaking your build before it reaches production.

### GitHub Actions Snippet
```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    ignore-unfixed: true
    format: 'table'
    exit-code: '1' # Fails the build if vulnerabilities are found
    severity: 'CRITICAL'
```
