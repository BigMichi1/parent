# AI Copilot Instructions for Parent Project

## Project Overview

**Parent** is a Maven parent POM project (version 6.1.0-SNAPSHOT) that manages plugin versions for Java Maven projects. It's a pure configuration/management artifact with no source code—it centralizes Maven plugin version declarations to ensure consistency across dependent projects. Published to Maven Central via OSSRH.

**Key Tech Stack:**
- **Build:** Maven 3.9.9 with Maven Wrapper
- **Java:** JDK 25 (with `maven.compiler.release=25`)
- **Code Quality:** SpotBugs, Checkstyle (Google style), JDeprscan, JaCoCo (coverage)
- **Modernization:** OpenRewrite with recipes for Java 21+ migration and best practices
- **CI/CD:** GitHub Actions (Maven CI, Release Drafter, Dependency Review, Scorecards)

## Critical Developer Workflows

### Building & Validation

```bash
# Use the Maven Wrapper (recommended - self-bootstrapping)
./mvnw clean package
./mvnw validate        # Runs pre-commit checks: Checkstyle, SpotBugs, JDeprscan, OpenRewrite
./mvnw clean package   # Full build with JaCoCo coverage preparation
```

**What happens during validation phase:**
1. **OpenRewrite** runs ~14 recipes (Java 21 migration, static analysis, Apache Commons upgrades)
2. **Checkstyle** validates against Google style guide (warn-only, fails build on violations)
3. **SpotBugs** detects likely bugs
4. **JDeprscan** scans for deprecated API usage (JDK version-specific)
5. **maven-enforcer-plugin** checks: Maven ≥3.9.9 and Java ≥25

### Release Workflow

**Automated via GitHub Actions** (maven-publish.yml):
1. Release created in GitHub UI
2. Maven runs `release:prepare release:perform` with GPG signing
3. Artifacts deployed to OSSRH (Central staging)
4. Uses `SemVerVersionPolicy` with tag format `v@{project.version}`

**Profile-based:** Release profile enables GPG signing via central-publishing-maven-plugin with loopback PIN entry mode.

### Pre-commit Checks

**File:** `.pre-commit-config.yaml` runs:
- Gitleaks (secret scanning)
- Standard hooks: trailing whitespace, CRLF/BOM checks, XML validation

## Project-Specific Patterns & Conventions

### 1. Plugin Management Strategy

**All plugins use `<pluginManagement>` sections** in the parent POM:
- **PluginManagement block:** Defines 30+ plugins with pinned versions, configurations, and execution bindings
- **Plugins block:** Declares which ones to actually execute (10 plugins active for this parent)
- **Reporting block:** Separate reporting plugin list for Maven site generation

**Why:** This structure allows consuming projects to:
1. Inherit exact plugin versions (consistency)
2. Override configs if needed without re-specifying versions
3. Activate only required plugins

### 2. Strict Code Quality Standards

**Checkstyle (google_checks.xml):**
- Configured to console output but `failsOnError=false`
- Used as validation phase check, not failure blocker
- Custom link XRef enabled for reports

**SpotBugs:**
- Auto-runs on every build (check goal in validate phase)
- Catches common Java defects: null dereference, infinite loops, suspicious comparisons

**JDeprscan:**
- Scans both source AND test code for deprecated APIs
- Critical for staying current with Java version (release 25)
- Configured with `<release>${maven.compiler.source}</release>`

### 3. OpenRewrite Integration

**Central modernization tool** with 14 active recipes:
- Java migration recipes (UpgradeToJava21, testing framework updates)
- Static analysis recipes (CommonStaticAnalysis, JavaApiBestPractices)
- Apache Commons upgrade recipes (collections, codec, lang3, math)
- SLF4J best practices

**Runs in validation phase automatically.** To test recipes manually:
```bash
./mvnw org.openrewrite.maven:rewrite-maven-plugin:6.18.0:run
```

