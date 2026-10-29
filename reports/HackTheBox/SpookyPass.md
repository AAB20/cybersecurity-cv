# SpookyPass (Hack The Box challenge)

## Summary

The `pass` binary compares user input to a password stored in plain text inside the file. `strings` shows the password and `ltrace` confirms the comparison. Entering it prints the flag.

## Target and version

- Platform: Hack The Box, Challenges, Reversing, rated Very Easy
- File: `pass`, from the challenge ZIP
- SHA256 of `pass`: unknown (not recorded)
- Completed: not recorded

## Preconditions

A Linux machine that can run a 64-bit ELF. No network access is needed.

## Environment

Not recorded. Tools: `file`, `strings` (binutils), `ltrace`. Ghidra is optional.

## Steps to reproduce

1. Identify the file:
   ```bash
   file pass
   ```
   Actual: 64-bit x86-64 ELF, PIE, dynamically linked, not stripped.

2. Run it with a wrong password:
   ```bash
   ./pass
   ```
   Actual: a welcome message asks for the password, and a wrong one is rejected.

3. Look for readable strings:
   ```bash
   strings -n 8 pass
   ```
   Expected: the prompts and, if the password is stored in clear, the password near them. Actual: a leetspeak passphrase sits next to the welcome and rejection messages. Value: `REDACTED`.

4. Confirm with `ltrace`:
   ```bash
   ltrace ./pass
   ```
   Actual: `strcmp` is called with my input and the passphrase from step 3.

5. Run `./pass` again with that passphrase. Actual: the flag is printed, `HTB{REDACTED}`.

The flag does not show up in `strings`. In Ghidra, `main` builds it at runtime from an array of integers, and only the password is stored as a plain string.

## Impact and limits

The password check gives no protection: anyone with the file can read the password in a few seconds. This is CWE-798 (hard-coded credentials). Building the flag from integers slows down `strings` only. It doesn't hide the flag from a decompiler or from simply entering the password.

## Fix

Don't keep the secret in the binary. Check it on a server, or at least store a salted hash (bcrypt or Argon2) and compare against that.

## Open items

- SHA256 of `pass` and the `ltrace` output line.
- Completion date.

## References

- Hack The Box Challenges: <https://app.hackthebox.com/challenges>
- CWE-798: <https://cwe.mitre.org/data/definitions/798.html>
