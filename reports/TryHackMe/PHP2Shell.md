# PHP2Shell (TryHackMe)

## Summary

The lab web server ran a PHP build with two engine bugs. Chained together they gave unauthenticated code execution as the web-server user: the first leaks memory to get past ASLR, and the second corrupts memory. The application code on the server wasn't vulnerable. The bugs are in PHP itself.

## Target and version

- Platform: TryHackMe, room `cvephp2shell` (recent-threat room)
- Software: PHP 8.4.26. The target is a Docker container `php2shell-target` on image `php:8.4.26-apache`. The CLI in the same container reports `PHP 8.4.26 (cli) (built: Oct 21 2026)`, Zend Engine v4.4.26, Zend OPcache v8.4.26.
- Fixed PHP versions: unknown (not shown in my run)
- CVSS: TryHackMe's announcement gives 9.2. At least one public write-up says the CVE IDs used in the room are lab identifiers, not NVD entries, so I haven't listed CVE numbers.
- Completed: 2026-10-24

## Preconditions

- Network access to the web server. No account is needed.
- `session.upload_progress.enabled` on, which is PHP's default.

## Environment

TryHackMe AttackBox (`ubuntu@tryhackme-2404`). The vulnerable PHP runs in the `php2shell-target` Docker container, served on `http://127.0.0.1:8088/index.php`, whose page is just `<?php echo "OK\n"; ?>`. Tools: `curl`, `docker`, and a Python exploit (`exploit.py`) in the `php2shell-bypass-aslr-exp` folder.

## How the bug works

PHP has a session upload progress feature. If a multipart POST contains a field named after `session.upload_progress.name` (default `PHP_SESSION_UPLOAD_PROGRESS`), PHP records the upload's progress in the session. This happens before the application's own code runs, and it works on any PHP page, even one that doesn't accept uploads. So an unauthenticated request can put data it controls into server memory.

The room explains how an engine mistake in this path turns that into a memory leak and a memory write. I don't describe the exploit itself here.

## Steps to reproduce

1. Confirm the target and its PHP version:
   ```bash
   docker ps --format '{{.Names}} {{.Image}} {{.Status}}'
   docker exec php2shell-target php -v
   curl -s http://127.0.0.1:8088/index.php
   ```
   Actual: `php2shell-target php:8.4.26-apache Up 6 minutes`; PHP 8.4.26; the page returns `OK`.
2. Room question about the upload progress field. Answer: `PHP_SESSION_UPLOAD_PROGRESS`.
3. Run the exploit against the local target. It leaks heap addresses over HTTP to get past ASLR, then runs a chosen command (here, reading a protected file):
   ```bash
   python3 exploit.py -u http://127.0.0.1:8088/index.php
   ```
   Actual output (trimmed):
   ```
   [*] offsets: zif=0x465370 dtor=0x65eb60 htdata=0x690d8
   [*] stage 1: heap leak (attempt 1/3)
   [+] chunk base = 0x7a5eafa00000
   PROOF: THM{REDACTED}
   ```
   The exploit returned the contents of a file the web page can't serve, so the chosen command ran as the PHP process (the `php:8.4.26-apache` image runs PHP under Apache). The run is not always first-try: some attempts printed `EXPLOIT_FAILED (no proof; the command may still have executed blind)` before one succeeded.

## Impact and limits

Shown in the lab: code execution as the PHP process, without credentials, proven by reading a file the web page doesn't expose.

Not shown: root access. I didn't try privilege escalation. The exploit isn't fully reliable: the heap leak runs in attempts (1/3) and some full runs failed before one worked, so the "actual" output above is one successful run, not every run.

## Detection

- Multipart POSTs containing `PHP_SESSION_UPLOAD_PROGRESS` sent to pages that don't handle uploads, especially many in a row.
- PHP-FPM or Apache error logs showing worker crashes (`SIGSEGV`) at the same time.
- The web-server user (`www-data`, `apache`, `nginx`) starting `sh`, `bash`, `curl`, `wget` or `nc`.

## Fixes

- Upgrade PHP to a fixed release.
- Until then, set `session.upload_progress.enabled = Off` if the site doesn't use upload progress.
- Set `expose_php = Off` so the version isn't advertised.

## Open items

- The fixed PHP versions and the room's difficulty (not shown in my run).

## References

- TryHackMe room: <https://tryhackme.com/room/cvephp2shell>
- PHP manual, session upload progress: <https://www.php.net/manual/en/session.upload-progress.php>
