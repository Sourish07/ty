# Custom ty for Zed on sk-ml

The custom semantic-token fork is used by Zed when connecting from the laptop
to sk-ml over SSH. The regular command-line installation stays on upstream ty.

## Installation boundaries

| Purpose                              | Location on sk-ml                             |
| ------------------------------------ | --------------------------------------------- |
| Zed's custom executable              | `/home/sourish/.local/share/zed/ty-custom/ty` |
| Zed's remote settings                | `/home/sourish/.config/zed/settings.json`     |
| Regular upstream command             | `/home/sourish/.local/bin/ty`                 |
| Regular upstream uv tool environment | `/home/sourish/.local/share/uv/tools/ty`      |
| Fork source checkout                 | `/home/sourish/src/ty-fork`                   |

The remote Zed settings contain this override; preserve other settings:

```json
{
  "lsp": {
    "ty": {
      "binary": {
        "path": "/home/sourish/.local/share/zed/ty-custom/ty",
        "arguments": ["server"]
      }
    }
  }
}
```

`uv tool install ty` refers to upstream ty. Its receipt at
`~/.local/share/uv/tools/ty/uv-receipt.toml` must specify `name = "ty"` without a
local source directory or a fork Git URL. Never install the custom source or
wheel through `uv tool install`, and never replace the regular command or any
file in its uv tool environment during a fork deployment. Build wheels with
`uv build` and extract the executable directly into Zed's dedicated directory.

## Rebase and transfer

Both forks use the branch `semantic-tokens-basedpyright-parity`:

- `Sourish07/ruff` contains the Rust implementation; rebase it onto
    `astral-sh/ruff` main.
- `Sourish07/ty` contains the release package and Ruff submodule pin; rebase it
    onto `astral-sh/ty` main, then commit the updated Ruff pin. Keep `.gitmodules`
    pointing to `https://github.com/Sourish07/ruff` and the parity branch.

The laptop checkouts are `/Users/sourish/src/ty` and its `ruff` submodule.
Check both working trees, preserve existing changes, and create backup refs
before rebasing. Resolve conflicts while retaining the parity implementation.
Run the affected IDE/server tests, compilation checks, Clippy, and required
repository hooks. Review generated snapshots; a forced snapshot pass alone
does not establish correctness.

When pushing, push Ruff first so the ty submodule commit is available. Check
the current fork tips before using an explicit `--force-with-lease` for rebased
branches. Transfer both exact commits to `/home/sourish/src/ty-fork` and its
`ruff` submodule on sk-ml. Preserve remote working changes and verify that the
parent's recorded Ruff pin equals the submodule's HEAD before building.

## Build and deploy on sk-ml

Run the following in Bash on sk-ml after validating the source. It builds a
fresh wheel without installing it, stages the executable beside the destination,
and checks that the upstream binary and uv receipt remain unchanged.

```bash
set -euo pipefail
export PATH="$HOME/.cargo/bin:$HOME/.local/bin:$PATH"
cd /home/sourish/src/ty-fork

test "$(git rev-parse HEAD:ruff)" = "$(git -C ruff rev-parse HEAD)"
deploy_work=$(mktemp -d /tmp/zed-ty-deploy.XXXXXX)
zed_ty_dir="$HOME/.local/share/zed/ty-custom"
sha256sum "$HOME/.local/bin/ty" \
    "$HOME/.local/share/uv/tools/ty/uv-receipt.toml" \
    > "$deploy_work/upstream.sha256"

uv build --wheel --out-dir "$deploy_work/wheels"
mkdir -p "$zed_ty_dir"
python3 - "$deploy_work/wheels" "$zed_ty_dir/ty.new" <<'PY'
import sys
from pathlib import Path
from zipfile import ZipFile

wheels = list(Path(sys.argv[1]).glob("ty-*.whl"))
if len(wheels) != 1:
    raise SystemExit("Expected exactly one newly built ty wheel")
with ZipFile(wheels[0]) as wheel:
    binaries = [n for n in wheel.namelist() if n.endswith(".data/scripts/ty")]
    if len(binaries) != 1:
        raise SystemExit("Expected exactly one ty executable in the wheel")
    candidate = Path(sys.argv[2])
    candidate.write_bytes(wheel.read(binaries[0]))
    candidate.chmod(0o755)
PY

"$zed_ty_dir/ty.new" --version
sha256sum -c "$deploy_work/upstream.sha256"
if test -f "$zed_ty_dir/ty"; then
    cp -p "$zed_ty_dir/ty" "$zed_ty_dir/ty.previous"
fi
mv -f "$zed_ty_dir/ty.new" "$zed_ty_dir/ty"
"$zed_ty_dir/ty" --version
"$HOME/.local/bin/ty" --version
sha256sum -c "$deploy_work/upstream.sha256"
```

Keep the build output and commit IDs for provenance. Check the configured Zed
binary path again and smoke-test that executable's LSP initialization and
semantic-token responses, including the `callable` and `magic` modifiers.
Restart Zed's language server in the SSH project to load the new executable;
an already running server continues using its old executable.

To roll back, stage a copy of `ty.previous` beside `ty` and atomically rename it
over `ty`, then restart Zed's language server. Leave the upstream installation
alone during rollback too.

## Verified baseline: October 26, 2026

The deployed parity fork is ty `0.0.92` at parent commit
`25da18367` with Ruff commit `6b90bc64f9`. The regular uv-installed command is
upstream ty `0.0.92`, with a plain `name = "ty"` receipt. The dedicated custom
binary passed an LSP smoke check with 65 tokens, both custom modifiers, and no
diagnostics. Update the baseline after a successful deployment.
