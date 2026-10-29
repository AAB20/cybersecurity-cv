# Ransom letter metadata (TryHackMe)

## Summary

A ransom note came as three files: a Word document, a PDF and a photo. The PDF's metadata names an author and the tool that made it. The photo carries GPS coordinates, a camera model and an editing history across three operating systems. Several of the photo's timestamps contradict each other, so the metadata has been edited and can't be taken as ground truth.

## Target and version

- Platform: TryHackMe. The files were in `~/tryhackme/fronsics`. The exact room name isn't shown in my recording.
- Files (all dated Feb 23 2022):
  - `letter-image.jpg`, 127107 bytes
  - `ransom-letter.doc`, 153088 bytes
  - `ransom-letter.pdf`, 71371 bytes
- Delivered as `ransom-lettter-2-1645608985174.zip`, 308063 bytes.
- Completed: 2026-10-25

## Preconditions

Only the three files. No system access is needed.

## Environment

Kali Linux (`aaa@DESKTOP-DSOU008`). Tools installed during the task: `unzip`, `poppler-utils` (for `pdfinfo`) and `exiftool`.

## Steps to reproduce

1. Unpack the archive:
   ```bash
   unzip ransom-lettter-2-1645608985174.zip
   ```
   Actual: `letter-image.jpg`, `ransom-letter.doc`, `ransom-letter.pdf`.

2. Read the PDF metadata:
   ```bash
   pdfinfo ransom-letter.pdf
   ```
   Actual (trimmed):
   ```
   Title:    Pay NOW
   Subject:  We Have Gato
   Author:   Ann Gree Shepherd
   Creator:  Microsoft® Word 2016
   Producer: Microsoft® Word 2016
   CreationDate: Wed Feb 23 12:10:36 2022 +03
   ModDate:      Wed Feb 23 12:10:36 2022 +03
   Pages: 1   Page size: 595.44 x 842.04 pts (A4)   PDF version: 1.7
   ```

3. Read the image metadata:
   ```bash
   exiftool letter-image.jpg
   ```
   Actual (the useful fields):
   ```
   Camera Model Name : Canon EOS R6
   Lens ID           : Canon EF 50mm f/1.8 STM
   Create Date       : 2022:02:25 13:37:33.42+03:00
   Modify Date       : 2022:02:15 17:23:40+01:00
   Digital Creation Date/Time : 2021:11:05 14:06:13+03:00
   GPS Latitude      : 51 deg 30' 51.90" N
   GPS Longitude     : 0 deg 5' 38.73" W
   GPS Time Stamp    : 13:37:33
   History Software Agent : Adobe Photoshop Lightroom Classic 10.2 (Macintosh),
                            Adobe Photoshop Camera Raw 14.0 (Windows), ... Gimp 2.10 (Linux)
   Image Size        : 1200x800
   ```

4. Pull just the location:
   ```bash
   exiftool letter-image.jpg | grep GPS
   ```
   Actual: latitude ref North, longitude ref West, position `51 deg 30' 51.90" N, 0 deg 5' 38.73" W`.

## What the metadata shows

- The PDF was written in Microsoft Word 2016 and the author field reads `Ann Gree Shepherd`. The title and subject are `Pay NOW` and `We Have Gato`.
- The photo's GPS puts it in London (`51 deg 30' 51.90" N, 0 deg 5' 38.73" W`).
- The photo claims a Canon EOS R6 body with a Canon EF 50mm f/1.8 STM lens.
- The editing history lists three tools on three operating systems: Lightroom Classic on macOS, Photoshop Camera Raw on Windows, and Gimp on Linux.

## Impact and limits

The metadata gives real leads for attribution: an author name, a location and a software trail. None of it is proof, because the fields disagree with each other:

- The Modify Date (2022-02-15) is earlier than the Create Date (2022-02-25), and the Digital Creation date is earlier still (2021-11-05). A straight-from-camera file would not have the modify time before the create time.
- The Canon EOS R6 is a mirrorless RF-mount body, but the lens field names an EF-mount lens, and the file passed through Lightroom, Photoshop and Gimp. The photo has clearly been processed.

So the camera, lens and timestamps are claims written by software, not evidence of how or when the photo was taken. The author name and GPS are the most useful starting points, but both can be edited too.

## Open items

- The room's exact name and its task questions. I was checking the metadata for a flag (`exiftool letter-image.jpg | grep THM`); none was visible in the recording.
- The `ransom-letter.doc` was not opened in the recording; it may hold the same or different author metadata.

## References

- ExifTool: <https://exiftool.org/>
- Poppler (`pdfinfo`): <https://poppler.freedesktop.org/>
