The bash script `.devcontainer/bin/simplify-fetchers.sh`:

Supports two modes:
- without CLI parameters
- with `--rollback`

## Without the parameter

It copies the following files from the git branch `origin/no-merge/reduce-fetchers`:
- `apps/indexer/lib/indexer/block/realtime/fetcher.ex`
- `apps/indexer/lib/indexer/fetcher/token_balance.ex`
- `apps/indexer/lib/indexer/fetcher/token.ex`
- `apps/indexer/lib/indexer/transform/token_transfers.ex`
- `apps/indexer/lib/indexer/block/catchup/bound_interval_supervisor.ex`
to the same subdirs of the `/workspace` directory (configured through WORKSPACE variable within the script)

Applies `git update-index --assume-unchanged` for every copied file.

## With `--rollback`

It applies `git update-index --no-assume-unchanged` for every file copied in the previous mode.

It checks by `diff` that there are no differences between the file within `origin/no-merge/reduce-fetchers` and the file in the current workspace.

For those files where there is no diff, it does `git checkout`.

It reports in the terminal about files that was not rolled back due to discovered difference.
