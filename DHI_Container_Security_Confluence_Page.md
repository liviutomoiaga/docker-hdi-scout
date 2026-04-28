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

## dhictl - Docker Hardened Images CLI

`dhictl` is the official command-line tool for managing Docker Hardened Images. It allows you to browse the DHI catalog, mirror images to your registry, and create customizations.

### Key Features

| Feature | Description |
|---------|-------------|
| **Catalog Browsing** | Explore available DHI images, filter by type/name, view metadata including CVE counts |
| **Image Mirroring** | Mirror DHI images to your Docker Hub repository |
| **Image Customization** | Create custom variations of DHI base images |
| **Build Monitoring** | Track customization builds and access build logs |

### Installation

#### Step 1: Download the Binary

Download the appropriate binary for your platform from the [releases page](https://github.com/docker-hardened-images/dhictl/releases).

**Linux (amd64) - Ubuntu/Debian/WSL2:**
```bash
curl -L -o dhictl https://github.com/docker-hardened-images/dhictl/releases/download/v0.0.2/dhictl-linux-amd64
chmod +x dhictl
sudo mv dhictl /usr/local/bin/
```

**Linux (arm64):**
```bash
curl -L -o dhictl https://github.com/docker-hardened-images/dhictl/releases/download/v0.0.2/dhictl-linux-arm64
chmod +x dhictl
sudo mv dhictl /usr/local/bin/
```

**macOS (Apple Silicon):**
```bash
curl -L -o dhictl https://github.com/docker-hardened-images/dhictl/releases/download/v0.0.2/dhictl-darwin-arm64
chmod +x dhictl
sudo mv dhictl /usr/local/bin/
```

**macOS (Intel):**
```bash
curl -L -o dhictl https://github.com/docker-hardened-images/dhictl/releases/download/v0.0.2/dhictl-darwin-amd64
chmod +x dhictl
sudo mv dhictl /usr/local/bin/
```

**Windows (PowerShell):**
```powershell
Invoke-WebRequest -Uri "https://github.com/docker-hardened-images/dhictl/releases/download/v0.0.2/dhictl-windows-amd64.exe" -OutFile "dhictl.exe"
Move-Item dhictl.exe C:\Windows\
```

Verify installation:
```bash
dhictl version
```

#### Step 2 (Optional): Enable `docker dhi` Command

By default, `dhictl` is a standalone CLI tool. To use it as a Docker plugin (i.e., `docker dhi ...`), copy it to the Docker CLI plugins directory:

```bash
mkdir -p ~/.docker/cli-plugins
cp /usr/local/bin/dhictl ~/.docker/cli-plugins/docker-dhi
chmod +x ~/.docker/cli-plugins/docker-dhi
```

Now you can use either:
- `dhictl <command>` (standalone)
- `docker dhi <command>` (as Docker plugin)

### Common Commands

```bash
# View available DHI images
dhictl catalog list

# Filter catalog by image name
dhictl catalog list --name python

# View image details and tags
dhictl catalog show python

# Mirror an image to your Docker Hub repository
dhictl mirror start python:3.11

# List active mirrors
dhictl mirror list

# Stop mirroring
dhictl mirror stop <mirror-id>

# Prepare a customization template
dhictl customization prepare python:3.11

# List customization builds
dhictl customization build list

# View build logs
dhictl customization build logs <build-id>
```

### Output Formats

```bash
# JSON output for scripting/automation
dhictl catalog list --output json

# Enable shell completion (bash)
source <(dhictl completion bash)
```

### Configuration File

Store default settings in a config file to avoid repeating flags:

| Platform | Config Path |
|----------|-------------|
| Linux/macOS | `$HOME/.config/dhictl/config.yaml` |
| Windows | `%USERPROFILE%\.config\dhictl\config.yaml` |

Example config:
```yaml
org: my-docker-org
api_token: dhr_xxxxxxxxxxxx
```

Environment variables (`DHI_ORG`, `DHI_API_TOKEN`) override config file settings.

---

## Customize DHI Images (DHI Enterprise)

DHI Enterprise allows organizations to create **customized versions** of Docker Hardened Images. This is useful when you need to add specific packages, certificates, or configurations while maintaining the security posture of the base DHI image.

### Why Customize?

| Use Case | Example |
|----------|---------|
| **Add packages** | Install additional runtime dependencies not in base image |
| **Internal certificates** | Add corporate CA certificates for internal services |
| **Compliance requirements** | Include specific security agents or monitoring tools |
| **Regional configurations** | Add locale-specific packages or timezone data |

### Customization Workflow

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   1. Prepare    │───▶│   2. Create     │───▶│   3. Build      │───▶│   4. Use        │
│   YAML scaffold │    │   Customization │    │   (automatic)   │    │   Custom Image  │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Step 1: Prepare Customization Scaffold

Generate a YAML template for your customization:

```bash
docker dhi customization prepare python 3.11 \
  --org my-org \
  --destination my-org/python-custom \
  --name "Python with extras" \
  --output python-custom.yaml
```

This creates a YAML file like:

```yaml
# python-custom.yaml
source:
  image: python
  version: "3.11"
destination:
  repository: my-org/python-custom
name: "Python with extras"
packages:
  - curl
  - ca-certificates
# Add custom packages, certificates, etc.
```

### Step 2: Create the Customization

Submit the customization to DHI:

```bash
docker dhi customization create python-custom.yaml --org my-org
```

### Step 3: Monitor the Build

DHI automatically builds your customized image. Monitor progress:

```bash
# List builds for your customization
docker dhi customization build list my-org/python-custom "Python with extras" --org my-org

# Get build details
docker dhi customization build get my-org/python-custom "Python with extras" <build-id> --org my-org

# View build logs
docker dhi customization build logs my-org/python-custom "Python with extras" <build-id> --org my-org
```

### Step 4: Use Your Custom Image

Once built, use your customized image:

```dockerfile
FROM my-org/python-custom:3.11
WORKDIR /app
COPY . .
CMD ["python", "app.py"]
```

### Managing Customizations

```bash
# List all customizations
docker dhi customization list --org my-org

# Get customization details (export to YAML)
docker dhi customization get my-org/python-custom "Python with extras" \
  --org my-org \
  --output python-custom.yaml

# Update an existing customization (YAML must include 'id' field)
docker dhi customization edit python-custom.yaml --org my-org

# Delete a customization
docker dhi customization delete my-org/python-custom "Python with extras" --org my-org
```

### Mirroring DHI Images (DHI Select & Enterprise)

Mirror DHI images to your Docker Hub organization:

```bash
# Start mirroring an image
docker dhi mirror start --org my-org -r python:3.11,my-org/python

# Include dependent images
docker dhi mirror start --org my-org -r python:3.11,my-org/python --dependencies

# List mirrored repositories
docker dhi mirror list --org my-org

# Stop mirroring (keeps existing images)
docker dhi mirror stop python --org my-org

# Stop mirroring and delete repositories
docker dhi mirror stop python --org my-org --delete
```

### Enterprise Package Repository

Generate credentials for accessing enterprise APK packages:

```bash
docker dhi auth apk
```

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

### Policy Configuration

Docker Scout policies are **declarative rules** that define your organization's security standards. When you run `docker scout policy <image>`, Scout evaluates the image against these rules and reports pass/fail.

```
Image → Scout Analysis → Policy Check → Pass/Deny
```

#### Conceptual Policy Example

The following YAML shows the structure and intent of Docker Scout policies:

```yaml
# .docker/scout-policy.yaml (conceptual example)
policies:
  - name: no-critical-cves
    description: Block images with critical CVEs
    rules:
      - type: vulnerability
        severity: critical
        action: deny          # Fails if ANY critical CVE exists

  - name: require-non-root
    description: Ensure images run as non-root
    rules:
      - type: config
        check: user
        value: "!root"
        action: deny          # Fails if image runs as root

  - name: approved-base-images
    description: Only allow approved base images
    rules:
      - type: base-image
        allow:
          - dhi.io/*          # Allow any DHI image
          - gcr.io/distroless/*
        # Fails if base image is NOT in this list
```

#### How to Configure Policies

> **Note**: The YAML example above is conceptual. Docker Scout policies are configured through:

| Method | Description |
|--------|-------------|
| **Docker Scout Dashboard** | Web UI for Docker Business/Team subscribers at https://scout.docker.com → Organization → Policies |
| **Built-in Policies** | Predefined policies: no critical/high CVEs, supply chain attestations, up-to-date base images |
| **CLI Evaluation** | `docker scout policy myimage:latest` to check against configured policies |

#### CLI Usage

```bash
# Check image against all configured policies
docker scout policy myimage:latest

# Fail command if policy violated (useful for CI/CD)
docker scout policy myimage:latest --exit-code
```

#### CI/CD Integration

Add policy checking to your GitHub Actions workflow:

```yaml
- name: Docker Scout Policy Check
  uses: docker/scout-action@v1
  with:
    command: policy
    image: myapp:${{ github.sha }}
    exit-code: true  # Fail workflow if policy violated
```


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
│   Build     │───▶│    Scan     │───▶│   Policy    │───▶│   Deploy   │
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

## Example: Secure Release Flow with DHI + Scout

This example demonstrates a practical release pipeline combining Docker Hardened Images and Docker Scout for a security-focused release cycle.

### Pipeline Overview

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  1. DEVELOP  │───▶│  2. PR GATE  │───▶│  3. BUILD    │───▶│  4. MONITOR  │
│  DHI Base    │    │  Scout Check │    │  SBOM+Sign   │    │  Continuous  │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       │                   │                   │                   │
       ▼                   ▼                   ▼                   ▼
  Near-zero CVEs     Block if new       Signed image +      Re-scan for
  from the start     vulnerabilities    full attestation    new CVEs
```

### Step 1: Use DHI Base Images

Start with hardened base images to minimize vulnerabilities from the beginning:

```dockerfile
# Standard image: 150+ CVEs typical
FROM python:3.11              # ❌ Not recommended

# Hardened image: 0-5 CVEs typical
FROM dhi.io/python:3.11       # ✅ Recommended
```

Mirror DHI images to a private registry:

```bash
dhictl mirror start --org my-org -r python:3.11,my-org/python
```

### Step 2: PR Security Gate

Configure Scout to compare pull request images against production:

```yaml
- name: Docker Scout Compare
  uses: docker/scout-action@v1
  with:
    command: compare
    image: ${{ steps.meta.outputs.tags }}
    to-env: production
    only-severities: critical,high
    exit-code: true  # Block PR if new vulnerabilities detected
```

**Result**: Pull requests introducing new critical/high CVEs are automatically blocked.

### Step 3: Build with SBOM and Provenance

Generate Software Bill of Materials and provenance attestation on merge:

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v7
  with:
    sbom: true          # Generate SBOM
    provenance: true    # Sign with provenance
    push: true
    tags: ${{ steps.meta.outputs.tags }}
```

### Step 4: Pre-Release Policy Check

Verify policy compliance before release approval:

```bash
docker scout policy myapp:v2.1.0 --exit-code
```

Example output:
```
✓ Policy "no-critical-cves" - PASSED
✓ Policy "require-sbom" - PASSED
✓ Policy "approved-base-images" - PASSED
```

### Release Checklist

| Check | Command | Pass Criteria |
|-------|---------|---------------|
| Zero critical CVEs | `docker scout cves --only-severity critical` | Count = 0 |
| High CVEs under limit | `docker scout cves --only-severity high` | Count ≤ 2 |
| SBOM attached | Build with `sbom: true` | Attestation present |
| Policy compliance | `docker scout policy --exit-code` | Exit code 0 |
| No regression | `docker scout compare --to-env production` | No new critical/high |

### Continuous Monitoring

Schedule weekly re-scans of production images:

```bash
# Quick security posture check
docker scout quickview myapp:latest

# Check for base image updates
docker scout recommendations myapp:latest
```

**Key outcomes**:
- Security gates automated in CI/CD
- SBOM enables rapid CVE impact assessment
- Audit evidence generated with every build
- New vulnerabilities detected before they reach production

---

## References

- [Docker Hardened Images Documentation](https://docs.docker.com/dhi/)
- [Docker Scout Documentation](https://docs.docker.com/scout/)
- [Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [SLSA Supply Chain Framework](https://slsa.dev/)
- [NIST SBOM Guidelines](https://www.ntia.gov/SBOM)

---

*Last updated: April 2026*
*Maintainer: DevOps/Security Team*
