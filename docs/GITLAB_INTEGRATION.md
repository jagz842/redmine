# Add another GitLab repository to Redmine

Use this procedure to connect a repository on your separate GitLab server to
an existing or new Redmine project. Repeat it for each additional repository.
GitLab hosts the source; the Redmine host fetches a local Git mirror, which
Redmine reads through the existing read-only Docker mount:

```text
GitLab server → SSH fetch on Redmine host → /var/lib/redmine/repos/new-repo.git
                                               ↓ read-only Docker mount
                                          Redmine project
```

The checked-in `compose.yml` already mounts `/var/lib/redmine/repos` at the
same path inside Redmine. Adding another mirror there needs no Compose change
or container restart. This guide assumes that mount is active in your running
deployment. It does not require a GitLab integration plugin or webhook.

## 1. Choose the repository and host account

Run commands on the **Redmine server**, as the same Linux account that owns
and updates your existing mirrors. Use that account's crontab too. Do not
clone with `sudo`: that would use root's SSH keys and create root-owned files.
Use `sudo` for Docker commands only if your account needs it.

Replace these example values throughout the guide:

| Example | Replace with |
| --- | --- |
| `gitlab.example.internal` | Your separate GitLab server's hostname or IP |
| `group/subgroup/new-repo.git` | The repository path from GitLab's **Clone with SSH** URL |
| `new-repo` | A unique local mirror name, also used in lock/log filenames |
| `/path/to/redmine` | The deployment checkout containing `compose.yml` and `.env` |

Use a unique mirror name even when repositories in different GitLab groups
share the same project name. The Redmine host must be able to resolve and
reach the GitLab server's SSH port.

## 2. Give the host read access in GitLab

If the existing host SSH key is already authorized for the new repository,
keep using it and skip key creation. Access to one GitLab repository does not
automatically grant access to another.

For a dedicated key, create it on the Redmine host. Choose a different filename
if this one already exists; do not overwrite an existing key:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/redmine_gitlab_new_repo -C 'Redmine new-repo mirror'
cat ~/.ssh/redmine_gitlab_new_repo.pub
```

For unattended fetching, the key must work without a password prompt or an
interactive SSH agent. If your policy allows it, use an empty passphrase for
this read-only deployment key and protect the private file on the host.

In the **new GitLab project**, open **Settings → Repository → Deploy keys**.
Add the public key, give it a descriptive title, and leave write access
**disabled**. A Maintainer or Owner can configure the deploy key. If you are
reusing a deploy key from another project, enable it for this project through
the available deploy keys instead.

For the dedicated key, add this separate alias to `~/.ssh/config`, preserving
any existing entries:

```sshconfig
Host gitlab-redmine-new-repo
    HostName gitlab.example.internal
    User git
    Port 22
    IdentityFile ~/.ssh/redmine_gitlab_new_repo
    IdentitiesOnly yes
    BatchMode yes
```

Replace `Port 22` if GitLab uses a custom SSH port. Use the SSH username from
your GitLab clone URL if it differs from `git`. Then set permissions:

```bash
chmod 600 ~/.ssh/config ~/.ssh/redmine_gitlab_new_repo
```

Establish host trust interactively once:

```bash
ssh -T gitlab-redmine-new-repo
```

Before accepting the host key, compare its fingerprint with the GitLab
administrator's trusted value. GitLab normally greets you rather than giving
a shell. The repository access test below is the decisive check:

```bash
git ls-remote git@gitlab-redmine-new-repo:group/subgroup/new-repo.git
```

If using existing SSH configuration, use the original **Clone with SSH** URL
instead of the alias in every command below. Never copy the private key into
GitLab, the checkout, or the Redmine container.

## 3. Create the new mirror on the Redmine host

The existing `/var/lib/redmine/repos` directory should already be writable by
your mirror account. Check before cloning:

```bash
test -w /var/lib/redmine/repos

git clone --mirror git@gitlab-redmine-new-repo:group/subgroup/new-repo.git \
  /var/lib/redmine/repos/new-repo.git

git -C /var/lib/redmine/repos/new-repo.git log -1 --oneline
```

If the write check fails, have the host administrator grant your mirror
account access consistent with the existing repository permissions. Do not
recursively change ownership of existing mirrors.

Use a separate mirror rather than a development working directory. If the
mirror already exists, verify its remote with `git -C
/var/lib/redmine/repos/new-repo.git remote -v` and fetch it instead of cloning
over it. An empty GitLab repository must receive its first commit before a
commit log or source browsing can be verified.

## 4. Verify Redmine can read the mirror

From your deployment checkout:

```bash
cd /path/to/redmine
docker compose exec --user redmine redmine \
  git --git-dir=/var/lib/redmine/repos/new-repo.git log -1 --oneline
