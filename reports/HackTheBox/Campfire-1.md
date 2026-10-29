# Campfire-1 (Hack The Box Sherlock)

## Summary

On 2024-05-21 the user account on workstation `172.17.79.129` ran `PowerView.ps1` and then `Rubeus.exe` from the Downloads folder. One second after Rubeus ran, the domain controller logged an RC4 service ticket request for `MSSQLService`. That pattern is Kerberoasting.

## Target and version

- Platform: Hack The Box, Sherlocks (DFIR), rated Very Easy
- Evidence from the domain controller: `SECURITY-DC.evtx`
- Evidence from the workstation: `Powershell-Operational.evtx` and the Prefetch folder
- Completed: not recorded

## Preconditions

Only the supplied logs and Prefetch files. Security auditing for Kerberos service tickets (event 4769) and PowerShell Script Block Logging (event 4104) had to be on at the time, and they were.

## Environment

Not recorded. Tools: EvtxECmd and Timeline Explorer for the event logs, PECmd for Prefetch (all Eric Zimmerman tools). Windows Event Viewer works for the logs too.

## Background

Any domain user can ask the DC for a service ticket for an account that has an SPN. Part of the ticket is encrypted with that account's password hash, so it can be cracked offline. On the DC each request is event 4769. Requests that ask for RC4 (`TicketEncryptionType 0x17`) for a user service account, not a computer account ending in `$` and not `krbtgt`, are the usual sign.

## Steps to reproduce

1. Parse the DC log and filter for RC4 service ticket requests:
   ```powershell
   EvtxECmd.exe -f SECURITY-DC.evtx --csv out --csvf dc.csv
   ```
   In Timeline Explorer: `Event Id = 4769`, `TicketEncryptionType = 0x17`, then hide service names ending in `$` and `krbtgt`.
   Expected: one or a few requests from a workstation. Actual: one event at `2024-05-21 03:18:09` UTC, service `MSSQLService`, client address `172.17.79.129`.

2. Parse the workstation PowerShell log:
   ```powershell
   EvtxECmd.exe -f Powershell-Operational.evtx --csv out --csvf ps.csv
   ```
   Filter `Event Id = 4104`. Actual: script blocks from `PowerView.ps1`, first seen at `2024-05-21 03:16:32` UTC.

3. Parse Prefetch:
   ```powershell
   PECmd.exe -d .\Prefetch --csv out --csvf prefetch.csv
   ```
   Actual: `RUBEUS.EXE`, loaded from `C:\Users\Alonzo.spire\Downloads\Rubeus.exe`, last run `2024-05-21 03:18:08` UTC.

## Answers

| Question | Answer |
|---|---|
| When the Kerberoasting happened (UTC) | `2024-05-21 03:18:09` |
| Service targeted | `MSSQLService` |
| Workstation IP | `172.17.79.129` |
| Enumeration script | `PowerView.ps1` |
| First run of that script (UTC) | `2024-05-21 03:16:32` |
| Kerberoasting tool | `Rubeus.exe` |
| Full path of the tool | `C:\Users\Alonzo.spire\Downloads\Rubeus.exe` |
| Tool run time (UTC) | `2024-05-21 03:18:08` |

## Timeline (UTC)

- 03:16:32, workstation: PowerView.ps1 runs (4104)
- 03:18:08, workstation: Rubeus.exe runs from Downloads (Prefetch)
- 03:18:09, DC: RC4 service ticket requested for `MSSQLService` (4769)

## Impact and limits

What the logs show: a ticket for `MSSQLService` was issued with RC4 to Alonzo's workstation right after Rubeus ran there.

What they don't show: whether the hash was cracked or the `MSSQLService` account was used afterwards. Cracking happens offline and leaves nothing in these logs. They also don't show how PowerView and Rubeus got onto the machine. Because the account password may now be known to the attacker, I'd treat it as compromised.

## MITRE ATT&CK

- T1087.002 Account Discovery: Domain Account (PowerView)
- T1059.001 PowerShell
- T1558.003 Steal or Forge Kerberos Tickets: Kerberoasting

## Detection and fixes

A rule that would have flagged the DC event:

```yaml
title: RC4 service ticket requested for a non-computer account
logsource:
  product: windows
  service: security
detection:
  selection:
    EventID: 4769
    TicketEncryptionType: '0x17'
  filter_computer:
    ServiceName|endswith: '$'
  filter_krbtgt:
    ServiceName: 'krbtgt'
  condition: selection and not filter_computer and not filter_krbtgt
level: medium
```

- Reset the `MSSQLService` password to a long random value, or move the service to a group Managed Service Account.
- Allow only AES for Kerberos on service accounts, so RC4 tickets can't be requested.
- Script Block Logging is what caught PowerView here. Keep it on.
- Isolate the workstation, reset Alonzo's credentials, and find out how the tools reached the Downloads folder.

## Open items

- Paste the actual 4769, 4104 and PECmd output rows for steps 1 to 3.
- Completion date.

## References

- Hack The Box Sherlocks: <https://app.hackthebox.com/sherlocks>
- Microsoft, event 4769: <https://learn.microsoft.com/windows/security/threat-protection/auditing/event-4769>
- MITRE T1558.003: <https://attack.mitre.org/techniques/T1558/003/>
- Eric Zimmerman tools: <https://ericzimmerman.github.io/>
