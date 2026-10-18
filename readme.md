# asbl-go_bin_cont_builder

## Description

An ansible role that builds a go binary from source inside a throwaway
`golang` docker container, then installs the compiled binary on the target
host with a configurable destination path, owner, group and mode. No go
toolchain is ever installed on the target host.

Source can come from either:

- a tarball/zip archive URL, downloaded and extracted on the target host
  (`go_bin_cont_builder_source_type: "url"`)
- a local directory on the Ansible controller, synchronized to the target
  host (`go_bin_cont_builder_source_type: "local"`)

## Prerequisite

- **Docker must already be installed and running on the target host.**
  This role does not install Docker itself (see `asbl-docker_install` for
  that); it only runs `docker version` up front to fail fast with a clear
  message if Docker is missing, rather than failing deep inside the build
  step.
- The build is a single `docker run --rm ...` invocation per role call:
  this role builds exactly one binary per invocation. To build several
  binaries, call it multiple times (e.g. via `include_role` in a loop) with
  different vars.

## Usage

### Use role

- Add the role git source in "requirements.yml" file :
```
  - name: role_name
    scm: git
    src: https://github.com/tiny-company/<repository_name>.git
    version: main
```

- And then use the galaxy command to load the file dependencies :
```
ansible-galaxy install -r requirements.yml
```

- Or manually get the playbook as collection (with ansible-galaxy) :
```
ansible-galaxy collection install https://github.com/tiny-company/<repository_name>.git
```

- finally use role in playbook : [see test_playbook example](./test_playbook.yml)

### Variables

For an exhaustive list of variables check the [defaults](defaults/main.yml)
file. Ideally, all values will have commentaries describing what are their
purposes and by the default value you can tell the type.

Notable ones:

- `go_bin_cont_builder_name`: namespaces staging/cache paths and defaults
  the output binary name
- `go_bin_cont_builder_golang_version`: **required, no default** - the
  exact `golang` image tag to build with (e.g. `"1.23-bookworm"`), so
  builds never silently drift onto a newer toolchain
- `go_bin_cont_builder_source_type` / `_source_url` / `_source_local_path`:
  where the go source comes from
- `go_bin_cont_builder_dest` / `_dest_owner` / `_dest_group` / `_dest_mode`:
  where the compiled binary is installed, and with what ownership/
  permissions
- `go_bin_cont_builder_build_uid` / `_build_gid`: the build itself runs as
  this non-root uid/gid inside the (otherwise root-capable) golang image,
  as a hardening default
- `go_bin_cont_builder_cache_enabled` / `_cache_dir`: persists the go
  module and build cache across runs so repeat builds do not re-download
  every dependency

## Design notes / known limitations

- Extracting a URL archive always wipes and recreates the build staging
  directory first (a tarball is a full snapshot, not an incremental
  sync), so nothing from a previous build lingers. Local-directory sync
  instead uses `rsync --delete`, which achieves the same end result
  incrementally.
- If the archive's content sits one directory level below the archive
  root (common for auto-generated GitHub source tarballs, e.g.
  `reponame-1.0.0/...`), set
  `go_bin_cont_builder_source_url_extra_opts: ["--strip-components=1"]`.
  This only works for tar-based archives; `.zip` archives do not support
  an equivalent flag via `unzip`, so a zip source with a wrapping
  directory will need `go_bin_cont_builder_build_subdir` adjusted instead.
- The downloaded archive is verified via `go_bin_cont_builder_source_url_checksum`
  *before* extraction when set (strongly recommended) - it is fetched with
  `ansible.builtin.get_url` rather than letting `unarchive` download it
  directly, specifically so the checksum can be checked first.
- `go_bin_cont_builder_build_subdir` changes the container's working
  directory and builds `.` from there, rather than staying at the source
  root and passing the subdir as a package-path argument - this is what
  makes building a subdirectory that is its own separate nested Go module
  (its own `go.mod`, e.g. `golang.org/x/tools/gopls` inside the
  `golang.org/x/tools` monorepo) actually work; the naive "stay at the
  root, build `./subdir`" approach fails for those with `main module
  (...) does not contain package ...`, since Go excludes nested-module
  directories from the parent module's package tree.
- The `docker run` invocation is always built as an argument list, never a
  shell string, so none of the role's variables are ever shell-interpreted.

## Source :

- [golang docker images](https://hub.docker.com/_/golang)
- [ansible.builtin.get_url module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html)
- [ansible.builtin.unarchive module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)
- [ansible.posix.synchronize module](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html)