### 4. Multi-profile Architecture

**Release Profile (`release`):**
- Activates GPG signing (maven-gpg-plugin)
- Configures central-publishing-maven-plugin with OSSRH server ID
- Uses loopback PIN entry for automated CI/CD signing

**Properties:**
- `maven.compiler.release=25` - Sets both source/target to Java 25
- `project.build.sourceEncoding=UTF-8` - Consistent file encoding

### 5. Dependency Management

**maven-dependency-plugin:**
- Auto-downloads sources for all dependencies (helpful for IDE browsing)
- Runs in validate phase, stores metadata for analysis
- Exclusions pattern: `.repository/**` (ignore repository folders)

### 6. JaCoCo Coverage

- Prepares agent in validate phase
- Generates report in deploy phase
- Key for tracking code coverage over time (used in release pipeline)

## Cross-Component Integration Points

### GitHub Actions Workflows

**maven.yml:** Core build (push to main, PRs)
- Runs: `mvn -B package` (batch mode, no colors)
- Uses setup-java for Maven caching
- Harden Runner applied for security

**maven-publish.yml:** Release deployment
- Triggered on release creation
- Checks out main, configures git/SSH, runs `release:prepare release:perform`
- Signs with GPG, deploys to OSSRH
- Environment: OSSRH_USERNAME, OSSRH_TOKEN, MAVEN_GPG_PASSPHRASE (GitHub secrets)

**dependency-review.yml:** Vulnerability scanning
- Scans PRs for known vulnerable dependencies
- Maven dependency submission for dependency tracking

**release-drafter.yml:** Changelog automation
- Runs on main pushes, PRs
- Uses `.github/release-drafter.yml` configuration
- Groups PRs by labels (feature/bug/docs/dependencies/automation)

### External Services

- **OSSRH (Sonatype Central):** Maven artifact repository (snapshot + release)
- **Maven Central:** Published artifacts endpoint
- **GitHub Security Advisory:** Manual security reporting channel

## Key File References

| File | Purpose |
|------|---------|
| `pom.xml` | Root POM with pluginManagement, reporting, release profile |
| `.github/workflows/maven.yml` | Build CI pipeline |
| `.github/workflows/maven-publish.yml` | Release deployment automation |
| `.mvn/wrapper/maven-wrapper.properties` | Maven 3.9.9 self-bootstrap config |
| `.pre-commit-config.yaml` | Git hooks (secrets, formatting) |
| `CONTRIBUTING.md` | Developer contribution guidelines |

## Common Task Examples

**Add a new plugin to manage:**
1. Add `<plugin>` section to `<pluginManagement>` in pom.xml with version
2. Optionally add to `<plugins>` active list
3. Define executions/configurations if needed
4. Test with: `./mvnw help:describe -Dplugin=groupId:artifactId`

**Update plugin versions:**
```bash
./mvnw versions:display-dependency-updates    # Show outdated plugins
./mvnw versions:update-properties              # Auto-update version properties
```

**Debug OpenRewrite recipes:**
```bash
./mvnw -e org.openrewrite.maven:rewrite-maven-plugin:dryRun    # Simulate changes
```

**Release a new version:**
1. GitHub creates release with tag (triggers maven-publish.yml)
2. Workflow handles version bumping via SemVer policy
3. GPG signs and deploys to OSSRH automatically

## Assumptions & Gotchas

- **No source code:** This is purely a dependency/plugin configuration project. Agents should not expect Java classes or tests.
- **Maven Central only:** Artifacts distributed via OSSRH→Central, not GitHub Packages.
- **Strict version pinning:** All plugin versions must be explicit—no ranges or BOMs for plugin management.
- **Java 25 strict mode:** Code must pass JDeprscan against Java 25; deprecated APIs fail builds.
- **Schema validation:** `maven-checkstyle-plugin` uses hardcoded `google_checks.xml`—custom configs require overrides.
