# Repository Guidelines

## Project Structure & Module Organization

This repository packages official Prometheus Node Exporter Linux binaries for fnOS; it does not maintain the upstream Go source.

- `.github/workflows/main.yml` coordinates builds, checksums, and optional prerelease publication.
- `pack_amd64.yml` and `pack_arm64.yml` in that directory download binaries and assemble architecture-specific packages.
- `node_exporter/` contains the manifest template, `cmd/` lifecycle scripts, permission/resource configuration, icons, and `i18n/` translations.
- `app/ui/` contains the application URL configuration and icon. Root `ui/` retains a tracked placeholder.
- `tmp/` is ignored scratch space. Keep downloaded binaries and generated archives out of commits.

## Build, Test, and Development Commands

Use Linux or WSL with Bash, `wget`, GNU `tar`, `sed`, and `sha256sum` for packaging. Reproduce workflow steps in a disposable checkout because they replace manifest placeholders.

- `gh workflow run main.yml -f make_release=false` starts both architecture builds without publishing a release.
- `bash -n node_exporter/cmd/main` checks service-script syntax; repeat for changed lifecycle hooks.
- After downloading the binary to `app/node_exporter` and substituting the manifest, `tar zcvf node_exporter/app.tgz app ui` creates the inner payload. Follow the architecture workflow for the outer archive.
- `tar -tzf <package>.tgz` inspects package contents; `sha256sum -c SHA256SUMS.txt` verifies downloaded release assets.

## Coding Style & Naming Conventions

Match surrounding formatting: YAML uses spaces, JSON generally uses four spaces, and Bash currently mixes tabs and spaces. Avoid unrelated reformatting. Preserve Bash shebangs, executable modes, and Linux-compatible line endings. No formatter or linter is configured.

Keep manifest placeholders intact in source. Update upstream and package versions in `main.yml`; package revisions follow `<upstream-version>-<revision>`. Keep both architecture workflows aligned. Maintain matching translation keys in all `node_exporter/i18n/` locales: `en`, `zh-CN`, `ja-JP`, `zh-TW`, `de-DE`, `fr-FR`, `es-ES`, `ko-KR`, `ru-RU`, and `it-IT`. Preserve embedded HTML and documentation links when translating.

## Testing Guidelines

For requests limited to syncing official upstream versions, verify the latest stable official release and update workflow versions and matching documentation. Do not run local builds or trigger CI builds unless explicitly requested; static version and diff checks are sufficient for this scope.

No automated test framework or coverage threshold exists. Validate both architecture packages, substituted manifest values, and nested archive contents. On fnOS, exercise `start`, `status`, and `stop` through the application lifecycle, verify `/metrics` on port `9100`, and inspect `/var/log/node_exporter.log`. Local archive checks do not establish device compatibility.

## Commit & Pull Request Guidelines

Preserve the repository's existing commit-title conventions and keep subjects concise. After the title and a blank line, write the commit body in Chinese using this general structure:

- Start with one or two overview paragraphs explaining the purpose and resulting behavior of the complete change.
- Group details by actual repository file path; combine closely related files when their changes are identical.
- Under each file group, use bullets in the form `- <变更主题> <新增|修改|修复|删除>：<具体变更及作用>。` Choose the applicable action word rather than listing alternatives.
- Describe only changes included in the commit. Scale detail to the change; avoid generic filler, unrelated project examples, and separate validation sections. Do not add `Signed-off-by:` trailers.

Keep commits focused. Pull requests should explain the change, affected architectures, and validation performed; link relevant issues and include screenshots for visible UI changes.
