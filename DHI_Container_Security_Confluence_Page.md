# Docker Hardened Images (DHI) & Container Security Tooling

---

## Introduction

Docker Hardened Images (DHI) are production-grade container images specifically engineered for security, compliance, and reliability. They represent a shift from "it works" to "it works securely."

**Why DHI exists:**
- **Security**: Standard images ship with hundreds of packages, many containing known vulnerabilities
- **Compliance**: Regulations (SOC2, PCI-DSS, HIPAA) increasingly require demonstrable supply chain security
- **Supply Chain Risk**: Recent attacks (SolarWinds, Log4Shell) highlight the danger of unvetted dependencies

**Key idea**: DHI images are *minimal + secure + production-ready*. They contain only what's necessary to run your application—nothing more.

---

## What Makes an Image "Hardened"

A hardened image is a container image that has been systematically stripped of unnecessary components and configured to minimize security risk.

### Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Reduced Attack Surface** | Fewer packages = fewer potential vulnerabilities |
| **Minimal Packages** | Only runtime dependencies included |
| **No Unnecessary Tools** | No shell, package manager, or debugging utilities |
| **Non-root Execution** | Application runs as unprivileged user |
| **Cryptographic Signing** | Image authenticity verified via signatures |
| **SBOM Included** | Full inventory of all components |
| **Provenance Attestation** | Build process is documented and verifiable |

### Important Security Concepts

<details>
<summary><b>CVEs (Common Vulnerabilities and Exposures)</b></summary>

Publicly disclosed security vulnerabilities. Each CVE has a severity score (CVSS). Hardened images target near-zero CVEs, especially critical/high severity.

</details>

<details>
<summary><b>SBOM (Software Bill of Materials)</b></summary>

A complete inventory of all software components in an image. Required for:
- Vulnerability tracking
- License compliance
- Incident response (e.g., "Are we affected by CVE-XXXX-YYYY?")

</details>

<details>
<summary><b>Supply Chain Security</b></summary>

Ensuring every component—from base image to final artifact—is verified and trustworthy. Includes:
- Signed images
- Verified build pipelines
- Dependency scanning

</details>

---

## Standard Image vs Hardened Image

| Feature | Standard Docker Image | Hardened Image |
|---------|----------------------|----------------|
| **Size** | 100MB – 1GB+ | 10MB – 100MB |
| **Attack Surface** | Large (hundreds of packages) | Minimal (runtime only) |
| **CVEs** | Often 50–500+ known CVEs | Near-zero (0–5 typical) |
| **Tools Included** | Shell, package manager, utilities | Application runtime only |
| **Security Posture** | Default/weak | Production-hardened |
| **Maintenance** | Manual patching, slow updates | Automated, rapid patching |
| **SBOM** | Usually not included | Included and attested |
| **Non-root** | Often runs as root | Non-root by default |

---

## Benefits of Docker Hardened Images

### Security Improvements
- **Near-zero CVEs**: Critical and high vulnerabilities eliminated at source
- **Reduced blast radius**: Attackers have no tools to leverage post-exploitation
- **No shell access**: Common attack techniques (reverse shells, privilege escalation) become impossible

### Compliance Benefits
- Pre-built SBOM satisfies audit requirements
- Signed images provide chain of custody
- Consistent security posture across environments

### Operational Benefits
- **Reduced maintenance overhead**: Vendor handles security updates
- **Faster patching lifecycle**: New CVE? Updated image available within hours
- **Smaller images**: Faster pulls, less storage, quicker deployments
- **Production-ready defaults**: Non-root, read-only filesystem, no setuid binaries

---

## How to Use Docker Hardened Images

### Example: Using DHI in Dockerfile

```dockerfile
# Use hardened Node.js image instead of standard node:18
FROM dhi.io/node:18

# Set working directory
WORKDIR /app

# Copy application files
COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Run as non-root (already configured in DHI base)
CMD ["node", "app.js"]
```

### Key Differences from Standard Images

