# log-archive

A small CLI that compresses a log directory into a timestamped `tar.gz`, records each run in `archive.log`, and can optionally prune old archives and copy the result to S3.

```
logs_archive_20261119_100648.tar.gz
```

Built for macOS (stock bash 3.2 + bsdtar) and also runs on Linux.

## Install (MacBook Pro)

```bash
mkdir -p ~/projects/log-archive && cd ~/projects/log-archive   # put the files here
chmod +x log-archive
sudo mkdir -p /usr/local/bin
sudo install -m 755 log-archive /usr/local/bin/log-archive
log-archive --help
```

## Usage

```bash
log-archive /var/log                              # archive to ~/log-archives
log-archive -o ~/archives /var/log                # custom output directory
log-archive -r 30 /var/log                        # also delete archives older than 30 days
log-archive --s3 s3://my-bucket/mac-logs/ /var/log  # also upload to S3
```

Each run appends a line to `<output-dir>/archive.log`:

```
2026-11-19 10:06:48 | OK     | source=/var/log archive=/Users/you/log-archives/logs_archive_20261119_100648.tar.gz size=1.2M entries=42
2026-11-19 10:06:50 | UPLOAD | s3://my-bucket/mac-logs/logs_archive_20261119_100648.tar.gz
```

Exit codes: `0` success (including a `WARN` when some files were unreadable or changed mid-read), `1` for bad input, a corrupt archive, or a failed upload.

## Trying it on a Mac

Modern macOS uses unified logging, so `/var/log` has only a few files. To get something meaningful:

```bash
log-archive ~/Library/Logs          # app and diagnostic logs, no sudo needed
sudo log-archive -o /Users/$USER/log-archives /var/log   # system logs (see note on sudo below)
tar -tzf ~/log-archives/logs_archive_*.tar.gz | head     # inspect an archive
```

Some files under `/var/log` are root-only. Either run with `sudo`, or accept the `WARN` line for the files that were skipped. When using `sudo`, pass `-o` explicitly so archives land in your home rather than root's. If macOS blocks access, add your terminal app under **System Settings → Privacy & Security → Full Disk Access**.

## Scheduling

**launchd (recommended on macOS)**: runs daily at 02:00 and catches up after sleep.

```bash
cp com.example.log-archive.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.log-archive.plist
launchctl kickstart -k gui/$(id -u)/com.example.log-archive   # run once now to test
launchctl bootout gui/$(id -u)/com.example.log-archive        # unload
```

**cron** (works, but needs Full Disk Access for `/usr/sbin/cron` on macOS):

```
0 2 * * * /usr/local/bin/log-archive -r 30 /var/log >> /tmp/log-archive.cron.log 2>&1
```

## Sending archives to S3

```bash
brew install awscli
aws configure sso          # or: aws configure
log-archive --s3 s3://my-bucket/mac-logs/ ~/Library/Logs
```

A least-privilege IAM policy for the uploader only needs write access to the prefix:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-bucket/mac-logs/*"
  }]
}
```

Let S3 handle long-term retention with a lifecycle rule on the `mac-logs/` prefix (for example, transition to Glacier after 30 days, expire after 365), so the local `--retention` window can stay short.

## Design decisions

- **Original logs are never deleted.** Archiving and cleanup are separate concerns; deleting live logs risks breaking processes that hold them open. Use `--retention` only for old *archives*.
- **Archives are owner-only (`umask 077`).** Logs often contain IPs, tokens and usernames.
- **Every archive is verified** (`tar -t`) before it is reported as OK.
- **The output directory may not be inside the log directory**, which would make the tool archive its own output.
- **`COPYFILE_DISABLE=1`** keeps macOS tar from adding `._*` metadata files.

## Ideas for next steps

- Email or Slack/SNS notification on `WARN`/`ERROR`
- `.sha256` checksum file next to each archive
- Truncate or rotate source logs after a verified upload
- Rewrite in Python or Go for a single distributable binary

---
Solution for the roadmap.sh project: https://roadmap.sh/projects/log-archive-tool
