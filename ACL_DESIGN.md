# ACL and permission model

This document tells how one `nginx` user reads the files of all sites, while each
site stays isolated from the other sites. No file has world-readable bits. The
document also gives the conditions that keep this model correct when WordPress
writes files. The failure modes below occurred during the migration from Debian 12
to Debian 13.

## Goals

- Each site has its own UID: site `<s>` runs as user `<s>` with group `<s>`. The group has one member.
- The shared `nginx` user can **read** the files of each site and **traverse** its directories.
- One site cannot read the files of a different site: site `A` cannot read site `B`.
- No file or directory under `/srv/www` has world bits (`other::---`).
- WordPress installs, updates and uploads from the dashboard operate correctly. These writes run as the site user.

## Layout

| Path | Mode | Owner | Notes |
|---|---|---|---|
| `/home/<s>` | `0750` | `<s>:<s>` | Login and SFTP only. No web content. |
| `/srv/www/<s>` | `0750` | `<s>:<s>` | Web root. |
| Directories in the web root | `0750` | `<s>:<s>` | |
| Files in the web root | `0640` | `<s>:<s>` | |
| `wp-config.php` | `0440` | `<s>:<s>` | nginx also has a `deny all` rule for this file. |

nginx reads the files through a **POSIX ACL**. It is not a member of the site group.
- The **access ACL** `user:nginx:rX` applies to the current tree.
- The **default ACL** `default:user:nginx:rX` applies to new files and directories. They get this entry when the system makes them.

The isolation does **not** come from the group bits. It comes from three items:

- `other::---` on all files and directories.
- One php-fpm pool for each site, with `user = <s>`.
- One socket for each site: `/run/php/<s>-fpm.sock`, mode `0600`, owner `nginx:nginx`.

The `group::` entry gives no access to other users, because the site group has
only one member: the owner. Refer to the single-member group condition below.

## How to read `ls -l` on this tree

When the `ls -l` line of a file ends with `+`, the file has an ACL. In this case, the
**middle triad does not show the group permission**. It shows the ACL *mask*. The mask
is the maximum access that a named user (for example `nginx`) or the group can get.

Thus `-rw-r-x---+` does **not** mean that a user can execute the file. It means that
the mask contains `x`. The owner has the permission `rw-` on this file.

`ls --color` shows a file in green when the displayed bits contain `x`. On a file with
an ACL, the displayed bits are the mask. Thus **a green file has a mask with `x`**. The
color has no other meaning. On these files, `chmod` also changes the mask,
not the `group::` entry. Use `getfacl -p <path>` to see the correct permissions.

Compare the mask to the width of a door. The width does not tell which person can go
through the door.

**The source of the `x`.** The default ACL of the site directory is the template for
all new files. The command `setfacl ... user:nginx:rX` made this template. A capital
`X` gives execute permission only to a directory or to a file that is already
executable. `setfacl` applies this test one time, to the *directory*. The directory
passes the test, so the template contains the literal entry `default:user:nginx:r-x`.

Each new file gets a copy of this entry, `x` included. One default ACL entry applies to
files and to subdirectories. It cannot give `x` to directories only.

**The `x` gives no access.** nginx serves a file when it reads the file
(`read()`/`sendfile()`). nginx does not execute file content. php-fpm runs PHP code. It
reads the *content* of the file and does not examine the execute bit. Thus an `x` on a
`.css`, `.php` or `.jpg` file gives no permission to a user.

**The mask is the important value.** The mask controls what `ls` shows. It also
controls the maximum access for nginx. A correct file has these entries:

```
$ getfacl -p /srv/www/example/index.php
user::rw-
user:nginx:r-x        #effective:r--
group::r-x            #effective:r--
mask::r--
other::---
```

The `user:nginx` entry contains the inherited `x`, but `mask::r--` limits it to `r--`.
Thus nginx can read the file, the ACL gives no execute permission, and other users
cannot read the file. `ls` does not show the file in green. When a file shows in green,
its mask contains `x`. Correct the mask. Do not change the content of the file.

## The three actors that change the mask

All changes that can break this model change the **mask**. Three actors can change
the mask, and each actor needs a different control:

1. **`setfacl` (Ansible ACL tasks).** A plain `setfacl -m u:nginx:rX` calculates the
   mask again as the union of `group::` and the named entries. The result is `r-x`,
   and the files show in green.
   **Control:** Use `recalculate_mask: no_mask` (setfacl `-n`) on each recursive
   ACL task. These tasks are in `2-create-users.yaml` (access ACL) and in
   `standalone_playbooks/5-rsync-migrate.yaml` (after the sync). With `-n`, each file
   keeps the mask that it had when the system made it (`r--` from `0640`).
   **`-n` prevents the problem, but it does not repair it.** It does not decrease a
   mask that is already `r-x`. To repair a tree with green files, use the reset
   command below.

2. **`chmod`.** `chmod` writes the mask directly. Use `chmod` to reset a tree with
   green files. The migration also uses this method to set the permissions (rsync
   `--chmod=F640`).

