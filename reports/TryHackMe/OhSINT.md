# OhSINT (TryHackMe)

## Summary

Starting from one image, `WindowsXP.jpg`, I found the owner's username, city, home Wi-Fi name, email address, holiday destination and a password, all from public sources.

## Target and version

- Platform: TryHackMe, room `ohsint` (OSINT, Easy)
- Starting file: `WindowsXP.jpg`
- The person and accounts are fictional and were set up for the room.
- Completed: not recorded

## Preconditions

The image and a web browser. No access to any system is needed.

## Environment

Not recorded. Tools: `exiftool`, a web browser, WiGLE (free account needed for BSSID search).

## Steps to reproduce

1. Read the image metadata:
   ```bash
   exiftool WindowsXP.jpg
   ```
   Actual: the `Copyright` field is `OWoodflint`. The file also has GPS coordinates.

2. Search the web for `OWoodflint`. Three accounts come up: an X (Twitter) profile, a GitHub account and a WordPress blog.

3. X profile: the avatar is a cat. One tweet gives the BSSID of a nearby access point.

4. Search that BSSID on WiGLE (Advanced search, BSSID field). Actual: the access point is in London, and WiGLE shows its SSID.

5. GitHub, repository `people_finder`: the README gives the email address and says the person is from London.

6. WordPress blog: a post says the person is in New York. Select all text on the page (or view the source): a line written in white on a white background holds the password.

## Answers

| Question | Answer |
|---|---|
| What is the user's avatar? | cat |
| What city is the person in? | London |
| SSID of the access point | UnileverWiFi |
| Personal email address | OWoodflint@gmail.com |
| Site where the email was found | GitHub |
| Holiday destination | New York |
| Password | `REDACTED` |

## Impact and limits

One photo led to a name, home city, Wi-Fi network, email, travel dates and a password. For a real person, that's enough for targeted phishing, password guessing against the email account, or knowing when their home is empty.

Limits: everything here is in the room's fictional accounts, and I didn't check whether the password works anywhere. The GPS data in the image isn't needed for any answer, so I didn't follow it up.

## Fixes for a real person

- Remove metadata before posting images: `exiftool -all= image.jpg`.
- Use different usernames on different sites if they shouldn't be linked.
- Don't post router details or "away from home" updates while away.
- Hidden text is still in the page source, so never put a password on a web page.

## Open items

- Completion date.

## References

- TryHackMe room: <https://tryhackme.com/room/ohsint>
- ExifTool: <https://exiftool.org/>
- WiGLE: <https://wigle.net/>
