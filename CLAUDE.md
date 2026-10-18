# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `salt-formulas/CLAUDE.md` describes the fork collection, the shared
toolchain (Kitchen, pre-commit, CI) and the `template-formula` layout. This file
only records where `vim-formula` departs from it.

## Commands

```bash
bundle install
bin/kitchen list                                 # 3 instances, one suite: default
bin/kitchen test default-debian-13-3008-2        # destroy + converge + verify + destroy
bin/kitchen verify default-debian-11-3006-27     # InSpec controls only
pre-commit run --all-files
```

There is no unit test layer: the only tests are the InSpec controls in
`test/integration/default/controls/`, run against a converged container.

## Not a template-formula layout

This formula predates the template and was never restructured. Do not look for
`defaults.yaml`, `os*map.yaml`, `libtofs.jinja` or `package/`/`config/`/`service/`
subdirectories — none exist.

- `vim/map.jinja` is a single inline `grains.filter_by` keyed by `os_family`,
  merged with the `vim:lookup` pillar. It carries the package name(s),
  `share_dir`, `group`, `config_root` and, for Debian only, the `alternatives`
  entry. A per-distro default is changed there, not in a YAML map.
- `vim/init.sls` is the whole formula in practice: `pkg.installed` plus
  `<config_root>/vimrc` from `vim/files/vimrc` (no TOFS, direct `source:`),
  skipped when `vim:managed_vimrc` is false. On Debian that path is
  `/etc/vim/vimrc`, elsewhere `/etc/vimrc`.
- `vim/files/vimrc` reads the pillar directly (`vim:config`, `settings`,
  `mappings`, `lets`, `allow_localrc`), not through `map.jinja`. Rendering rules:
  keys are emitted sorted; a `settings` key with a null value (`~`) becomes
  `set <key>`, otherwise `set <key>=<value>`; nested `config` mappings are
  flattened into a space-joined prefix (`autocmd FileType sls …`), lists into one
  line per item. `vimrc.local` is sourced only when `allow_localrc` is true.
  This file is rendered verbatim onto minions, so it must not get the
  `# -*- coding` / modeline header — neither do the legacy `.sls` files.
- `vim/_mapdata/` exists and runs first in the Kitchen `state_top`, but no
  control compares its dump: there is no `files/_mapdata/` reference.

## Production usage and its traps

Per the audit of 2026-10-17 (`AUDIT.md`, untracked): every minion includes
`vim` only, i.e. `vim/init.sls`, with the pillar keys `managed_vimrc`, `config`
and `settings`. Nothing else in `vim/` reaches a host.

- `vim.editor` uses the `alternatives.install` / `alternatives.auto` **state**
  module, absent from Salt 3008.x (the execution module remains). Including it
  would fail on the 3008 minions.
- `vim.nerdtree`, `vim.pyflakes` and `vim.salt` `file.recurse` bundled plugins
  from `vim/files/` into `share_dir`. They are years old; the pyflakes copy is
  Python 2. Not included anywhere.
- The InSpec controls assert only the package and two lines of the vimrc
  (`syntax on`, `set number`), which come from `test/salt/pillar/vim.sls`. A
  change to that pillar must be carried into `config_spec.rb`.