3. **WordPress.** WordPress can change permissions that the ACL does not control.
   `WP_Filesystem_Direct` applies `chmod` to each file that it writes. The value is
   `FS_CHMOD_FILE`, and the default is `(fileperms(index.php) & 0777) | 0644`. This
   default gives files the mode **`0644`**, so all users can read them
   (`other::r--`). Also, when `index.php` is green, WordPress copies the `x` to all
   files that it writes.
   **Control:** Set fixed values in `wp-config.php`. `5a-wordpress.yaml` sets
   these values:
   ```php
   define('FS_CHMOD_FILE', 0640);
   define('FS_CHMOD_DIR',  0750);
   ```
   Use **plain octal values**. Do not use `0644 & ~umask()`, the form in the WordPress
   documentation. With that form, the file mode depends on the umask of the php-fpm
   pool. A strict umask of 077 gives the mode `0600`, and then nginx cannot read the
   files. A constant value prevents this problem.

The model stays correct only when all three actors have a control. When only
`setfacl` has a control, the next WordPress core or plugin update adds the
permissions again.

## Inheritance continues without Ansible

The default ACL applies to files from all UIDs, root included. It also applies at each
directory level. Thus a directory that WordPress makes (for example
`wp-content/uploads/2026/09/`) also gets the default ACL. The *next* file in that
directory gets nginx read access and `other::---`. No Ansible run is necessary.

A test on the live server showed this result. A media upload got `mask::r--`,
`other::---` and an effective `r--` for `user:nginx`. nginx served the file with status
`200`. Because `other::---` blocks world read, nginx can read the file only through
the ACL.

## Failure modes

- **A tree with green files does not repair itself.** `no_mask` does not decrease a
  current `r-x` mask. When you run rsync again (`-rt`, without `-p`), it does not
  change files that are the same. Run this reset command one time:
  ```bash
  find /srv/www -type f -perm /g+x ! -perm /u+x -exec chmod g-x {} +
  ```
  The command removes group `x` only from files that the owner cannot execute. It does
  not change executable files or `0600` files. A new installation does not have this
  problem, because all files start with a correct mask.
- **Do not run `setfacl -m` without `-n` on this tree.** The inherited `x` makes all
  files green again. The Ansible ACL tasks have a comment about this rule.
- **The site group must have only one member.** `group::r-x` gives no access to other
  users only because the site group has one member, the owner. `/etc/group` controls
  this condition. When you add a user, or a php-fpm pool with a different UID, to the
  site group, that user can immediately read the full tree.
- **`move_uploaded_file()` and `/tmp` on the same filesystem.** PHP moves an upload
  from `/tmp` with `rename()`. When `/tmp` and `/srv` are on the same filesystem, the
  moved file keeps the inode that the system made outside the default-ACL tree. The
  file has no `user:nginx` entry, and nginx returns 403 until the next Ansible run.
  Debian 13 puts `/tmp` on **tmpfs**. Thus the move crosses filesystems, PHP *copies*
  the file, and the copy gets the inherited entry. To examine `/tmp`, use
  `findmnt /tmp`. If `/tmp` is not tmpfs, set `upload_tmp_dir` to a directory in the
  web root.
- **Files without the nginx entry.** These operations make files without the
  `user:nginx` entry: `mv` or `rename()` from outside the tree, coreutils `install`,
  `rsync -A` from a source without ACLs, and `setfacl -b` or `-k`. nginx cannot read
  these files until the next ACL task. These operations keep the inherited entry:
  `cp`, `cp -a`, `tar -x`, `unzip`, shell redirects and `touch`.
- **A directory with mode `0700`.** A directory that you make with `mkdir -m 700` or
  `mktemp -d` gets `mask::---`. nginx cannot traverse it, so nginx cannot read **any
  file below it**. The permissions of those files have no effect. To repair the
  directory, run `chmod 750` on it. A plain `mkdir` in the tree is correct, because the
  default ACL has priority over the umask.
- **The `wp-config.php` anchor in a translated installation.** `wp config set` uses
  the English line `/* That's all, stop editing! */` to find the position for new
  constants. A zh_CN or ja installation has a translated line, and `--add` fails with
  "Unable to locate placement anchor." To repair this, add the English anchor line in
  the **source** files. The line then stays after the final rsync.

## Verification

`7-validate.yaml` runs these checks for all sites at the end of `main.yaml`. To run the
checks by hand on the server, use these commands:

```bash
# per-file truth
getfacl -p <path>                                  # expect mask::r--, other::---, nginx eff r--

# whole-tree audits — each should print NOTHING
find /srv/www -type f -perm /g+x ! -perm /u+x      # green/executable data files
find /srv/www -perm /o+rwx                          # any world access
find /srv/www \( ! -user <s> -o ! -group <s> \)     # wrong owner (per site root)

# nginx actually reads a WP-written file (expect 200, proves ACL not world-read)
curl -skI --resolve <domain>:443:127.0.0.1 https://<domain>/wp-content/uploads/<y>/<m>/<f> | head -1
```

## Non-goals

- This model does not use containers or namespaces for each site. One administrator
  operates this server for trusted friends. It is not a paid hosting service. The
  isolation comes from the UID, the ACL, the php-fpm pool and the socket.
- This model does not give `r--` to files and `r-x` to directories through
  inheritance. One default ACL entry cannot do this. The model accepts the inherited
  `x`, which gives no access (refer to the sections above).
