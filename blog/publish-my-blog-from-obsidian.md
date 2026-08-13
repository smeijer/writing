---
draft: true
description: Use Obsidian, WebDAV, a Synology NAS, Git, and Cloudflare to publish Markdown without leaving your notes.
tags:
  - tooling
  - devops
date: 2026-08-13
---

# How I Publish My Blog from Obsidian

I've moved my blog writing to Obsidian. Not because Obsidian needs to become a CMS, but because my blog already understands Markdown and Obsidian writes Markdown. The missing part was getting a file from one to the other without turning publishing into a checklist.

My articles now live in `writing/blog` inside my Obsidian vault. When I save one, it gets synced to my Synology NAS over WebDAV. A script on the NAS waits for me to stop typing, commits the changes to a separate Git repository, and triggers a Cloudflare build. The website clones that repository during its build and turns the Markdown into articles.

In short, the flow looks like this:

```text
Obsidian
  └─ Remotely Save
       └─ WebDAV
            └─ Synology NAS
                 └─ GitHub writing repository
                      └─ Cloudflare build
                           └─ meijer.works
```

The setup is deliberately boring. There's no CMS API, no custom Obsidian plugin, and no Git client running on every device. Each part does one small thing.

## Syncing Obsidian over WebDAV

I'm using the [Remotely Save] community plugin to sync my vault to a WebDAV endpoint hosted by my NAS. The relevant settings are:

- remote service: WebDAV
- sync direction: bidirectional
- conflict strategy: keep the newer file
- sync after save: 1 second
- periodic sync: disabled
- sync on startup: after 1 second

I don't sync the Obsidian config directory or bookmarks. I do sync names starting with an underscore, because article assets live in `writing/blog/_assets`.

My publishing boundary isn't at the sync layer. Remotely Save syncs the vault, while the NAS only watches the `writing` directory. Notes can move between my devices without every note becoming a potential blog post.

One warning about the plugin config: Remotely Save calls its settings file "obfuscated". That's accurate, but don't mistake obfuscation for encryption. The WebDAV credentials can be decoded. Don't commit `.obsidian/plugins/remotely-save/data.json` to a public repository.

## Watching for changes on the NAS

WebDAV writes the synced files to `/volume1/obsidian/stephan`. The `writing` directory inside it is also a Git checkout of my private writing repository.

I use `inotifywait` to watch that checkout. On Synology, it can be installed through the `inotify-tools` package. The watcher lives at `/volume1/storage/scripts/watch-writing.sh`:

```shell
#!/bin/sh

LOCK='/tmp/obsidian-writing-watcher.lock'

if ! mkdir "$LOCK" 2>/dev/null; then
  echo 'Watcher already running'
  exit 1
fi

cleanup() {
  trap - TERM INT EXIT
  pkill -P $$ 2>/dev/null || true
  rm -rf "$LOCK"
  exit 0
}

trap cleanup TERM INT EXIT

WATCH_DIR='/volume1/obsidian/stephan/writing'
PUBLISH='/volume1/storage/scripts/publish-writing.sh'
LOG='/volume1/storage/scripts/writing-watcher.log'

echo "$(date): watcher started" >> "$LOG"

while true; do
  inotifywait \
    --quiet \
    --recursive \
    --event close_write,create,delete,move \
    --exclude '(^|/)(\.git|\.davfs\.tmp[^/]*)(/|$)' \
    "$WATCH_DIR" >> "$LOG" 2>&1

  echo "$(date): change detected, waiting for quiet period" >> "$LOG"

  while inotifywait \
    --quiet \
    --recursive \
    --timeout 300 \
    --event close_write,create,delete,move \
    --exclude '(^|/)(\.git|\.davfs\.tmp[^/]*)(/|$)' \
    "$WATCH_DIR" >> "$LOG" 2>&1
  do
    echo "$(date): additional change, resetting timer" >> "$LOG"
  done

  echo "$(date): quiet for 5 minutes, publishing" >> "$LOG"

  if "$PUBLISH" >> "$LOG" 2>&1; then
    echo "$(date): publish complete" >> "$LOG"
  else
    echo "$(date): publish failed" >> "$LOG"
  fi
done
```

The first `inotifywait` blocks until something changes. The second one has a five-minute timeout. Every additional change starts that timer again.

That delay matters. Obsidian saves often, and WebDAV can produce multiple filesystem events for what feels like one edit. Without a quiet period, writing a paragraph could result in multiple commits and deployments. Five minutes after my last save, the article is probably ready for the next step.

The lock directory prevents Synology's Task Scheduler from accidentally starting the watcher twice. The `.git` and temporary WebDAV files are excluded to stop the watcher from reacting to its own work.

I start this script with a triggered task in Synology's Task Scheduler. The event is **Boot-up**, and the command is:

```shell
bash /volume1/storage/scripts/watch-writing.sh
```

This makes the automation survive a NAS reboot without needing a service definition of its own. The task runs as my `stephan` user. That's important, because the GitHub key lives in that user's home directory.

## Giving the NAS access to GitHub

The writing repository is private, and I don't want the NAS to use my personal GitHub key. It gets a dedicated deploy key that can only access this one repository.

Create an Ed25519 key on the NAS as the same user that runs the watcher:

