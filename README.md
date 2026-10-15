# Seal CLI Action

Fix vulnerable open-source dependencies in your GitHub Actions build, without upgrading them.

This action runs the [Seal CLI](https://github.com/seal-community/cli) in your workflow. The CLI finds the vulnerable packages your build just installed and replaces them in place with **sealed** versions: the same versions you already use, with the security fixes backported. Your source code doesn't change, and the rest of the build runs against the sealed packages.

It supports npm, Yarn, pnpm, Maven, Gradle, pip, Poetry, Pipenv, uv, Go, NuGet, Composer and Bundler projects. See the [GitHub Actions guide](https://docs.sealsecurity.io/setup-apps-os/cli-in-cicd/github-actions) for the full setup.

## Quick start

1. Store your Seal token as a repository secret, for example `SEAL_TOKEN`. See [Tokens](https://docs.sealsecurity.io/getting-started/tokens).
2. Add the step **after** the step that installs your dependencies (`npm ci`, `mvn install`, `pip install` and so on) and **before** any step that builds, bundles or packages them.

```yml
steps:
  - uses: actions/checkout@v4

  - run: npm ci

  - name: Seal fix
    uses: seal-community/cli-action@v0.1.17
    with:
      mode: fix
      fix_mode: remote
      target: ./package-lock.json
      token: ${{ secrets.SEAL_TOKEN }}
      project: my-project-id

  - run: npm run build
```

If the step runs before the install step there is nothing on disk to replace. If it runs after packaging, the output was already built from the vulnerable versions.

## Choosing a fix mode

With `mode: fix`, the `fix_mode` input decides which packages get replaced:

| `fix_mode` | What gets replaced |
| --- | --- |
| `remote` | Packages covered by the Sealing Rules your security team manages for this project on the Seal Platform. |
| `local` (default) | Packages listed in the `.seal-actions.yml` file in your repository, reviewed and versioned with your code. The [Seal GitHub App](https://docs.sealsecurity.io/integrations/github/github-app) can open pull requests that fill it in for you. |
| `all` | Every vulnerable package that has a sealed version, with no rules to maintain. |

See [The CLI fix mode](https://docs.sealsecurity.io/setup-apps-os/cli-in-cicd/cli-fix-mode) for how to pick one.

To report what's vulnerable without changing anything, use `mode: scan`.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `mode` | `scan` | `scan` finds and reports vulnerable packages. `fix` replaces them with sealed versions. |
| `fix_mode` | `local` | `local`, `remote` or `all`. Only used with `mode: fix`. See [Choosing a fix mode](#choosing-a-fix-mode). |
| `target` | `./requirements.txt` | Path to the dependency manifest, such as `package-lock.json`, `pom.xml`, `go.mod` or `requirements.txt`. Set it for anything that isn't a pip project. |
| `token` | | Your Seal token. Required for `fix`. Pass it from a secret, never in plain text. `CLI_TOKEN` is an older name for the same input. |
| `project` | | The Seal Project ID that this run reports to. Required for `fix`. See [Project IDs](#project-ids). |
| `summary` | | Path of a file listing the packages that were replaced. Only used with `mode: fix`. By default no file is written. |
| `version` | latest | The Seal CLI release to run, for example `v0.3.368`. Leave empty to use the latest release. |
| `verbosity` | `v` | Log level: `v`, `vv` or `vvv`. |
| `verify_signature` | `true` | Verifies the downloaded CLI's signature before running it. Only `false` turns it off. See [Signature verification](#signature-verification). |

The action has no outputs. If the CLI fails, the step fails.

### Project IDs

The project ID ties the run to a Seal Project on the Seal Platform. If the project already exists there, for example because the Seal GitHub App imported the repository, use the same ID, or the platform treats this run as a separate project. If it doesn't exist yet, pick a new ID and the first run creates it.

An ID can contain ASCII letters, digits, underscores, hyphens and periods, up to 255 characters.

## Signature verification

Before it runs the CLI, the action checks that the download was signed by Seal Security. If the check fails, the step fails and the CLI never runs.

| Runner | What the action checks |
| --- | --- |
| Linux and Windows | It installs [cosign](https://github.com/sigstore/cosign) and verifies the signature bundle that ships with the CLI against Seal's public signing key, which is built into the action. Each signature is also recorded in Sigstore's public transparency log, Rekor, and linked from the [CLI release notes](https://github.com/seal-community/cli/releases). |
| macOS | The installer and the `seal` binary must both be signed with Seal's Apple Developer ID (team `SNZS6SJMFA`), and the installer must be notarized by Apple. |

Older CLI releases are not signed, so the check fails for them: Linux builds before v0.3.365 and Windows builds before v0.3.370. To run one of them, set `verify_signature: false`. To run the check yourself, see [Verifying the Linux download](https://docs.sealsecurity.io/setup-apps-os/cli-in-cicd/installing-the-cli#verifying-the-linux-download); the Windows check is the same, with `seal.exe` and `seal.exe.sigstore.json`.

## More examples

**Scan only:**

```yml
- name: Seal scan
  uses: seal-community/cli-action@v0.1.17
  with:
    mode: scan
    target: ./pom.xml
```

**Fix everything and keep a record of what changed:**

```yml
- name: Seal fix
  uses: seal-community/cli-action@v0.1.17
  with:
    mode: fix
    fix_mode: all
    target: ./requirements.txt
    token: ${{ secrets.SEAL_TOKEN }}
    project: my-project-id
    summary: seal-summary.json

- uses: actions/upload-artifact@v4
  with:
    name: seal-summary
    path: seal-summary.json
```

**Monorepos and matrix builds:** run one step per manifest, each with its own `target` and, usually, its own `project`.

**Pulling sealed packages through your own registry:** set `SEAL_ARTIFACT_SERVER_BASE_URL` in the step's `env` to the URL of a Nexus or Artifactory proxy repository you control. See [Pulling sealed packages through your own registry](https://docs.sealsecurity.io/setup-apps-os/cli-in-cicd/custom-artifact-server).

```yml
- name: Seal fix
  uses: seal-community/cli-action@v0.1.17
  env:
    SEAL_ARTIFACT_SERVER_BASE_URL: https://nexus.example.com/repository/seal-npm-proxy
  with:
    mode: fix
    fix_mode: remote
    target: ./package-lock.json
    token: ${{ secrets.SEAL_TOKEN }}
    project: my-project-id
```

## Requirements

- **A supported runner**, GitHub-hosted or self-hosted: Linux x64 or Arm64 (`ubuntu-latest`, `ubuntu-24.04-arm`), macOS on Apple Silicon or Intel (`macos-latest`, `macos-15-intel`), or Windows x64 (`windows-latest`). Self-hosted runners need `curl` or `wget` to download the CLI, plus `unzip` on Linux and Windows; the signature check on Linux and Windows also needs `curl` and `sha256sum` (or `shasum`) for [cosign-installer](https://github.com/sigstore/cosign-installer), and Windows runners also need [Git for Windows](https://gitforwindows.org), because the action runs its steps in Bash. GitHub-hosted runners already have all of these. For any other CI system, [install the CLI](https://docs.sealsecurity.io/setup-apps-os/cli-in-cicd/installing-the-cli) directly.
- Network access to the Seal Platform and Seal's artifact servers. See [Common issues](https://docs.sealsecurity.io/setup-apps-os/cli-in-cicd/common-issues) if a run fails.

## Versioning

Pin the action to a release tag, such as `@v0.1.17`, for reproducible builds. `@latest` always points to the newest release. The `version` input pins the CLI itself, separately from the action.

## FAQ

**What is a sealed version?**
The version of the package you already use, with only the upstream security fix backported onto it. It keeps the same major version, API and behavior, so it's a drop-in replacement, and it's signed by Seal. For example, Seal provides `spring-beans` `4.3.30+sp1` for Spring4Shell (CVE-2022-22965), where the only upstream fix is a migration to Spring 5.

**Will a sealed version break my build?**
Seal uses the community fix with the smallest change possible, usually fewer than 10 lines of code, compared with the hundreds or thousands of lines in a typical upgrade. Every sealed version is tested and reviewed by Seal's vulnerability researchers. And because the action runs in your build, all your existing tests run against the sealed packages.

**What about transitive dependencies and end-of-life versions?**
Seal replaces only the vulnerable package, wherever it sits in the dependency tree, so a deep transitive CVE doesn't force a framework upgrade. There's no hard limit on how far back Seal goes: as long as the source code is public, Seal can fix it.

**How quickly are new vulnerabilities covered?**
Seal handles critical and high severity vulnerabilities within 72 hours of being made public. Seal also watches upstream projects for security fixes that land before a CVE is published, so a sealed version can be ready before the CVE goes public.

**Will my scanners know the vulnerability is fixed?**
Yes. Seal integrates with SCA tools and vulnerability scanners such as Snyk, Black Duck, GitHub Advanced Security, Checkmarx, Ox, Wiz, Trivy, StackRox and Grype, so they show the vulnerability as remediated by Seal. See [Integrations](https://docs.sealsecurity.io/integrations).

**Does Seal need access to my production environment?**
No. Seal works entirely at build time, with no runtime agents and no production permissions.

**My builds can't reach the internet directly. Can I still use it?**
Yes. Set up JFrog Artifactory or Sonatype Nexus as a proxy to Seal and point the action at it with `SEAL_ARTIFACT_SERVER_BASE_URL`, as shown in [More examples](#more-examples), so your builds only talk to your internal registry.

**How do sealed versions affect licensing?**
They don't change your licensing requirements. Seal's patches are released under a permissive license, but you still need to follow the license of the original package.

**Where can I get help?**
The [documentation](https://docs.sealsecurity.io) has setup guides for every deployment method and the CLI reference. Customers can also reach Seal on their shared Slack or Teams channel, or at [support@sealsecurity.io](mailto:support@sealsecurity.io).

## Feedback and contributions

Questions, bug reports and feature requests are welcome. Open an issue or a pull request, or write to [contribute@seal.security](mailto:contribute@seal.security). To report a security vulnerability, see [Security.md](Security.md).

## About Seal Security

![Seal Security Logo](docs/assets/logo.png)

Seal Security is the leading open source vulnerability remediation platform. Seal backports the upstream security fix onto the exact version of the open source package you already run and delivers it as a signed, drop-in sealed version. CVEs get fixed without breaking changes, version upgrades, migrations or code changes.

Seal covers application dependencies (direct and transitive) and Linux OS packages, including containers and end-of-life distros, with a 72-hour SLA for critical and high severity CVEs.

Seal is well established and supports some of the largest financial institutions in the world. For more, visit [seal.security](https://seal.security).
