---
title: "Actions"
date: 2019-06-28
draft: false
disableToc: false
weight: 9
---

# Actions

You can probably find yourself in a situation where some part of the packit workflow needs to be
tweaked for your package.

Packit supports actions, a way to change the default implementation for a command
of your choice.  Packit is able to execute multiple commands. Each action accepts
a list of commands. By default, the commands are executed directly and
not in a shell - if you need a shell, just wrap your command like this: `bash -c
"my fancy $command | grep success"`.

All actions are also executed inside Packit-as-a-Service. The service
creates a new sandbox environment where the command is run.

Actions have a default behaviour which you can override, hooks don't have any -
hooks are a way for you to perform operations following a certain packit event,
e.g. cloning an upstream repo.

Currently, these are the actions you can use:

## `propose-downstream` command

|        | name                  | working directory | when run                                                                          | description                               |
| ------ | --------------------- | ----------------- | --------------------------------------------------------------------------------  | ----------------------------------------- |
| [hook] | `post-upstream-clone` | upstream git repo | after cloning of the upstream repo (main) and before other operations             |                                           |
| [hook] | `pre-sync`            | upstream git repo | after cloning and checkout to the correct (release) branch                        |                                           |
|        | `prepare-files`       | upstream git repo | after cloning, checking out of both upstream and dist-git repos                   | replace patching and archive generation   |
|        | `create-patches`      | upstream git repo | after sync of upstream files to the downstream                                    | replace patching                          |                                         | replace the code for creating an archive  |
|        | `get-current-version` | upstream git repo | when the current version needs to be found                                        | expect version as a stdout parameter      |


## Creating SRPM

These apply to the `srpm` command and building in COPR.

|        | name                  | working directory | when run                                                                          | description                               |
| ------ | --------------------- | ----------------- | --------------------------------------------------------------------------------  | ----------------------------------------- |
| [hook] | `post-upstream-clone` | upstream git repo | after cloning of the upstream repo (main) and before other operations             |                                           |
|        | `get-current-version` | upstream git repo | when the current version needs to be found                                        | expect version as a stdout                |
|        | `create-archive`      | upstream git repo | when the archive needs to be created                                              | replace the code for creating an archive  |
|        | `create-patches`      | upstream git repo | after sync of upstream files to the downstream                                    | replace patching                          |
|        | `fix-spec-file`            | upstream git repo | after creation of a tarball and before running rpmbuild command                   | this action changes spec file to use the new tarball                          |

**create-archive** - is expected to return a relative path to the archive - relative within the repository. If there are more steps, then one of them has to return the archive name.

**fix-spec-file** — this action updates the specfile so it's possible to have the spec reference
                    the tarball and unpack it. This method tries to perform 3 operations on a spec file:

1. It replaces Source configured by [`spec_source_id`](/docs/configuration/#spec_source_id) (default `Source0`) with a local path to the generated tarball
2. It changes the first `%setup` (or `%autosetup`) macro in `%prep` and adds `-n` so the generated
 tarball can be unpacked (it tries to extract the directory name directly from the archive
 or uses the configured [`archive_root_dir_template`](/docs/configuration#archive_root_dir_template))
3. It changes %version

For example a package may define multiple Sources. In such cases, the
default implementation of `fix-spec-file` won't be able to update `%prep`
correctly. You can write a simple shell script which will be run with the `cwd` set to
the root of the repository and use `sed` to set the new
Sources correctly, e.g. `sed -i my_specfile_path -e
"s/https.*only-vendor.tar.xz/my_correct_tarball_path/"`


### Environment variables set by packit

Additionally, packit sets a few env vars for specific actions

**fix-spec-file**

`PACKIT_PROJECT_VERSION` — current version of the project (coming from `git describe`)
`PACKIT_PROJECT_COMMIT` — commit hash of the top commit
`PACKIT_PROJECT_ARCHIVE` — expected name of the archive

**create-archive**

`PACKIT_PROJECT_VERSION` — current version of the project (coming from `git describe`)
`PACKIT_PROJECT_NAME_VERSION` — current name and version of the project (coming from `git describe`)


-----

In your package config they can be defined like this:

```yaml
specfile_path: package.spec
synced_files:
  - packit.yaml
  - package.spec
upstream_package_name: package
downstream_package_name: package
dist_git_url: https://src.fedoraproject.org/rpms/package.git
actions:
  prepare-files: "make prepare"
  create-archive:
  - "make archive"
  - "ls"
```
