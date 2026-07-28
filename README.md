Role Name
=========

A brief description of the role goes here.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

The `sync` deployment strategy uses `ansible.posix.synchronize` (shipped with
the `ansible` package, otherwise `ansible-galaxy collection install
ansible.posix`) and needs `rsync` on the target, which the role installs.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

### Deployment strategy

`deploy_archive_strategy` controls how the archive content is put in place:

| value | behaviour |
| --- | --- |
| `overlay` (default) | The archive is extracted on top of `deploy_archive_dest`. Existing files are overwritten, but files that were removed from the archive since the previous release are **left behind** on the target. |
| `sync` | The archive is extracted into `deploy_archive_stage_dir` then `rsync --archive --delete`'d into `deploy_archive_dest`, so the destination ends up matching the archive exactly. Requires `rsync` on the target (installed by the role). |

`sync` deletes everything in `deploy_archive_dest` that is not in the archive,
so runtime state living inside the destination (dependencies installed on the
target, logs, uploads, generated code, ...) must be declared in
`deploy_archive_keep`. Patterns follow rsync `--exclude` semantics and are
relative to `deploy_archive_dest`: a leading `/` anchors the pattern to the root
of the destination, without it the pattern matches at any depth. The uploaded
archive and the config file managed by this role (`deploy_archive_conf`) are
protected automatically.

```yaml
deploy_archive_strategy: "sync"
deploy_archive_keep:
  - "/backend/node_modules"   # installed on the target, not shipped
  - "/backend/logs"           # runtime state
  - "/uploads"
```

Three safety nets apply to `sync`: a staging directory left over by an
interrupted run is discarded instead of reused, the role fails rather than
running `rsync --delete` when the archive extracted to an empty tree, and rsync
runs as `deploy_archive_appuser.name` instead of root, so `--delete` can only
reach what that user is allowed to unlink.

That last one is worth understanding: on POSIX, unlinking depends on write
permission on the *parent directory*, not on the file's owner. Since the
destination and the extracted tree belong to the app user, files that root
dropped into an app-user-owned directory are still deletable. What the
unprivileged rsync does protect is anything sitting in a directory the app user
cannot write — a root-owned subdirectory inside the destination, for instance,
where rsync fails loudly rather than removing it:

```
rsync: [generator] delete_file: unlink(rootstuff/secret.txt) failed: Permission denied (13)
rsync error: some files/attrs were not transferred (code 23)
```

Add such a path to `deploy_archive_keep` (or hand it to the app user) to make the
deployment pass again. Note that the run aborts where it failed: files already
transferred or deleted before that point are not rolled back.

Note that `deploy_archive_appuser.home` defaults to `deploy_archive_dest`, so
`useradd` copied `/etc/skel` (`.bashrc`, `.profile`, `.bash_logout`) in there when
the user was created. Those are not in the archive, so the first `sync` removes
them; add them to `deploy_archive_keep` if the app user needs a usable shell.

With `deploy_archive_dest_permissions`, the recursive chmod is applied to the
extracted tree: the destination itself under `overlay`, the staging directory
under `sync` — where rsync then carries the modes over, leaving the kept runtime
state untouched.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
