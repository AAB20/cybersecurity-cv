# PhantomRing (Hack The Box Sherlock)

## Summary

`agent`, a Linux ELF found in `/var/tmp`, is a small C2 client. It connects out to a hard-coded IP and port, retries on a fixed sleep, and runs commands sent by the server. The commands cover listing logged-in users, looking for SUID binaries, disabling kernel tracing and eBPF-based security tools, and deleting itself. All of this comes from static analysis. The sample was never run.

## Target and version

- Platform: Hack The Box, Sherlocks (malware analysis), rated Very Easy
- Evidence: password-protected ZIP containing one file, `agent`
- SHA256: unknown (not recorded in this report yet)
- Completed: not recorded

## Preconditions

Only the sample. Analysis was static, in a VM with networking turned off.

## Environment

Not recorded. Tools: `sha256sum`, `file`, `strings`, Ghidra (CodeBrowser and decompiler).

## Steps to reproduce

1. Hash and identify the file:
   ```bash
   sha256sum agent
   file agent
   ```
   Actual: 64-bit ELF, dynamically linked, not stripped. Since it isn't stripped, the function names survive in Ghidra.

2. Pull strings:
   ```bash
   strings -n 6 agent
   ```
   Actual: the C2 IP is in clear text, along with the file paths the handlers use.

3. In Ghidra, open `main` from the Symbol Tree and read the decompiled code.
   - The connect log line has the port as a hex constant. Convert it to decimal.
   - The reconnect loop ends in `sleep(<hex value>)`. That value is the retry delay in seconds.

4. Open the function `main` passes each received message to. It is a chain of `strcmp` checks, one per command. Count the branches, leaving out the invalid-command branch.

5. Decompile each command handler:
   - User listing: note which system file it opens.
   - SUID search: note the directory it opens.
   - Security-tool killer: it reads `/proc/<pid>/maps` for every process and compares against a string to find eBPF tools, then writes to a list of kernel tracing files. Note the string and the first file in the list.
   - Self-destruct: it calls `readlink` on a procfs path to find its own file and then deletes it. Note the path and the command string that triggers it.

6. Check the Symbol Tree for repeated imported calls that belong to one kernel interface. That interface is the answer to the EDR-evasion question.

## Answers

| Question | Answer |
|---|---|
| SHA256 | unknown |
| C2 IP | unknown |
| C2 port | unknown |
| Reconnect delay (seconds) | unknown |
| Number of commands | unknown |
| Kernel interface used to avoid EDR | unknown |
| File read to list logged-in users | unknown |
| Directory scanned for SUID binaries | unknown |
| String looked for in `/proc/<pid>/maps` | unknown |
| First tracing file disabled | unknown |
| procfs path used to find its own file | unknown |
| Command that triggers self-deletion | unknown |

These come from my own Ghidra session and haven't been copied in yet (see Open items).

## Impact and limits

What static analysis shows: the binary can reach a C2 server, give the operator a list of users and SUID binaries, turn off tracing, interfere with eBPF tools, and remove itself.

What it can't show: whether the C2 server was reachable, which commands were actually sent on the compromised server, or how the file got into `/var/tmp`. That needs network logs and host telemetry, and the Sherlock doesn't include them.

## MITRE ATT&CK

- T1071 Application Layer Protocol and T1571 Non-Standard Port (C2)
- T1562.001 Impair Defenses: Disable or Modify Tools (tracing and eBPF tools)
- T1070.004 Indicator Removal: File Deletion (self-destruct)
- T1033 System Owner/User Discovery
- T1083 File and Directory Discovery (SUID search)

## Detection and fixes

- Block the C2 IP and port, and search firewall and NetFlow logs for earlier connections to them.
- Alert on executables running from `/tmp`, `/var/tmp` and `/dev/shm`.
- Alert on writes to `/sys/kernel/tracing/` or `/sys/kernel/debug/tracing/` by anything other than admin tooling.
- The binary deletes itself, so during response copy `/proc/<pid>/exe` and memory before killing the process.

## Open items

- Fill in every "unknown" in the answers table from my Ghidra notes, and add the SHA256 under Target and version.
- Add the kernel-interface detection once the interface is confirmed.
- Completion date.

## References

- Hack The Box Sherlocks: <https://app.hackthebox.com/sherlocks>
- Ghidra: <https://ghidra-sre.org/>
- MITRE T1562.001: <https://attack.mitre.org/techniques/T1562/001/>
