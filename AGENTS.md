# ty Repository

## Custom ty deployment to sk-ml

For rebases and deployments of this fork, follow
[the sk-ml deployment runbook](agents/references/sk-ml-ty-deployment.md).
Only Zed's remote language server uses the custom executable at
`/home/sourish/.local/share/zed/ty-custom/ty`. Keep the regular `ty` command and
its uv tool environment on upstream ty. Never deploy the fork with
`uv tool install` or write into `~/.local/bin/ty` or `~/.local/share/uv/tools/ty`.

## Security reviews

For reviews of this repository's GitHub automation, CI, and release process, use the [GitHub repository threat model](agents/references/repository-threat-model.md). The `ruff` submodule pins ty's Rust source for release builds; include it when tracing the source of release artifacts.
