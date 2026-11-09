# incus-spawn-templates

Community template images and tool definitions for [incus-spawn](https://github.com/Sanne/incus-spawn).

These extend isx's built-in `tpl-minimal` with project-specific environments that include pre-cloned repositories, build tools, and primed dependency caches.

`tpl-dev` and `tpl-java` live here too. They used to ship with isx, which now marks its own copies deprecated and will drop them ([#1195](https://github.com/Sanne/incus-spawn/issues/1195)). The definitions here have the same names and contents, so with this directory on your search path they replace the built-ins without the deprecation warning, and templates you already built stay in sync.

## Setup

Add this directory as a search path in `~/.config/incus-spawn/config.yaml`:

```yaml
searchPaths:
  - /path/to/incus-spawn-templates
```

Then build and branch:

```shell
isx build tpl-quarkus
isx branch my-feature --from tpl-quarkus
```

## Templates

| Template | Parent | Description                                                          |
|----------|--------|----------------------------------------------------------------------|
| `tpl-dev` | `tpl-minimal` | Podman, GitHub CLI, Starship, tmux                                   |
| `tpl-java` | `tpl-dev` | JDK 25 + Maven                                                       |
| `tpl-mydev` | `tpl-java` | Shared layer under the project templates: Claude Code, tig           |
| `tpl-quarkus` | `tpl-mydev` | Quarkus + quickstarts source, Podman, GraalVM, mvnd                  |
| `tpl-hibernate-orm` | `tpl-mydev` | Hibernate ORM source, Podman, Gradle, `idea-backend`                 |
| `tpl-hibernate-reactive` | `tpl-hibernate-orm` | Hibernate Reactive source (includes ORM)                             |
| `tpl-tamboui` | `tpl-mydev` | Tamboui TUI framework source, Gradle, `idea-backend`, selected skills |
| `tpl-isx` | `tpl-mydev` | incus-spawn and its sibling repos, GraalVM, zmx, `/dev/kvm`          |
| `tpl-graalvm-dev` | `tpl-dev` | GraalVM development — `mx`, labsjdk, the `oracle/graal` source |

Most templates inherit the `tpl-java` chain, which provides JDK 25, Maven, Podman, GitHub CLI, Starship, and tmux. That chain has no Claude Code: the templates under `tpl-mydev` get it from there, and `tpl-graalvm-dev` has none.

### Working on GraalVM

`tpl-graalvm-dev` is ready to build on first shell — `mx`, a JVMCI-enabled labsjdk in
`/opt/jdks` (already `JAVA_HOME`), and the `oracle/graal` source, with the shell opening
in `~/graal`:

```shell
isx branch graal-work --from tpl-graalvm-dev
cd compiler && mx build
```

It hangs off `tpl-dev` rather than `tpl-java` on purpose: graal has to be built against the
labsjdk, and `JAVA_HOME` can only be set once across a template chain.

To build against a different JDK, override the `labsjdk` tool's `jdk_id` parameter (any id
from `mx fetch-jdk --list`) in your own template.

`mx` validates TLS in Python rather than through the JVM truststore, so the `mx fetch-jdk`
step in this build is also a good check that the isx MITM proxy's certificates satisfy
OpenSSL 3.5+ strict verification.

## Tools

| Tool | Description |
|------|-------------|
| `podman` | Podman with Docker socket compatibility, configured for Testcontainers |
| `gradle` | Gradle 9.9.0 (installed to `/opt`, symlinked to PATH) |
| `graalvm` | Oracle GraalVM for JDK 25 (installed to `/opt`, `native-image` on PATH) |
| `mx` | GraalVM's `mx` build tool (cloned to `/opt/mx`, `mx` on PATH) |
| `labsjdk` | JVMCI-enabled labsjdk fetched by `mx` into `/opt/jdks` and set as `JAVA_HOME` (param: `jdk_id`) |
| `jtreg` | JTReg test harness (installed to `/opt`, `jtreg` on PATH) |
| `pinchtab` | [PinchTab](https://github.com/pinchtab/pinchtab) browser control for agents — headless Chromium, the `pinchtab` skill (params: `allowed_domains`, `mcp`) |

### Giving an agent a browser

Add `pinchtab` to any template to put a real browser behind a token-efficient CLI:

```yaml
tools:
  - pinchtab
```

The agent gets the upstream `pinchtab` skill in the shared skills directory, so
it can open pages, read them as accessibility snapshots rather than raw HTML,
click through flows, and export screenshots or PDFs. Nothing needs to be
started first — `pinchtab nav <url>` launches the control-plane server on
demand. For a server that survives a container restart, run `pinchtab daemon
install` inside the container.

As with any isx skill, Claude Code picks it up automatically through the
`~/.claude/skills` symlink; other agents get a working `pinchtab` CLI but have
to be pointed at `~/.agents/skills/pinchtab/SKILL.md` themselves.

PinchTab ships locked down to locally hosted sites, which makes it useless for
an agent doing research, so this tool opens the allowlist (`allowed_domains`
defaults to `*`) on the isx premise that the container is the blast radius.
Everything else stays at PinchTab's recommended defaults — IDPI strict mode
enforcing, the content guard wrapping page text as untrusted, sensitive
endpoint families off — and `pinchtab security` inside the container reports
the allowlist as the single deviation.

Know what the open allowlist means here specifically. The host proxy injects
real credentials for intercepted domains, and a loaded page's *own JavaScript*
reaches them: a cross-origin `fetch('https://api.github.com/user')` from a
visited page comes back authenticated, because GitHub's API allows every
origin. That is exposure the agent never chose, and it is a property of putting
any browser in a credential-injecting container — not of PinchTab.

`allowed_domains` gates **navigation**, so narrowing it decides which pages get
to run JavaScript at all. It does not contain one that already did: a page
served from localhost still read the GitHub API with the allowlist set to
`localhost` alone. Narrow it to keep hostile pages out, not to declaw them:

```yaml
tools:
  - pinchtab: {allowed_domains: "quarkus.io,*.quarkus.io,github.com"}
```

(`*.quarkus.io` matches subdomains but not the apex, so list both.)

Set `mcp: true` to also register PinchTab's MCP server with Claude Code and
install the `pinchtab-mcp` skill; the default is the CLI path, which works for
any agent that can run a shell command.

## Contributing

To add a new template, create a YAML file in `images/` — see the [incus-spawn documentation](https://github.com/Sanne/incus-spawn) for the image definition format. Custom tools go in `tools/`.
