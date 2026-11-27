# Disk & Server Health Checker v2.0

Checks the hard disks and the server around them ( PERC RAID / SAS on Ubuntu,
and plain SATA/SAS disks on any Linux box). It tests every disk and array at the same time, shows a
live web dashboard, and can post a tamper-evident record of every step to Discord.

The program is encrypted and asks for a password when it starts.

**It never deletes, creates, imports or changes a RAID array or its configuration.** The write test
can overwrite the *data stored on* a disk or array, but only when you allow it (see below).

## 1. The complete unattended run

One command, no questions, no stops. Run it on the server, as your normal user:

```bash
sudo DISCORD_WEBHOOK='https://discord.com/api/webhooks/XXX/YYY' \
  python3 disk_health_check_secure.py --mode full --yes --allow-erase --write-method destructive --all-disks
```

| Part | What it does |
|---|---|
| `--mode full` | Everything: SMART logs, short + long self-test, full read scan, Patrol Read, Consistency Check, write + verify test. |
| `--yes` | Never asks a question; every prompt takes its safe default. |
| `--allow-erase --write-method destructive` | Permission for the write test to **overwrite the data** on each disk/array. Both flags are needed: `--yes` alone never erases anything and falls back to the slower write-then-restore test. |
| `--all-disks` | Also read/write-test individual disks that are not part of a RAID array. On a machine with no RAID this is automatic. |
| `DISCORD_WEBHOOK=...` | Optional. Leave it out and the Discord feed is simply off. |

Always protected, whatever flags you give:

- The RAID configuration is never touched.
- A disk or array that is mounted, holds swap, or holds the running Ubuntu is **not** erased; it gets a
  test file in its free space instead.
- Anything in use by LVM or Linux software RAID is skipped, with a note in the report.

Without `--no-tmux` the tool restarts itself inside a tmux session called `diskcheck`, so the run keeps
going if AnyDesk/RustDesk or the terminal closes. Reattach with `sudo tmux attach -t diskcheck`.

To skip the password prompt on an unattended run, put `DHC_PW='<the password>'` in front of the command,
next to `DISCORD_WEBHOOK`.

## 2. Other ways to run it

```bash
sudo python3 disk_health_check_secure.py                 # interactive: asks the mode and each write-test choice up front
sudo python3 disk_health_check_secure.py --mode quick    # ~10 min, read-only, no load on the disks
sudo python3 disk_health_check_secure.py --mode standard # + long self-test + full read scan (nothing is written)
sudo python3 disk_health_check_secure.py --mode full     # + Patrol Read + Consistency Check + write test (asks how)
sudo python3 disk_health_check_secure.py --mode full --yes   # unattended, keeps all data: write, verify, then restore
sudo python3 disk_health_check_secure.py --monitor       # quick SMART-only recheck against the last run
sudo python3 disk_health_check_secure.py --blink-off     # turn off the bay locate lights
```

Useful options: `--jobs N` (how many disks/arrays at once; default all), `--disks SERIAL,SERIAL`
(on a RAID machine: also test just these individual non-array disks, instead of `--all-disks`), `--include-ssd` (also check SSDs and NVMe drives; they are never write-tested unless you pick a write method yourself), `--skip-long --skip-read --skip-patrol --skip-cc --skip-write`.

**Heat does not stop the run.** Temperatures are watched, shown and graded, but a hot disk keeps testing.
Only a disk that reaches its *own* shut-down (trip) temperature waits, for at most 20 minutes
(`--max-hot-min`); then its test is recorded as "could not run" and the run carries on. `--no-temp-pause`
turns even that off. `--max-temp 55` brings back a fixed pause temperature if you want one.