```

This should show a commit for a nonempty repository. Resolve any read or path
error before continuing. Redmine needs read access to files and traversal
access to directories; fetching remains the host account's job.

## 5. Register the repository in the intended Redmine project

1. Under **Administration → Settings → Repositories**, ensure Git is enabled.
2. Open the Redmine project that should display this GitLab repository.
3. Under **Settings → Modules**, enable **Repository** if necessary.
4. Open **Settings → Repositories → New repository** and select **Git**.
5. Enter `/var/lib/redmine/repos/new-repo.git` as **Path to repository**
   (sometimes labelled **Path to .git directory**).
6. Set a unique repository identifier within the project, for example
   `new-repo`. Select **Main repository** if this should be the default.
7. Save and open the project's **Repository** tab. Check files, branches,
   commits, and a diff.

Use the local container path in Redmine. Keep the GitLab SSH URL in the
mirror's remote configuration. You can add multiple repositories to the same
Redmine project, or select a different project for each repository.

Project roles need permission to browse repositories and view changesets.
Anyone granted repository browsing access in Redmine can see the mirrored
content through Redmine, independently of their GitLab account permissions.

## 6. Keep this mirror updated

First test a manual fetch as the mirror-owning host account:

```bash
git -C /var/lib/redmine/repos/new-repo.git remote update --prune
```

Open that account's `crontab -e`. Preserve existing schedules and add one new
entry for this mirror, with the following environment if not already present:

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

*/5 * * * * /usr/bin/flock -n /var/lib/redmine/repos/new-repo.fetch.lock /usr/bin/git -C /var/lib/redmine/repos/new-repo.git remote update --prune >> /var/lib/redmine/repos/new-repo.fetch.log 2>&1
```

Check `systemctl status cron` on Ubuntu/Debian and inspect the log after the
first scheduled run:

```bash
tail -n 50 /var/lib/redmine/repos/new-repo.fetch.log
```

Each mirror needs its own schedule, lock, and log. Arrange log rotation.
Scheduled SSH access must succeed without prompts.

Fetching the mirror and importing commits into Redmine are separate steps.
With **Fetch commits automatically** enabled under Redmine's repository
settings, visiting the Repository tab imports changesets. To import explicitly
after a successful fetch, run from the deployment checkout:

```bash
docker compose exec --user redmine redmine \
  bundle exec rake redmine:fetch_changesets RAILS_ENV=production
```

This imports changesets for configured repositories; it does not fetch GitLab
remotes. Initial imports of large histories can take time.

## 7. Verify the complete connection

Push a normal commit to the new GitLab repository, run the host fetch command,
then open the Redmine Repository tab or run the import command. Confirm that
the same commit SHA appears in GitLab and Redmine. For issue linking, configure
referencing keywords in **Administration → Settings → Repositories** and use
an existing issue number, for example `Update documentation, refs #123`.

## Troubleshooting and recovery

| Symptom | Check |
| --- | --- |
| SSH timeout or name resolution failure | DNS, routing, firewall, and the GitLab SSH port from the Redmine host |
| `Permission denied (publickey)` or repository not found | Correct SSH key, clone path, and deploy key enabled on the new GitLab project |
| Manual fetch works but cron fails | Same Linux account, SSH config, trusted host key, noninteractive key access, cron service, and fetch log |
| Container cannot find the repository | Mirror under `/var/lib/redmine/repos`, active bind mount, and exact Redmine path |
| Container reports permission denied | Host mirror permissions and Redmine's ability to traverse parent directories and read files; follow existing mirror access policy |
| Git reports dubious ownership | Trust only the specific mirror path in persistent Git configuration visible to the Redmine user; see the general Git guide |
| Latest commit exists in the mirror but is absent in Redmine | Import changesets and check the selected repository and branch |

See [Git integration](GIT_INTEGRATION.md) for mount setup and additional
troubleshooting. `scripts/backup.sh` does not back up these mirrors, host SSH
keys, SSH configuration, or crontab. Protect host configuration separately.
Mirrors can be recreated from GitLab at their original paths, provided GitLab
still holds the required history. Redmine repository registrations are in its
database backup.

## References

- [GitLab deploy keys](https://docs.gitlab.com/user/project/deploy_keys/)
- [Redmine repositories](https://www.redmine.org/projects/redmine/wiki/RedmineRepositories)
