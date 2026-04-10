<p align="center">

  <h1 align="center">
    Awesome PyPI Security Best Practices
  </h1>
  <p align="center">
    A curated and practical list of security best practices for using Python packages from PyPI.
  </p>

<!-- Shields -->
<p align="center">
 <img src="https://cdn.rawgit.com/sindresorhus/awesome/d7305f38d29fed78fa85652e3a63e154dd8e8829/media/badge.svg" alt="Awesome" />
 <img src="https://badgen.net/badge/total%20best%20practices/17/blue" alt="PyPI security best practices" />
 <img src="https://badgen.net/badge/Last%20Update/Apr%209/green" />
 <a href="https://www.github.com/lirantal/pypi-security-best-practices" target="_blank">
  <img src="https://badgen.net/badge/PyPI/Security Best Practices/purple" alt="PyPI Security Best Practices"/>
 </a>
</p>
<!-->

</p>

**Scope**:
- safe-by-default Python package manager command-line options
- hardening against supply chain attacks on PyPI
- deterministic and secure dependency resolution
- security vulnerability scanning and package health signals
- instructions for the `uv` and `pip` package managers

**Context**: The LiteLLM/Telnyx[^1] (119k+ malicious downloads in under 3 hours, 40-50% of installs unpinned), Ultralytics[^2] and other incidents are a growing concern of supply chain security attacks and compromised PyPI packages. Follow these developer security best practices around PyPI, package maintenance and secure local development to mitigate security risks.

---

## Table of Contents

**PyPI Security Best Practices:**

