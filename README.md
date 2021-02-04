# restic-run
Simplify running of [restic](restic.net) commands with pre-configured environments.

## Introduction

`restic-run` provides a simplified way to run restic commands against a restic repository by storing per-repository configuration parameters in a manner easily referenced with a simple unique identifier at runtime.

The primary motivations for creating `restic-run` are:

- easily specify repo-specific parameters/configuration at runtime.
- provide simplified management of restic repos whether interactively or via automated mechanisms such as cron/systemd.
- reusable program that provides optional detailed logging and optional email notifications.

Instead of doing this:
```text
export RESTIC_REPOSITORY=sftp:remote-host:/path
export RESTIC_PASSWORD=...password...
restic backup ${HOME} --bunch -of -args ...
```
use `restic-run` and a sample repository configuration named _foorepo_:

`restic-run foorepo backup`

Reduction in time and effort is fully realized when working with multiple respositories.

## Usage

`usage: restic-run [-i] [-l] [-m | -M] {reponame} {restic_cmd} [restic_args ...]`

- `-i` - install `restic-run` config file framework. Typically done when first setting up `restic-run`.
- `-l` - log to `${LOGDIR}/${reponame}`
- `-m` - mail log if restic exits with a non-zero return code.
- `-M` - always mail output, regardless of restic return code.

- `reponame` - name of directory in `~/.config/restic/` containing config files for a particular repository.
- `restic_cmd` - restic command to run such as `backup` or `purge`.
- `restic_args` - passed, verbatim, to `restic` after `{restic_cmd}`. [OPTIONAL]

## Repository Configuration

Each repository configuration is stored in a separate directory, `~/.config/restic/{reponame}/`, containing the following files:

- `{restic_cmd}_args.sh` - arguments specified in the array `args` to pass to restic when `restic ${restic_cmd}` is run, e.g. `backup_args.sh` or `prune_args.sh`. [OPTIONAL]

- `env.sh` - sourced by `restic-run`, contains exported environment such as `RESTIC_REPOSITORY` or `RESTIC_PASSWORD`. [_technically OPTIONAL, but probably want to at least specify `RESTIC_REPOSITORY` otherwise why are you using `restic-run`?_]
	See the [restic environment variables docs](https://restic.readthedocs.io/en/latest/040_backup.html#environment-variables).

- `passwd` - repo password. If `RESTIC_PASSWORD` is not defined in `env.sh` the presence of this file will cause `RESTIC_PASSWORD_FILE` to be defined and exported in the shell. [OPTIONAL]

If the following files are present, their absolute paths are exported as the variables shown in brackets which can be referenced in `{restic_cmd}_args.sh`.

- `includes` - files to back up (`--files-from`) [`INCLUDES`].
- `excludes` - exclude patterns (`--exclude-file`) [`EXCLUDES`].
- `iexcludes` - case-insensitive exclude patterns (`--iexcludes-file`) [`IEXCLUDES`].

Example: if the file `excludes` exists in `backup_args.sh` one could:

`args+=("--exclude-file=${EXCLUDES}")`

An optional global environment configuration file, `~/.config/restic/glob_env.sh`, will always be sourced regardless of the value of `{restic_cmd}` for the purpose of exporting the following variables:

- `MAILTO` - email address where logs are sent, if enabled with `-m` or `-M`.
- `LOGDIR` - directory where restic-run logs are stored, if enabled with `-l`.

## Example Repo Configuration

### SFTP-Based Repository

Every config will source the global environment file:

#### `~/.config/restic/global_env.sh`
```text
#
# glob_env.sh - Global environment file for restic-run.
#
# This file is always read regardless of which restic command is
# specified. These variables can be overridden in the
# repo-specific env.sh file.
#

# Recipient for emailed logs if '-m' or '-M' is specified.
#export MAILTO=

# Directory to write restic logs if '-l' is specified.
export LOGDIR=/var/log/restic
```

My repo name is _sample_ and contains related environment variables:

#### `~/.config/restic/sample/env.sh`
```text
# This file is sourced by restic-run and typically contains
# restic-specific environment variables. See the following URL for a
# comprehensive list:
# https://restic.readthedocs.io/en/latest/040_backup.html#environment-variables
#
export RESTIC_REPOSITORY=sftp:samplerepo:/restic/sample
export RESTIC_PASSWORD=blippity-bloop-bloop-blargh
```

The `includes` file contains all the files/paths that I want to back up to this repository. The path to this file will be defined in the environment variable `INCLUDES` and can be used in `{cmd}_args.sh` files:

#### ~/.config/restic/sample/includes
```text
/home
/root
/etc
```

The `excludes` file contains all the files/paths that I want to exclude from this repository. The path to this file will be defined in the environment variable `EXCLUDES` and can be used in `{cmd}_args.sh` files:

#### ~/.config/restic/sample/excludes
```text
tmp
TMP
Tmp
temp
Temp
TEMP
cache
Cache
CACHE
trash
Trash
downloads
Downloads
```

When backing up, these are the restic args I prefer. Note the use of the environment variables `INCLUDES` and `EXCLUDES`:

#### `~/.config/restic/sample/backup_args.sh`
```text
# This file is sourced by restic-run if the {restic_cmd} is 'backup'.
# Append arguments to pass to 'restic backup' to the 'args' array.
#
args+=("--verbose" "--one-file-system" "--exclude-caches=true")
args+=("--cleanup-cache" "--exclude-if-present" ".nobackup")
args+=("--exclude-file" "${EXCLUDES}")
args+=("--files-from" "${INCLUDES}")
```

When viewing stats I like the "raw-data" mode, but don't like having to specify it every time:

#### `~/.config/restic/sample/stats_args.sh`
```text
# This file is sourced by restic-run if the {restic_cmd} is 'stats'.
# Append arguments to pass to 'restic stats' to the args array.
args+=("--mode=raw-data")
```

With the configuration files in place, perform a backup:

`restic-run sample backup`

Get some stats:

`restic-run sample stats`

List last snapshot:

`restic-run sample snapshots --last`