```shell
mkdir -p ~/.ssh
chmod 700 ~/.ssh

ssh-keygen \
  -t ed25519 \
  -C 'obsidian-blog@nas' \
  -f ~/.ssh/obsidian-blog \
  -N ''

chmod 600 ~/.ssh/obsidian-blog
chmod 644 ~/.ssh/obsidian-blog.pub
```

The empty passphrase is intentional here. Synology starts the watcher without an interactive terminal, so there is no place to enter one. The trade-off is acceptable because this key is scoped to one repository. It cannot access the rest of my GitHub account.

Print the public half of the key:

```shell
cat ~/.ssh/obsidian-blog.pub
```

In GitHub, navigate to the writing repository, then **Settings → Deploy keys → Add deploy key**. Give it a recognizable title, paste the public key, and enable **Allow write access**. Read-only deploy keys are the default, but this NAS needs to push commits.

Don't copy `~/.ssh/obsidian-blog`. That's the private half and it never leaves the NAS.

### Selecting the key with a host alias

Having a key isn't enough when a machine has multiple GitHub keys. SSH still needs to know which one to use. I added this to `~/.ssh/config` on the NAS:

```sshconfig
Host git-blog
    HostName github.com
    User git
    IdentityFile ~/.ssh/obsidian-blog
    IdentitiesOnly yes
```

`git-blog` is a local alias for `github.com`. `IdentitiesOnly yes` prevents SSH from trying every other key it can find first.

Test the alias before involving Git:

```shell
ssh -T git-blog
```

GitHub responds that authentication succeeded and that it doesn't provide shell access. That's the expected result.

The repository remote must use the alias instead of `github.com`, otherwise this config block won't be selected:

```shell
cd /volume1/obsidian/stephan/writing
git remote set-url origin git@git-blog:smeijer/writing.git
git remote -v
```

Git isn't selecting the deploy key directly. It calls SSH with the host from the remote URL, and `git-blog` makes SSH select `~/.ssh/obsidian-blog`. Other repositories can use their own aliases and keys without interfering with this one.

## Committing and triggering a build

Once the directory has been quiet for five minutes, the watcher calls `publish-writing.sh`:

```shell
#!/bin/sh

set -eu

REPO='/volume1/obsidian/stephan/writing'
DEPLOY_HOOK='<cloudflare-deploy-hook>'

cd "$REPO"
git add --all

if git diff --cached --quiet; then
  exit 0
fi

git commit -m 'chore: update writing'
git push origin main

curl --fail --silent --show-error --request POST "$DEPLOY_HOOK"
```

`git diff --cached --quiet` is a small but important guard. Filesystem events don't guarantee meaningful Git changes. If there is nothing to commit, the script stops and Cloudflare doesn't get a pointless build.

With the deploy key and remote in place, the repository still needs a commit identity. That part is easy to miss:

```shell
git config user.name 'Stephan Meijer'
git config user.email '<email-address>'
```

I missed it too. The first automated commit failed with “Author identity unknown”, which is exactly why the watcher writes everything to a log instead of sending it to `/dev/null`.

The deploy hook is a credential. Keep the real URL out of the writing repository and out of examples like this one. Anyone with that URL can trigger a build.

## Consuming the writing repository

The website doesn't include the articles in its own Git history anymore. Its production build starts by fetching them:

```json
{
  "scripts": {
    "build": "npm run sync:articles && wrangler types && astro check && astro build",
    "sync:articles": "node scripts/sync-articles.js"
  }
}
```

The sync script clones `https://github.com/smeijer/writing.git` into a temporary directory and maps two source paths into the Astro project:

```js
const contentMap = {
  blog: {
    to: "src/content/articles",
    type: "markdown",
    assetBasePath: "/articles/",
  },
  "blog/_assets": {
    to: "public/articles",
    type: "assets",
  },
};
```

Markdown files go to Astro's article collection. Images go to the public article directory, and relative links such as `_assets/example.png` are rewritten to `/articles/example.png` along the way.

The sync also checks that every article has a title, description, and date. I prefer the title as the first `# heading`, so the script moves that into Astro's frontmatter during the build. That lets the source file remain pleasant to read in Obsidian without making the website guess what the title is.

## The publish button is frontmatter

An automated pipeline needs a way to distinguish writing from publishing. Mine is the `draft` property:

```yaml
---
draft: true
description: Explain how I publish from Obsidian.
tags:
  - tooling
date: 2026-08-13
---
```

Drafts still travel through WebDAV and Git. They can even be rendered by the local Astro development server, but production filters them out:

```ts
const posts = await getCollection(
  "articles",
  ({ data }) => import.meta.env.DEV || (!data.draft && data.date < new Date()),
);
```

Removing `draft: true` is my publish button. The next sync creates a Git commit, the NAS calls the deploy hook, and Cloudflare rebuilds the site with the article included. A future date works as a second guard for scheduled articles.

## Final words

This is more plumbing than installing a CMS, but every seam is visible. Obsidian owns writing, WebDAV owns transport, Git owns history, and the website owns rendering. None of them needs to pretend to be the other.

More importantly, writing no longer starts with opening a code repository, creating a branch, or remembering a deployment command. I create a Markdown file in the place where I already keep my notes. Five quiet minutes after the last save, the machinery takes it from there.

[Remotely Save]: https://github.com/remotely-save/remotely-save