- 1 [Prefer Binary-Only Installs](#1-prefer-binary-only-installs)
- 2 [Install with Cooldown](#2-install-with-cooldown)
  - 2.1 [uv exclude-newer cooldown](#21-uv-exclude-newer-cooldown)
  - 2.2 [pip uploaded-prior-to cooldown](#22-pip-uploaded-prior-to-cooldown)
  - 2.3 [Dependabot and Renovate cooldowns](#23-dependabot-and-renovate-cooldowns)
- 3 [Pin Dependencies with Hash Verification](#3-pin-dependencies-with-hash-verification)
  - 3.1 [uv lockfile hash verification](#31-uv-lockfile-hash-verification)
  - 3.2 [pip require-hashes verification](#32-pip-require-hashes-verification)
- 4 [Use Deterministic Installations](#4-use-deterministic-installations)
- 5 [Prevent Dependency Confusion](#5-prevent-dependency-confusion)
- 6 [Scan for Known Vulnerabilities](#6-scan-for-known-vulnerabilities)
  - 6.1 [pip-audit for vulnerability scanning](#61-pip-audit-for-vulnerability-scanning)
  - 6.2 [uv-secure for lockfile scanning](#62-uv-secure-for-lockfile-scanning)
- 7 [Harden Package Installs with Security Tools](#7-harden-package-installs-with-security-tools)

**Secure Local Development Best Practices:**

- 8 [No Plaintext Secrets in .env Files](#8-no-plaintext-secrets-in-env-files)
- 9 [Work in Dev Containers](#9-work-in-dev-containers)

**PyPI Maintainer Security Best Practices:**

- 10 [Enable 2FA for PyPI Accounts](#10-enable-2fa-for-pypi-accounts)
- 11 [Publish with Trusted Publishing (OIDC)](#11-publish-with-trusted-publishing-oidc)
- 12 [Publish with Package Attestations](#12-publish-with-package-attestations)
- 13 [Secure Your CI/CD Release Pipeline](#13-secure-your-cicd-release-pipeline)
- 14 [Reduce Your Package Dependency Tree](#14-reduce-your-package-dependency-tree)

**PyPI Package Health Best Practices:**

- 15 [Generate and Track SBOMs](#15-generate-and-track-sboms)
- 16 [Consult Vulnerability Databases for Package Health](#16-consult-vulnerability-databases-for-package-health)
- 17 [Verify Published Package Contents](#17-verify-published-package-contents)

---

## 1. Prefer Binary-Only Installs

> [!WARNING]
> Source distributions (`sdist`) can execute arbitrary code during installation via `setup.py`, making them a common attack vector for supply chain attacks. Unlike pre-built wheels, source distributions require a build step that runs untrusted code on your machine.

Python packages are distributed in two forms: source distributions (sdists) that require a build step and may execute `setup.py`, and pre-built binary wheels that install without executing arbitrary code. Malicious packages frequently exploit the `setup.py` execution during `sdist` installs to exfiltrate credentials, install backdoors, or perform other malicious activities[^3].

> [!TIP]
> **Security Best Practice**: Configure your package manager to prefer or require pre-built binary wheels, avoiding source distribution builds that execute arbitrary code during installation.

> [!NOTE]
> **How to implement?**
>
> With uv (default behavior), uv prefers wheels over source distributions by default. To strictly enforce binary-only installs:
> ```bash
> $ uv pip install --only-binary :all: <package-name>
> ```
>
> Or set it persistently in `uv.toml` or `pyproject.toml`:
> ```toml
> # uv.toml or [tool.uv.pip] in pyproject.toml
> [pip]
> only-binary = true
> ```
>
> With pip:
> ```bash
> $ pip install --only-binary :all: <package-name>
> ```
>
> Or set it globally via environment variable:
> ```bash
> $ export PIP_ONLY_BINARY=:all:
> ```

Not all packages provide pre-built wheels for every platform. If a build is required, use `--no-build-isolation` with a pre-audited build environment rather than allowing arbitrary `setup.py` execution from untrusted sources.

---

## 2. Install with Cooldown

> [!WARNING]
> Newly released packages and versions may contain malicious code that is often quickly discovered by the community within hours or days and subsequently yanked or removed. The LiteLLM supply chain attack[^1] was live on PyPI for 2 hours and 32 minutes before being quarantined — during which 119,000+ downloads occurred, many from unpinned installations fetching the latest version.

Attackers exploit PyPI's versioning and publishing model to push malicious versions of packages. During the LiteLLM incident, **40-50% of all installs were unpinned** and fetching the latest version on each invocation — leaving very little time for researchers and PyPI admins to report, triage, and quarantine malware[^1]. By implementing a "cooldown" period before installing or upgrading to new package versions, you reduce the risk of installing compromised packages that may be quickly discovered and removed from the index.

> [!TIP]
> **Security Best Practice**: Configure your package manager to delay installations of recently published packages, allowing time for the community to discover and report potential security issues. PyPI recommends a minimum of 3 days (`P3D`); consider 7 days for general use and 30 days for production deployments requiring higher assurance[^1][^4][^5].

> [!NOTE]
> **How to implement?**
>
> With uv, set `exclude-newer` in your project configuration:
> ```toml
> # pyproject.toml
> [tool.uv]
> exclude-newer = "7 days"
> ```
>
> Or apply it globally in your user-level configuration:
> ```toml
> # ~/.config/uv/uv.toml (macOS/Linux)
> # %APPDATA%\uv\uv.toml (Windows)
> exclude-newer = "7 days"
> ```
>
> Or pass it on the command line:
> ```bash
> $ uv lock --exclude-newer "7 days"
> $ uv sync --exclude-newer "7 days"
> ```
>
> Or as an environment variable (useful for CI/containers):
> ```bash
> $ export UV_EXCLUDE_NEWER="7 days"
> ```

`exclude-newer` accepts friendly durations (`"7 days"`, `"1 week"`, `"30 days"`), ISO 8601 durations (`"P7D"`), or RFC 3339 timestamps (`"2026-03-20T00:00:00Z"`).

uv also supports **per-package overrides** via `exclude-newer-package` if you need to exempt specific packages from the cooldown (e.g., to pull in an urgent security patch without disabling the cooldown globally):

```toml
# pyproject.toml
[tool.uv]
exclude-newer = "7 days"
exclude-newer-package = { requests = "1 day" }
```

> [!CAUTION]
> When `exclude-newer` filters out a package version, uv does not mention it in error messages. If you encounter unexpected resolution failures, temporarily removing the cooldown can help diagnose whether a recent release is required[^6].

### 2.1. uv exclude-newer cooldown

The `exclude-newer` setting in uv creates a dependency cooldown that ignores packages published within the specified time window. The LiteLLM supply chain attack was discovered and quarantined within 2 hours and 32 minutes[^1] — even a 3-day cooldown would have been more than sufficient to avoid the compromised versions entirely.

```toml
# pyproject.toml
[tool.uv]
exclude-newer = "P3D"  # "3 days" in RFC 3339 format
```

Use it with `uv pip compile` to generate pinned requirements with cooldown:

```bash
$ uv pip compile --exclude-newer "3 days" requirements.in -o requirements.txt
```

### 2.2. pip uploaded-prior-to cooldown

Starting with pip v26.1, you can set **relative** dependency cooldowns directly in your `pip.conf` file, matching uv's ergonomics[^1][^15]:

```ini
# ~/.config/pip/pip.conf (macOS/Linux)
[install]
uploaded-prior-to = P3D
```

With pip v26.0, only absolute timestamps are supported, typically paired with `date` for a relative offset:

```bash
$ python -m pip install \
  --uploaded-prior-to=$(date -d '-3days' -Idate) \
  <package-name>
```

For containers or ephemeral CI environments, set an absolute date via environment variable:

```bash
$ export PIP_UPLOADED_PRIOR_TO="2026-03-27"
```

> [!TIP]
> To **bypass the cooldown** for urgent security patches, set `--uploaded-prior-to=P0D` to get the actual latest release:
> ```bash
> $ python -m pip install --uploaded-prior-to=P0D <package-name>==26.3.31
> ```

> [!NOTE]
> Dependency cooldowns should be paired with a vulnerability scanning strategy (see [section 6](#6-scan-for-known-vulnerabilities)) so security updates aren't delayed. Dependabot and Renovate both bypass cooldowns by default for security updates[^1].

### 2.3. Dependabot and Renovate cooldowns

Automated dependency update tools support cooldown periods:

**Dependabot** has a [`cooldown`](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference#cooldown-) configuration option for delaying updates to new versions by a configurable number of days.

**Renovate bot** has a [`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage) config option:

```json
{
  "minimumReleaseAge": "7 days"
}
```

---

## 3. Pin Dependencies with Hash Verification

> [!WARNING]
> Without cryptographic hash verification, an attacker who compromises a package index, mirror, or network path can substitute a malicious package with the same version number. Hash verification detects any modification to package contents.

Pinning dependencies to exact versions is necessary but not sufficient — version pinning alone cannot detect if a package's contents have been tampered with. Cryptographic hash verification creates SHA-256 checksums that are validated at install time. If a package is modified — even under the same version number — installation fails immediately[^7].

> [!TIP]
> **Security Best Practice**: Pin all dependencies with cryptographic hashes using `uv lock` or `uv pip compile --generate-hashes` to ensure that package contents are verified against known-good checksums at every install.

### 3.1. uv lockfile hash verification

uv's lockfile (`uv.lock`) automatically includes cryptographic hashes for all resolved dependencies. Using `uv sync` or `uv pip install` with a lockfile ensures hash verification by default:

```bash
# Generate a lockfile with hashes (automatic)
$ uv lock

# Install from lockfile with hash verification (automatic)
$ uv sync
```

For requirements-based workflows, generate hashes explicitly:

```bash
$ uv pip compile --generate-hashes requirements.in -o requirements.txt
```

This produces a requirements file with embedded hashes:

```
requests==2.32.3 \
    --hash=sha256:70761cfe03c773ceb22aa2f671b4757976145175cdfca038c02654d061d6dcc6
```

### 3.2. pip require-hashes verification

With pip, use `--require-hashes` to enforce that every package in a requirements file includes a hash:

```bash
$ pip install --require-hashes -r requirements.txt
```

Generate hashed requirements using `pip-compile` from `pip-tools`:

```bash
$ pip-compile --generate-hashes requirements.in -o requirements.txt
```

> [!CAUTION]
> **Anti-pattern**:
> Avoid installing dependencies without hash verification in production environments:
> ```bash
> $ pip install -r requirements.txt  # No hash verification
> ```

---

## 4. Use Deterministic Installations

> [!WARNING]
> Using `uv sync` or `pip install` without a frozen lockfile can lead to inconsistent installations, potentially introducing unintended package versions that differ from what was tested and reviewed.

Deterministic installations ensure that the exact same set of packages — same versions, same hashes — are installed in every environment. This prevents the case where a dependency resolver picks up a newly published (and potentially malicious) version during install time.

> [!TIP]
> **Security Best Practice**: Use frozen lockfile installations in CI/CD and production environments to ensure only the exact versions specified in the lockfile are installed, and abort if any inconsistency is detected.

> [!NOTE]
> **How to implement?**
>
> With uv, use `--frozen` to enforce strict lockfile adherence:
> ```bash
> $ uv sync --frozen
> ```
>
> This will fail if `uv.lock` is out of date with `pyproject.toml`, preventing accidental resolution of new versions.
>
> With pip, use a hashed requirements file:
> ```bash
> $ pip install --require-hashes --no-deps -r requirements.txt
> ```
>
> The `--no-deps` flag prevents pip from resolving additional transitive dependencies not listed in the requirements file.

### Lockfile management best practices

Ensure proper lockfile management across your development workflow:

**Commit lockfiles to version control:**
- `uv.lock` (uv)
- `requirements.txt` with hashes (pip-compile/uv pip compile)
- `pylock.toml` (pip lock — experimental, [PEP 751](https://peps.python.org/pep-0751/))
- `Pipfile.lock` (pipenv)
- `pdm.lock` (pdm)
- `poetry.lock` (poetry)

> [!CAUTION]
> **`pip freeze` does NOT create a lockfile.** It only records packages and their versions — it does not include checksums or hashes. Without hashes, there is no protection against package tampering. Use `uv lock`, `pip-compile --generate-hashes`, or `pip lock` (experimental) instead[^1].

---

## 5. Prevent Dependency Confusion

> [!WARNING]
> When using both private/internal and public package indexes, an attacker can publish a malicious package on PyPI with the same name as an internal package but with a higher version number, causing the public malicious package to be installed instead of the legitimate internal one.

Dependency confusion attacks exploit how package managers resolve packages across multiple indexes. If a package manager searches public PyPI after (or alongside) a private index, an attacker can "shadow" internal package names by publishing identically-named packages on the public registry.

> [!TIP]
> **Security Best Practice**: Use uv's first-match index strategy to ensure that internal packages are always resolved from your private index, preventing dependency confusion attacks. Explicitly pin packages to specific indexes using `[tool.uv.sources]`[^8].

> [!NOTE]
> **How to implement?**
>
> With uv, configure index priority in `pyproject.toml`:
> ```toml
> # pyproject.toml
>
> # Internal index is checked first — if a package exists here,
> # uv will never look for it on PyPI
> [[tool.uv.index]]
> name = "internal"
> url = "https://pypi.internal.corp.com/simple"
>
> # Public PyPI is the fallback
> [[tool.uv.index]]
> name = "pypi"
> url = "https://pypi.org/simple"
> ```
>
> For even stronger guarantees, pin specific packages to specific indexes:
> ```toml
> [tool.uv.sources]
> my-internal-package = { index = "internal" }
> ```
>
> Or mark an index as `explicit` so it's only used for packages that explicitly reference it:
> ```toml
> [[tool.uv.index]]
> name = "internal"
> url = "https://pypi.internal.corp.com/simple"
> explicit = true  # Only used for packages pinned via [tool.uv.sources]
> ```

By default, uv stops at the first index on which a given package is available (the `first-index` strategy). This is secure-by-default — if a package exists on your internal index, it will always be installed from that index, never from PyPI[^8].

> [!NOTE]
> With pip, dependency confusion is harder to prevent. pip searches all configured indexes and picks the highest version by default. Use `--index-url` (singular, not `--extra-index-url`) to restrict to a single index, or maintain a private mirror that proxies PyPI with namespace restrictions.

---

## 6. Scan for Known Vulnerabilities

> [!WARNING]
> Your dependencies may contain known security vulnerabilities (CVEs) that expose your application to exploitation. Without automated scanning, these vulnerabilities go undetected until an attacker exploits them.

Vulnerability scanning tools query advisory databases (OSV, PyPA, GitHub, NVD) to identify known CVEs in your dependency tree. Running these scans in CI/CD on every commit ensures that vulnerabilities are caught before they reach production[^7].

> [!TIP]
> **Security Best Practice**: Integrate `pip-audit` or `uv-secure` into your CI/CD pipeline to automatically detect known vulnerabilities in your Python dependencies on every commit.

### 6.1. pip-audit for vulnerability scanning

[pip-audit](https://github.com/pypa/pip-audit) queries the OSV database (aggregating PyPA, GitHub, and NVD advisories) to identify known CVEs in your dependency tree:

```bash
# Scan using uv as the runner
$ uvx pip-audit --requirement requirements.txt

# Output in JSON for programmatic use
$ uvx pip-audit --format json --requirement requirements.txt > audit-report.json
```

Integrate into GitHub Actions:

```yaml
# .github/workflows/security.yml
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uvx pip-audit --requirement requirements.txt
```

### 6.2. uv-secure for lockfile scanning

[uv-secure](https://pypi.org/project/uv-secure/) scans `uv.lock` files, PEP 751 `pylock.toml` files, and `requirements.txt` files for dependencies with known vulnerabilities:

```bash
# Install as a tool
$ uv tool install uv-secure

# Scan your project's lockfile
$ uv-secure
```

Configure via `pyproject.toml`:

```toml
# pyproject.toml
[tool.uv-secure.vulnerability_criteria]
severity = "medium"
ignore_vulnerabilities = ["CVE-2024-12345"]
aliases = true

[tool.uv-secure.maintainability_criteria]
max_package_age = "P1000D"
forbid_deprecated = true
```

---

## 7. Harden Package Installs with Security Tools

> [!WARNING]
> You should never install PyPI packages without assessing their security posture. Malicious packages can execute arbitrary code during installation, exfiltrate sensitive data, or introduce vulnerabilities without your knowledge.

Installing a new ad-hoc Python package can expose your system to supply chain attacks. Many attacks target popular PyPI packages, exploit typosquatting, or introduce malicious code in `setup.py` scripts that execute during the installation process.

> [!TIP]
> **Security Best Practice**: Use [Socket Firewall (`sfw`)](https://socket.dev/blog/introducing-socket-firewall) as a real-time firewall that intercepts `pip install` and `uv pip install` commands and blocks packages flagged for malicious behavior, using Socket's deep package analysis and threat intelligence.

> [!NOTE]
> **How to implement?**
>
> Install `sfw` globally:
> ```bash
> $ npm install -g sfw
> ```
>
> Socket Firewall Free runs in wrapper mode. Prefix your package manager command with `sfw`:
> ```bash
> $ sfw pip install requests
> $ sfw uv pip install flask
> ```
>
> Socket Firewall will block the package fetch/install if a package is flagged, and prompt you with details so you can make an informed decision.

#### What Socket Firewall checks

Socket Firewall performs deep analysis on packages using Socket's threat intelligence, checking for:

- **Malicious code detection**: Identifies packages that contain known malware or obfuscated code
- **Install script risks**: Flags packages with suspicious `setup.py` or build scripts
- **Typosquatting detection**: Catches packages with names similar to popular packages
- **Dependency confusion**: Detects potential dependency confusion attacks
- **Known vulnerabilities**: Cross-references CVE databases for disclosed vulnerabilities
- **Network and filesystem access**: Highlights packages that perform unexpected network or disk operations

You can also use Datadog's open-source [Supply-Chain Firewall (SCFW)](https://github.com/DataDog/supply-chain-firewall) which transparently wraps `pip install` and blocks known malicious packages[^9].

### Report suspicious packages

If you encounter a suspicious package on PyPI, use the ["Report project as malware"](https://blog.pypi.org/posts/2024-03-06-malware-reporting-evolved/) feature on the package's PyPI page. During the LiteLLM incident, PyPI received 13 inbound reports from concerned users through this feature, accelerating review and quarantine. PyPI maintains a pool of trusted reporters whose reports add weight and can trigger automated quarantine[^1].

---

## 8. No Plaintext Secrets in .env Files

> [!WARNING]
> Storing secrets in plaintext environment variables or `.env` files creates a significant security risk, making sensitive data easily accessible to attackers who successfully launch supply chain attacks or gain access to your system.

Environment variables and `.env` files are commonly used to store API keys, database passwords, and tokens. However, these secrets are stored in plaintext and can be easily exfiltrated by malicious PyPI packages or compromised dependencies. The LiteLLM supply chain attack[^1] specifically targeted environment variables, SSH keys, and cloud credentials for exfiltration.

> [!TIP]
> **Security Best Practice**: Use secrets management solutions that only store references in environment variable data and require additional authentication to access the actual secret values just-in-time.

> [!CAUTION]
> **Anti-pattern**:
> Avoid storing plaintext secrets in `.env` files:
> ```bash
> DATABASE_PASSWORD=my-secret-password
> API_KEY=sk-1234567890abcdef
> ```

> [!NOTE]
> **How to implement?**
>
> Step 1: Use secret references in `.env` files:
> ```bash
> DATABASE_PASSWORD=op://vault/database/password
> API_KEY=infisical://project/env/api-key
> ```
> Step 2: Use the secret manager CLI to inject secrets at runtime:
> ```bash
> $ op run -- python manage.py runserver
> ```

### Follow-up secure secrets resources

- 1Password's [Secrets Automation with 1Password CLI](https://developer.1password.com/docs/cli/get-started/)
- Infisical's [Getting Started with Infisical CLI](https://infisical.com/blog/stop-using-env-files)

---

## 9. Work in Dev Containers

> [!WARNING]
> Running `pip install` or `uv sync` directly on your host development machine exposes your entire system to potential malware, allowing malicious packages to access sensitive files, environment variables, and system resources.

[Development containers](https://code.visualstudio.com/docs/devcontainers/containers) (dev containers) provide an isolated, sandboxed environment that limits the blast radius of supply chain attacks. When malicious Python packages execute during installation or at import time, they are confined to the container environment rather than having access to your entire host system.

> [!TIP]
> **Security Best Practice**: Use dev containers to isolate your project's local development workflows from your host system so that Python package execution is limiting the potential impact of supply chain attacks and malicious package behavior.

> [!NOTE]
> **How to implement?**
>
> Create a `.devcontainer/devcontainer.json` file in your project:
> ```json
> {
>   "name": "Python Dev Container",
>   "image": "mcr.microsoft.com/devcontainers/python:3.12",
>   "features": {
>     "ghcr.io/devcontainers/features/1password:1": {}
>   },
>   "postCreateCommand": "uv sync --frozen"
> }
> ```
>
> Use VS Code to open your project in the dev container.

### Follow-up resources

- Consider further hardening of the Dev Container:
```jsonc
  "runArgs": [
    "--security-opt=no-new-privileges:true",
    "--cap-drop=ALL",
    "--cap-add=CHOWN",
    "--cap-add=SETUID",
    "--cap-add=SETGID"
  ],
```

---

## 10. Enable 2FA for PyPI Accounts

> [!WARNING]
> PyPI accounts without two-factor authentication are vulnerable to credential theft and account takeover attacks, potentially allowing malicious actors to publish compromised versions of your packages. The LiteLLM incident[^1] demonstrated how compromised maintainer credentials can lead to malicious package uploads reaching millions of users.

The LiteLLM/Telnyx supply chain attack[^1] in March 2026 showed the devastating impact of compromised credentials — 119,000+ downloads of malicious versions occurred during the 2.5-hour exposure window before community detection triggered quarantine. Two-factor authentication provides essential protection against credential theft and account takeover.

> [!TIP]
> **Security Best Practice**: Enable two-factor authentication on **all accounts associated with open source development** — not just PyPI, but also GitHub, GitLab, Codeberg, and your email provider. Use phishing-resistant hardware security keys for the strongest protection[^1][^10].

> [!NOTE]
> **How to implement?**
>
> 1. Sign in to [PyPI](https://pypi.org/manage/account/) and navigate to your account settings
> 2. Under "Two factor authentication (2FA)", add a security key (WebAuthn) or authentication application (TOTP)
> 3. Provision recovery codes as a backup
>
> PyPI recommends setting up **at least two** supported 2FA methods. Hardware security keys (FIDO2/WebAuthn) are preferred over TOTP applications as they are resistant to phishing attacks. [PyPI has required 2FA for publishing since the beginning of 2024](https://blog.pypi.org/posts/2024-01-01-2fa-enforced/), but enabling phishing-resistant 2FA like a hardware key provides further protection[^1].

> [!NOTE]
> PyPI checks passwords against the [Have I Been Pwned](https://haveibeenpwned.com/) database and will warn you if your password has appeared in known data breaches. Always use a unique, strong password for your PyPI account[^10].

---

## 11. Publish with Trusted Publishing (OIDC)

> [!WARNING]
> Long-lived PyPI API tokens can be compromised, accidentally exposed in logs, or provide persistent unauthorized access if stolen. The TeamPCP campaign[^1] demonstrated how stolen credentials enable direct malicious uploads to PyPI, bypassing CI/CD workflows entirely.

Trusted Publishing eliminates the need for long-lived PyPI API tokens by using OpenID Connect (OIDC) authentication from your CI/CD environment. This approach uses short-lived, cryptographically-signed tokens that expire within 15 minutes and are scoped to your specific workflow and repository[^11].

> [!TIP]
> **Security Best Practice**: Configure Trusted Publishing for your packages to eliminate token-based authentication risks and automatically generate provenance attestations. Over 50,000 PyPI projects already use Trusted Publishing, and more than 20% of all uploads are now done via trusted publishers[^12].

> [!NOTE]
> **How to implement?**
>
> Step 1: Configure a Trusted Publisher on PyPI:
> 1. Go to your project's settings on [pypi.org](https://pypi.org)
> 2. Under "Publishing", add a new trusted publisher
> 3. Enter your GitHub repository, workflow file name, and environment name
>
> Step 2: Update your GitHub Actions workflow:
> ```yaml
> # .github/workflows/publish.yml
> jobs:
>   publish:
>     runs-on: ubuntu-latest
>     environment: release
>     permissions:
>       contents: read
>       id-token: write
>     steps:
>       - uses: actions/checkout@v4
>       - uses: astral-sh/setup-uv@v5
>       - run: uv build
>       - uses: pypa/gh-action-pypi-publish@release/v1
> ```

### Security considerations for Trusted Publishing

- Use a dedicated GitHub Actions **environment** (e.g., `release`) with required reviewers to prevent unauthorized publishes[^11]
- Use **per-job** `permissions` (not workflow-level) to limit elevated credentials to only the publish job[^11]
- Treat your Trusted Publishers as if they are API tokens — weaknesses in CI/CD workflows that you register as Trusted Publishers can be equivalent to credential compromise[^11]
- When offboarding a project maintainer, review and remove any Trusted Publishers they may have registered[^11]

Trusted Publishing also provides a valuable signal to downstream users through [Digital Attestations](https://docs.pypi.org/attestations/). Users can detect when a release *hasn't* been published using the typical release workflow, drawing more scrutiny to potentially compromised versions[^1].

Trusted Publishing supports GitHub Actions, GitLab CI/CD (including self-managed instances), Google Cloud Build, and ActiveState[^12].

---

## 12. Publish with Package Attestations

> [!WARNING]
> Packages without attestations cannot be verified for their build origin or authenticity, making it difficult for users to trust that a package was built from the intended source code by an authorized CI/CD pipeline rather than by a malicious actor.

Package attestations (PEP 740[^13]) provide cryptographic proof linking published artifacts to source repositories and specific build workflows via Sigstore transparency logs. This establishes a verifiable chain of custody from source code to published package. As of March 2026, over 132,360 packages on PyPI include attestations[^14].

> [!TIP]
> **Security Best Practice**: Generate attestations for your packages using `gh-action-pypi-publish` v1.11.0+ with Trusted Publishing. Attestations are generated automatically when using Trusted Publishing and provide users with verifiable build provenance[^7][^14].

> [!NOTE]
> **How to implement?**
>
> Attestations are generated automatically when publishing via `gh-action-pypi-publish` with Trusted Publishing:
> ```yaml
> # .github/workflows/publish.yml
> jobs:
>   publish:
>     runs-on: ubuntu-latest
>     environment: release
>     permissions:
>       contents: read
>       id-token: write
>       attestations: write
>     steps:
>       - uses: actions/checkout@v4
>       - uses: astral-sh/setup-uv@v5
>       - run: uv build
>       - uses: pypa/gh-action-pypi-publish@release/v1
> ```
>
> Verify attestations for an installed package:
> ```bash
> $ python -m pypi_attestations verify <package-name>
> ```

Attestations use Sigstore's OIDC-based signing, eliminating risks of key loss or compromise that were common with PGP signing[^13][^14].

---

## 13. Secure Your CI/CD Release Pipeline

> [!WARNING]
> A compromised CI/CD pipeline can be used to publish malicious package versions even when Trusted Publishing is enabled. The LiteLLM attack[^1] originated from a compromised Trivy GitHub Action in the CI/CD pipeline, which cascaded into credential theft and unauthorized package uploads.

Your CI/CD release pipeline is a critical part of your supply chain. A compromised action, unpinned dependency, or overly permissive workflow can give attackers the ability to inject malicious code into your published packages[^3].

> [!TIP]
> **Security Best Practice**: Pin all GitHub Actions to commit SHAs, audit workflows with `zizmor`, start with minimal permissions, and require manual approval for release workflows[^3].

> [!NOTE]
> **How to implement?**
>
> **Pin GitHub Actions to commit SHAs** (not mutable tags or branches):
> ```yaml
> # Good: pinned to specific commit SHA
> - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
> - uses: astral-sh/setup-uv@d4b2f3b6ecc6e67c4457f6d3e41ec42d3d0fcb86 # v5.4.2
> - uses: pypa/gh-action-pypi-publish@7f25271a4aa483500f742f9492b2ab5648d61011 # v1.12.4
> ```
>
> **Start with empty permissions and expand only as needed**:
> ```yaml
> permissions: {}  # Start with no permissions
> jobs:
>   publish:
>     permissions:
>       contents: read
>       id-token: write
> ```
>
> **Audit workflows with zizmor**:
> ```bash
> $ uvx zizmor .
> $ uvx zizmor --gh-token $(gh auth token) .
> ```

### Additional CI/CD hardening measures

- **Avoid insecure triggers.** Workflows that can be triggered by an attacker with controlled inputs (such as PR titles, branch titles) have been used to inject commands. The `pull_request_target` trigger from GitHub Actions in particular is difficult to use securely and should be avoided[^1][^3]
- **Sanitize parameters and inputs.** Any workflow parameter or input that can expand into an executed command carries potential for template injection. Pass values as environment variables to commands instead of interpolating them directly[^1]
- **Use reviewable deployments.** Trusted Publishers for GitHub supports "GitHub Environments" as a required step, meaning publishing your package to PyPI requires a review from your GitHub account — a higher bar for attackers to clear[^1]
- **Use `pinact`** to automatically pin actions to their commit SHAs[^3]
- **Disable caching during release builds** to prevent cache poisoning attacks[^3]
- **Use deployment environments** to isolate secrets rather than storing them at the organization level[^3]
- **Protect release tags** so only maintainers can create or modify tags matching your release pattern (e.g., `v*`)[^11]

If you are using GitHub Actions, PyPI highly recommends the tool [zizmor](https://github.com/zizmorcore/zizmor/) for detecting and fixing insecure workflows[^1].

---

## 14. Reduce Your Package Dependency Tree

> [!WARNING]
> Each dependency in your package increases the attack surface and potential for supply chain vulnerabilities, as users inherit all transitive dependencies when installing your package.

Minimizing dependencies reduces security risks, improves performance, and decreases the likelihood of supply chain attacks. Fewer dependencies mean fewer potential points of failure and reduced exposure to malicious packages in the dependency tree.

> [!TIP]
> **Security Best Practice**: Design packages with minimal or zero dependencies by leveraging Python's rich standard library instead of external packages.

> [!NOTE]
> **How to implement?**
>
> Replace common dependencies with Python standard library equivalents:
> ```python
> # Instead of requests for simple HTTP calls
> from urllib.request import urlopen
> response = urlopen(url)
>
> # Instead of python-dotenv for env file loading
> # Python 3.13+ has built-in .env support via tomllib
> import os
> value = os.environ.get("KEY", "default")
>
> # Instead of pathlib2 or os.path utilities
> from pathlib import Path
> config = Path("config.toml").read_text()
> ```

Actively audit and eliminate unnecessary dependencies. Contribute to upstream projects' security rather than forking or re-implementing. Consider the maintenance burden, security implications, and dependency tree depth before adding any dependency.

---

## 15. Generate and Track SBOMs

> [!WARNING]
> Without a Software Bill of Materials (SBOM), you cannot rapidly assess whether your applications are affected when a new vulnerability is disclosed or a supply chain compromise is announced.

When the next supply chain attack drops — like LiteLLM[^1] — you need to answer "are we affected?" in minutes, not days. SBOMs provide a complete inventory of all software components in your application, enabling rapid impact assessment and incident response[^7].

> [!TIP]
> **Security Best Practice**: Generate SBOMs at build time using CycloneDX format and maintain them alongside your release artifacts for rapid vulnerability response.

> [!NOTE]
> **How to implement?**
>
> Generate an SBOM from your installed environment:
> ```bash
> $ uv pip install cyclonedx-bom
> $ cyclonedx-py environment --output-file sbom.json
> ```
>
> Or integrate into your CI/CD pipeline:
> ```yaml
> # .github/workflows/build.yml
> steps:
>   - uses: actions/checkout@v4
>   - uses: astral-sh/setup-uv@v5
>   - run: uv sync --frozen
>   - run: uvx cyclonedx-py environment --output-file sbom.json
>   - uses: actions/upload-artifact@v4
>     with:
>       name: sbom
>       path: sbom.json
> ```

CycloneDX SBOMs include package metadata, cryptographic hashes, source repository links, and standardized package URLs (pURLs) for cross-referencing with vulnerability databases[^7].

---

## 16. Consult Vulnerability Databases for Package Health

> [!WARNING]
> Installing Python packages without reviewing their health signals can expose your project to unmaintained, insecure, or low-quality dependencies.

Package health encompasses more than just known vulnerabilities — it includes maintenance activity, community adoption, and security posture. A package that is rarely maintained or has a shrinking community may be at greater risk of future compromise or abandonment.

> [!TIP]
> **Security Best Practice**: Before adopting a new Python package, consult vulnerability databases and package health tools to review its security posture, maintenance status, and community signals.

> [!NOTE]
> **How to implement?**
>
> Consult the [Snyk Security Database](https://security.snyk.io) for Python package health:
> ```
> https://security.snyk.io/package/pip/<package-name>
> ```
>
> Use the [OSV database](https://osv.dev/) for Python-specific vulnerability data:
> ```
> https://osv.dev/list?ecosystem=PyPI&q=<package-name>
> ```
>
> Check a package's PyPI page for Trusted Publisher badges and attestation status, which indicate stronger supply chain security practices by the maintainer.

---

## 17. Verify Published Package Contents

> [!WARNING]
> The source code on GitHub may not match what is actually published to PyPI. Attackers who compromise a build pipeline can inject malicious code into published packages while the repository source remains clean — as demonstrated in the Ultralytics compromise[^2], where GitHub Actions cache poisoning produced builds that did not match the tagged source.

PyPI does not display package source code — it shows the README and metadata provided by the package author. Unlike a source repository, there is no way to browse or diff the actual installed code on pypi.org. This means a package's PyPI page can appear legitimate while the published sdist or wheel contains entirely different code. Additionally, a package's sdist (source distribution) and wheel (binary distribution) may contain different files from each other.

> [!TIP]
> **Security Best Practice**: Do not rely solely on a package's PyPI page or GitHub repository to evaluate its safety. Inspect the actual contents of published distributions before adoption, and prefer packages with attestations that cryptographically link builds to source commits[^13][^14].

> [!NOTE]
> **How to implement?**
>
> Download and inspect a package's published wheel without installing it:
> ```bash
> $ uv pip download <package-name> --no-deps -d ./inspect
> $ unzip ./inspect/<package-name>-*.whl -d ./inspect/unpacked
> ```
>
> Or inspect a source distribution:
> ```bash
> $ uv pip download <package-name> --no-deps --no-binary :all: -d ./inspect
> $ tar -tzf ./inspect/<package-name>-*.tar.gz
> ```
>
> Compare the published contents against the GitHub source at the tagged release to verify they match.

For packages that use Trusted Publishing with attestations (see [section 12](#12-publish-with-package-attestations)), you can verify the cryptographic link between the published artifact and the source repository and build workflow, providing much stronger assurance than manual inspection alone.

---

## Author

**PyPI Security Best Practices** © [Liran Tal](https://github.com/lirantal), Released under [Apache 2.0](./LICENSE) License.

[^1]: [LiteLLM/Telnyx Supply Chain Attack Incident Report (March 2026)](https://blog.pypi.org/posts/2026-04-02-incident-report-litellm-telnyx-supply-chain-attack/) — Malicious versions of litellm (2h 32m exposure, 119k+ downloads) and telnyx (3h 42m exposure) published to PyPI via compromised Trivy CI/CD dependency, exfiltrating credentials and sensitive files. 40-50% of LiteLLM installs were unpinned. See also: [LiteLLM Security Update](https://docs.litellm.ai/blog/security-update-march-2026), [Datadog Security Labs analysis](https://securitylabs.datadoghq.com/articles/litellm-compromised-pypi-teampcp-supply-chain-campaign/), [Snyk analysis](https://snyk.io/blog/poisoned-security-scanner-backdooring-litellm/)
[^2]: [Ultralytics Supply Chain Compromise](https://blog.pypi.org/posts/2024-12-11-ultralytics-attack-analysis/) — Compromised PyPI package via GitHub Actions cache poisoning
[^3]: [Open Source Security at Astral](https://astral.sh/blog/open-source-security-at-astral) — Astral's comprehensive security practices for uv and Ruff, including CI/CD hardening, trusted publishing, Sigstore attestations, and dependency management
[^4]: [How to Protect Against Python Supply Chain Attacks with uv](https://pydevtools.com/handbook/how-to/how-to-protect-against-python-supply-chain-attacks-with-uv/) — Practical guide to uv's `exclude-newer` feature for dependency cooldowns
[^5]: [Securing Python/Linux Supply Chains](https://scaledpython.substack.com/p/securing-pythonlinux-supply-chains) — pip and uv configuration for delayed package ingestion, proxy management, and enterprise deployment patterns
[^6]: [uv exclude-newer for PyPI Supply Chain Security](https://docs.bswen.com/blog/2026-04-02-uv-exclude-newer-supply-chain/) — Detailed guide on `exclude-newer` configuration formats and caveats
[^7]: [Defense in Depth: A Practical Guide to Python Supply Chain Security](https://bernat.tech/posts/securing-python-supply-chain/) — Comprehensive guide covering hash pinning, pip-audit, SBOMs, trusted publishing, and CI/CD security
[^8]: [uv Package Indexes Documentation](https://docs.astral.sh/uv/concepts/indexes/) — uv's first-match index strategy for preventing dependency confusion attacks
[^9]: [Datadog Supply-Chain Firewall (SCFW)](https://github.com/DataDog/supply-chain-firewall) — Open-source tool that wraps `pip install` to block known malicious packages
[^10]: [Securing PyPI accounts via Two-Factor Authentication](https://blog.pypi.org/posts/2023-05-25-securing-pypi-with-2fa/) — PyPI's 2FA implementation and recommendations
[^11]: [PyPI Trusted Publishers: Security Model and Considerations](https://docs.pypi.org/trusted-publishers/security-model/) — Security model, best practices, and considerations for Trusted Publishing
[^12]: [PyPI Trusted Publishers Documentation](https://docs.pypi.org/trusted-publishers/) — Setup guide for configuring Trusted Publishing with OIDC
[^13]: [PEP 740 — Index Support for Digital Attestations](https://peps.python.org/pep-0740/) — Specification for digital attestations on Python package indexes
[^14]: [PyPI Now Supports Digital Attestations](https://blog.pypi.org/posts/2024-11-14-pypi-now-supports-digital-attestations/) — Announcement and technical details of Sigstore-powered attestations on PyPI
[^15]: [What's New in pip 26.0 — Excluding Distributions by Upload Time](https://ichard26.github.io/blog/2026/01/whats-new-in-pip-26.0/#excluding-distributions-by-upload-time) — pip v26.0 absolute cooldowns, pip v26.1 relative duration support