| Aspect | Standard Image | DHI Image |
|--------|---------------|-----------|
| Debugging | `docker exec -it container sh` | There is no shell. Prevents users or attackers from executing arbitrary commands inside containers |
| Installing packages | `apt-get install` | Disables the ability to install software post-build, reducing drift and exposure|
| File inspection | `cat`, `ls`, `find` | Use `docker cp` or sidecar |

### Limitations & Workarounds

> **Note**: The lack of shell is intentional—it's a security feature, not a bug.

<details>
<summary><b>Debugging in Production</b></summary>

**Problem**: No shell means no `exec` into container

**Solutions**:
1. **Debug sidecar**: Deploy ephemeral debug container sharing the namespace
2. **Logging**: Ensure comprehensive application logging
3. **kubectl debug**: For Kubernetes, use debug containers
4. **Local testing**: Use standard image for development, DHI for production

```bash
# Kubernetes debug container example
kubectl debug -it pod/myapp --image=busybox --target=myapp
```

</details>

<details>
<summary><b>Adding Custom Certificates</b></summary>

**Problem**: Can't modify CA bundle at runtime

**Solution**: Add certificates at build time

```dockerfile
FROM dhi.io/node:18
COPY ./certs/internal-ca.crt /etc/ssl/certs/
```

</details>

---

## How to Create / Build Hardened Images

### Approach 1: Use Official DHI Images (Recommended)

Simplest approach—vendor maintains security posture.

```dockerfile
FROM dhi.io/python:3.11
```

### Approach 2: Build Your Own Hardened Images

For custom applications or when DHI doesn't cover your stack.

#### Multi-stage Build Pattern

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app

# Stage 2: Runtime (minimal)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Tips: Use COPY Instead of ADD When Writing Dockerfiles. The COPY instruction copies files from the local host machine to the container file system.
The ADD instruction can potentially retrieve files from remote URLs and perform unpacking operations. Since ADD could bring in files remotely, 
the risk of malicious packages and vulnerabilities from remote URLs is increased

#### Best Practices for Hardened Builds

| Practice | Implementation |
|----------|---------------|
| **Use multi-stage builds** | Separate build tools from runtime |
| **Choose minimal base** | `distroless`, `alpine`, or `scratch` |
| **Remove unnecessary layers** | Combine RUN commands, clean up in same layer |
| **Run as non-root** | `USER nonroot` or numeric UID |
| **Pin versions** | Use SHA256 digest, not tags |
| **Disable setuid** | Remove setuid/setgid bits |
| **Read-only filesystem** | Set at runtime: `--read-only` |

#### Version Pinning Example

```dockerfile
# Bad: Tag can change
FROM node:18

# Better: Specific version
FROM node:18.19.0

# Best: SHA256 digest (immutable)
FROM node@sha256:abc123def456...
```

### Distroless vs Alpine vs Scratch

| Base Image | Size | Shell | Package Manager | Use Case |
|------------|------|-------|-----------------|----------|
| `scratch` | 0 MB | No | No | Static Go binaries |
| `distroless` | 2-20 MB | No | No | Most compiled languages |
| `alpine` | 5 MB | Yes | Yes (apk) | When shell needed |
| `debian-slim` | 80 MB | Yes | Yes (apt) | Maximum compatibility |

---

## Docker Image Security Scanning Tools

Security scanning tools analyze container images for:

- **Vulnerabilities**: Known CVEs in OS packages and application dependencies
- **SBOM Generation**: Complete inventory of components
- **License Compliance**: Identify problematic licenses (GPL in commercial software)
- **Secrets Detection**: Hardcoded credentials, API keys
- **Misconfigurations**: Dockerfile anti-patterns, insecure settings
- **Remediation Guidance**: Specific fix recommendations

### Tool Categories

| Category | Purpose | Examples |
|----------|---------|----------|
| **Vulnerability Scanning** | Find CVEs | Trivy, Grype, Scout |
| **SBOM Generation** | Inventory components | Syft, Scout |
| **Policy Enforcement** | Block non-compliant images | OPA, Scout Policies |
| **Registry Scanning** | Continuous monitoring | Harbor, Scout |

