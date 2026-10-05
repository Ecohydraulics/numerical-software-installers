# Recover from a failed installation

## Environment-loading errors

The error `etc/config.sh/setup: ... /bin/bash: cannot execute binary file` was caused by installer command arguments leaking into OpenFOAM's environment script. OpenFOAM interprets non-option arguments as settings or files to source; the installer accidentally supplied the command executable `/bin/bash`.

The subsequent `pop_var_context: head of shell_variables not a function context` error arose when `set -e` interrupted a nested source/eval/function chain. In the stock v2406 startup scripts, CGAL's optional GMP/MPFR path checks legitimately return 1 for unset paths. Treating these probes as fatal interrupts initialization; some Bash versions then emit the internal context warning. This is not evidence of a damaged archive or missing CGAL package.

The corrected installer sources OpenFOAM at shell top level with empty positional arguments and with `errexit` and `nounset` temporarily disabled. It restores the arguments and shell options, then checks the source status, selected project, API, and ABI before running build commands. Generated solver launchers use the same loader. Only project-level OpenFOAM preferences are loaded (`FOAM_CONFIG_MODE=o`); unrelated user/group preferences are excluded. Regression tests now exercise the actual checksum-pinned v2406 startup scripts, not only a simulated environment.

These failures occur in environment verification after source extraction but before the installer starts compilation. Downloaded archives and extracted sources should remain intact. Do not modify OpenFOAM's `etc/config.sh/setup`, `/bin/bash`, or their permissions. The corrected installer records probe stdout/stderr in `<prefix>/logs/environment-probe.log`, including failed attempts.

## Archive-name error

The archive error for `jouleHeatingSource:V` was caused by the installer's filename validator, not by Linux Mint or a damaged download. A colon is valid in a Linux/WSL filename. The corrected validator permits it on POSIX systems while retaining native Windows, traversal, and link-containment checks. Do not rename or omit the OpenFOAM file, or change archive checksums.

For this error, extraction stops during archive validation, before compilation. Its temporary extraction directory is removed; the verified archive remains in `cache/`. If `--install-system-packages` was used, APT dependencies and ParaView may already be installed. No existing OpenFOAM installation or shell startup file is changed.

## Preserve the failed installation and retry

Installer changes alter the installation identity. The corrected installer will therefore refuse the previous prefix. Preserve that directory and let the corrected installer create a fresh one; do not edit `.sediment-installer.json` to bypass the check.

For the failed `openfoam-sediment-v2406-shellfix` installation, leave the directory in place and run from the corrected installer directory:

```bash
python3 install.py --install-system-packages \
  --prefix "$HOME/.local/openfoam-sediment-v2406-initfix" \
  --source-cache "$HOME/.local/openfoam-sediment-v2406-shellfix/cache" \
  --examples --smoke-test
```

`--source-cache` reads the two core `.tgz` archives only; it does not reuse extracted sources, object files, or binaries. Each archive must match its pinned SHA256. Missing archives download into the new installation's cache; mismatched archives are retained and rejected. The old cache is not modified. This option is not used with `--reuse-openfoam`. Windows users can pass `-SourceCache` to `install.ps1`, using the WSL Linux path.

After successful installation, activate the new prefix:

```bash
source "$HOME/.local/openfoam-sediment-v2406-initfix/activate.sh"
```

The old prefix remains available for diagnosis. Retain any original options such as `--jobs`, `--skip-visualization`, or `--visit-platform` as needed.

Alternatively, move a failed installation aside to reuse its original path. The following commands apply to the **default** installation directory. If `--prefix` was used, replace `install_prefix` with that exact dedicated directory. Exit any activated installer shell before moving an installation.

```bash
install_prefix="$HOME/.local/openfoam-sediment-v2406"
if [ -d "$install_prefix" ] && [ ! -L "$install_prefix" ] \
   && [ -f "$install_prefix/.sediment-installer.json" ] \
   && [ ! -L "$install_prefix/.sediment-installer.json" ]; then
    recovery_dir="$(mktemp -d "${install_prefix}.failed-XXXXXXXX")" \
      && mv -T -- "$install_prefix" "$recovery_dir/install" \
      && printf 'Failed installation preserved at: %s\n' "$recovery_dir/install"
else
    printf 'No marked installation at this path; nothing moved.\n' >&2
fi
```

This is a recoverable move, not a deletion. It retains downloaded archives, logs, and any case files. If the move succeeded, retry from the corrected installer directory:

```bash
python3 install.py --install-system-packages --examples --smoke-test \
  --source-cache "$recovery_dir/install/cache"
```

Retain any original options, including a custom `--prefix` or `--jobs`. For an installation that used `--reuse-openfoam`, retain that option and omit `--source-cache`. The saved failed directory need not be deleted before retrying.

## System packages

`apt-get` remains the recommended APT interface for scripts because it preserves backward compatibility. `apt` is the interactive interface; using it manually is appropriate. See the [Debian APT manual](https://manpages.debian.org/bookworm/apt/apt.8.en.html#SCRIPT_USAGE_AND_DIFFERENCES_FROM_OTHER_APT_TOOLS).

Moving the user installation does **not** undo APT installations or upgrades. Inspect the relevant transaction first:

```bash
less /var/log/apt/history.log
```

Older transactions may be in rotated or compressed history files. The `Install` and `Upgrade` entries distinguish newly added packages from upgraded ones. Retain dependencies needed by other applications; do not remove the installer's entire dependency list or run an unreviewed `autoremove`. Exact reversal of package upgrades requires the previous package versions or a preinstallation system snapshot. Leaving these dependencies installed is sufficient for retrying the corrected installer.