**A stuck disk does not stop the run.** A test with no progress for 15 minutes raises a warning
(`--stall-warn-min`); after 60 minutes (`--stall-fail-min`) that disk is recorded as FAILING ("stopped
responding"), and the other disks finish normally. The result then says INCOMPLETE.

**When the run ends** the results are saved first, then the live page stays open so you can read it: until
you press Enter in the terminal (at most 12 hours), or 10 minutes when there is no terminal.
`--linger-min N` changes that; `--linger-min 0` exits at once.

**Check one SSD or NVMe drive** (read-only, about 10 minutes):

```bash
sudo python3 disk_health_check_secure.py --mode quick --include-ssd --no-tmux
```

**If the run is interrupted** (Ctrl+C, power cut, reboot): run the *same command* again. Finished tests
are skipped, unfinished ones run again. A run that was stopped early is always labelled INCOMPLETE on the
page and in the report, with the grades marked "so far". A write-then-restore test that was cut off is restored and
verified automatically at the next start, before anything else.

## 3. Live dashboard

It starts by itself. The terminal prints the address, normally `http://127.0.0.1:8787`, and a browser
opens on the server's desktop. If no browser opens, open that address yourself:

```bash
xdg-open "http://127.0.0.1:8787"
```

To watch from a phone or another PC on the same network, add `--bind 0.0.0.0`. The terminal then prints
an address with a secret token in it, and the page shows a QR code for it. Other flags: `--port N`,
`--no-browser`, `--no-dashboard`. The dashboard is read-only: nothing on the page can change the server.

## 4. Discord record

1. In Discord: channel settings > Integrations > Webhooks > New Webhook > Copy Webhook URL.
2. Give it to the tool in one of these ways (first one found wins):
   - `--discord-webhook URL` on the command line (visible to other users of the machine in `ps`),
   - `DISCORD_WEBHOOK=URL` in front of the command, as in section 1,
   - the config file `~/.config/disk-health-check/config.ini` (best; `chmod 600` it):

     ```ini
     [notify]
     discord_webhook = https://discord.com/api/webhooks/<id>/<token>
     heartbeat_min = 15
     ```

The channel gets: a start message (server, every disk, a hash of each disk's starting SMART data), a
message after every finished test, an immediate alert the first time anything is graded FAILING, DEAD
or TAMPERED, a heartbeat every 15 minutes, and at the end the verdict plus the report files. Every
message carries a sequence number and a hash that depends on all earlier messages, so a message cannot
be changed, reordered or removed without the numbers no longer adding up. A dead or slow Discord never
slows a test; missed messages are sent when it comes back.

The webhook URL is never written into a report, a log or the locked build.

## 5. Results and checking the evidence

Everything lands in `~/disk-health-report-<date-time>/`:

| File | What |
|---|---|
| `report.html` | The full report. Rewritten after every finished test, so it is always current. |
| `report.txt`, `results.json` | The same as text and as data. |
| `raw/` | Every raw tool output (smartctl, perccli, ipmitool): the evidence. |
| `events.jsonl` | The hash-chained event log (what was posted to Discord). |
| `manifest.json` | SHA-256 of every evidence file, sealed into the event chain at the end. |
| `run.log` | The terminal output. |

To check later that a report folder was not altered:

```bash
python3 disk_health_check_secure.py --verify-evidence ~/disk-health-report-<date-time>
```

It re-hashes every file, checks the event chain, and prints the final sequence number and chain head.
**Compare those two values with the last message in the Discord channel.** If they match, the folder is
what the run produced. A PASS on its own only shows the folder agrees with itself. This is a published
hash chain, not a digital signature: its strength comes from the channel being outside the control of
whoever is at the machine, so watch the channel during the run or keep it where they cannot delete
messages.

## 6. Grades, and why you can check each one

| Grade | Meaning |
|---|---|
| HEALTHY | Every check that ran came back clean. |
| WATCH | Minor, stable wear. Fine to use with normal backups. |
| WARNING | A real problem. Non-critical data only, or replace. |
| FAILING | The disk is failing: unreadable areas, failed writes, a failed self-test, growing defects. |
| DEAD | The controller dropped it, the drive reports its own failure, or data read back different from what was written. |
| TAMPERED | A counter was reset or faked: the hidden hour log disagrees with SMART, or more data was processed than is physically possible in the hours shown. |

The grade is the worst single finding. The score is 100 minus a penalty per finding (WATCH 5, WARNING 15,
FAILING 40, critical 100). Power-on hours are shown for information and never lower a grade.

Every disk, array, controller and the server gets a **"Why this grade"** block in the report: the score
arithmetic, then every finding with a plain explanation and its evidence (the measured value, the
threshold, the source, and the file in `raw/` it was saved in). A HEALTHY item lists which checks ran and
which did not, so HEALTHY never means "not checked".