---

## Docker Scout

Docker Scout is Docker's native security scanning and analysis tool, integrated directly into Docker Desktop and CLI.

### What is Docker Scout?

Docker Scout provides real-time security insights for container images. It continuously monitors your images against updated vulnerability databases and provides actionable remediation guidance.

### Key Features

| Feature | Description |
|---------|-------------|
| **CVE Detection** | Identifies vulnerabilities with severity ratings |
| **SBOM Analysis** | Automatic generation and analysis of software inventory |
| **Base Image Recommendations** | Suggests updated/hardened alternatives |
| **Policy Checks** | Enforce organizational security standards |
| **Remediation Guidance** | Specific fix recommendations, not just problem identification |
| **CI/CD Integration** | GitHub Actions, GitLab CI, Jenkins support |
| **Real-time Monitoring** | Continuous scanning of pushed images |

### CLI Usage Examples

```bash
# Quick overview of image security posture
docker scout quickview my-image:latest

# Detailed CVE analysis
docker scout cves my-image:latest

# Filter by severity
docker scout cves --only-severity critical,high my-image:latest

# Compare two images
docker scout compare my-image:v1 --to my-image:v2

# Get base image update recommendations
docker scout recommendations my-image:latest

# Generate SBOM
docker scout sbom my-image:latest

# Check against policies
docker scout policy my-image:latest
```

### Example Output

```
docker scout quickview myapp:latest

    i New version 1.5.0 available (installed version is 1.4.2)
    ✓ Image stored for analysis
    ✓ SBOM obtained from attestation

  Target     │  myapp:latest
  Base image │  node:18.19.0
  Digest     │  sha256:abc123...

           │ Analyzed Image
───────────┼──────────────────────
  Critical │  0
  High     │  3
  Medium   │  12
  Low      │  24
  Total    │  39

  Base image update available: node:18.19.1
  Updating would fix 2 critical and 5 high vulnerabilities
```

### CI/CD Integration

#### GitHub Actions Example

```yaml
name: Container Security Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Docker Scout Scan
        uses: docker/scout-action@v1
        with:
          command: cves
          image: myapp:${{ github.sha }}
          only-severities: critical,high
          exit-code: true  # Fail pipeline on critical/high CVEs

      - name: Check Policy Compliance
        uses: docker/scout-action@v1
        with:
          command: policy
          image: myapp:${{ github.sha }}
```

#### GitLab CI Example

```yaml
container_scan:
  stage: security
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker scout cves $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --only-severity critical,high --exit-code
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

### Policy Configuration

Docker Scout policies allow you to define and enforce security standards:

```yaml
# .docker/scout-policy.yaml
policies:
  - name: no-critical-cves
    description: Block images with critical CVEs
    rules:
      - type: vulnerability
        severity: critical
        action: deny

  - name: require-non-root
    description: Ensure images run as non-root
    rules:
      - type: config
        check: user
        value: "!root"
        action: deny

  - name: approved-base-images
    description: Only allow approved base images
    rules:
      - type: base-image
        allow:
          - dhi.io/*
          - gcr.io/distroless/*
```

---

## Other Image Scanning Tools

<details>
<summary><b>Trivy</b></summary>

Open-source scanner by Aqua Security. Fast, comprehensive, widely adopted.

```bash
# Install
brew install trivy

# Scan image
trivy image my-image:latest

# Generate SBOM
trivy image --format spdx-json -o sbom.json my-image:latest

# Fail on critical
trivy image --exit-code 1 --severity CRITICAL my-image:latest
```

**Pros**: Fast, good accuracy, active community, Kubernetes integration
**Cons**: No built-in remediation suggestions

</details>

<details>
<summary><b>Grype</b></summary>

Open-source scanner by Anchore. Pairs with Syft for SBOM generation.

```bash
# Install
brew install grype

# Scan image
grype my-image:latest

# Use with Syft SBOM
syft my-image:latest -o json | grype
```

**Pros**: Fast, good accuracy, pluggable architecture
**Cons**: Smaller community than Trivy

</details>

<details>
<summary><b>Snyk</b></summary>

Commercial tool with free tier. Strong developer experience.

```bash
# Scan image
snyk container test my-image:latest

# Monitor continuously
snyk container monitor my-image:latest
```

**Pros**: Excellent remediation guidance, IDE integrations, developer-friendly
**Cons**: Commercial (limited free tier), requires account

</details>

### Tool Comparison

| Feature | Docker Scout | Trivy | Grype | Snyk |
|---------|-------------|-------|-------|------|
| **Cost** | Free tier + paid | Free | Free | Freemium |
| **Installation** | Built into Docker | Standalone | Standalone | Standalone |
| **Speed** | Fast | Very fast | Fast | Moderate |
| **SBOM** | Yes | Yes | Via Syft | Yes |
| **Remediation** | Yes | No | No | Yes |
| **CI/CD** | Yes | Yes | Yes | Yes |
| **Kubernetes** | Limited | Yes | Yes | Yes |

---

## Best Practices

### Image Selection & Building

- [ ] **Use minimal base images**: Prefer DHI, distroless, or alpine
- [ ] **Pin image versions**: Use SHA256 digests for reproducibility
- [ ] **Multi-stage builds**: Separate build-time and runtime dependencies
- [ ] **Run as non-root**: Always specify `USER nonroot` or numeric UID
- [ ] **Avoid latest tag**: Explicit versions prevent unexpected changes

### Scanning & Monitoring

- [ ] **Scan in CI/CD**: Block deployments with critical CVEs
- [ ] **Scan continuously**: Images safe today may have new CVEs tomorrow
- [ ] **Generate SBOMs**: Required for incident response and compliance
- [ ] **Set quality gates**: Define acceptable risk thresholds

### Pipeline Integration

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Build     │───▶│    Scan     │───▶│   Policy    │───▶│   Deploy    │
│   Image     │    │   (Scout)   │    │   Check     │    │   (if pass) │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                          │                  │
                          ▼                  ▼
                   ┌─────────────┐    ┌─────────────┐
                   │   SBOM      │    │   Block if  │
                   │   Generated │    │   Critical  │
                   └─────────────┘    └─────────────┘
```

### Operational Recommendations

| Area | Recommendation |
|------|----------------|
| **Development** | Use standard images for dev/debug, DHI for staging/prod |
| **CI/CD** | Fail builds on critical/high CVEs |
| **Monitoring** | Re-scan deployed images weekly (CVE databases update daily) |
| **Incident Response** | Maintain SBOM inventory for rapid CVE impact assessment |
| **Compliance** | Automate SBOM generation and attestation |

---

## Conclusion

Container security is not optional—it's a fundamental requirement for production systems. The combination of Docker Hardened Images and security scanning tools creates a robust DevSecOps foundation.

### Key Takeaways

1. **Start with hardened images**: DHI or distroless should be the default, not the exception
2. **Scan continuously**: A clean scan today doesn't mean clean tomorrow
3. **Automate everything**: Security gates in CI/CD prevent vulnerable code from reaching production
4. **Generate SBOMs**: Know exactly what's in your images for compliance and incident response
5. **Accept the tradeoffs**: Less convenience (no shell) = more security

### Recommended Adoption Path

```
Phase 1: Enable scanning in CI/CD (awareness)
    ↓
Phase 2: Set quality gates (enforcement)
    ↓
Phase 3: Migrate to DHI base images (hardening)
    ↓
Phase 4: Continuous monitoring & policy (maturity)
```

> **Bottom line**: Adopt hardened images by default. The slight inconvenience in debugging is vastly outweighed by the security benefits. Your future incident response team will thank you.

---

## References

- [Docker Hardened Images Documentation](https://docs.docker.com/hardened-images/)
- [Docker Scout Documentation](https://docs.docker.com/scout/)
- [Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [SLSA Supply Chain Framework](https://slsa.dev/)
- [NIST SBOM Guidelines](https://www.ntia.gov/SBOM)

---

*Last updated: April 2026*
*Maintainer: DevOps/Security Team*
