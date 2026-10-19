# Capture-date metadata across formats

Research for [#4](https://github.com/bradenhc/photothing/issues/4), part of map [#1](https://github.com/bradenhc/photothing/issues/1).

**Question:** Which capture-date fields exist in the HEIC, JPEG, PNG, MOV and MP4 files that Apple devices produce? For each field: what are its time-zone semantics, its precision, and how often it is missing or wrong? What fallbacks exist, and how reliable are they?

This note covers facts only. It does not choose a technology or library.

Confidence labels:
- **[spec]**: stated by the owning specification or vendor doc.
- **[observed]**: seen in real device files (the ExifTool Apple sample set, analysed below).
- **[community]**: ExifTool forum or other secondary reports. Treat these as leads, not facts.
- **[unverified]**: plausible but not confirmed against a source or a sample. Check it with real files before relying on it.

---

## 1. Summary table

| Container (Apple producer) | Field | Clock | Offset stored? | Precision | Notes |
|---|---|---|---|---|---|
| JPEG / HEIC (Exif) | `DateTimeOriginal` (0x9003) | local wall clock [spec] | No, it lives in a separate tag | 1 s | Capture time. Exif "Freeze 2", so editors should not change it [spec] |
| JPEG / HEIC (Exif) | `OffsetTimeOriginal` (0x9011) | n/a | `±HH:MM` [spec] | n/a | Added in Exif 2.31 (July 2016) [spec]. Written by iPhones from iOS 13 on [observed] |
| JPEG / HEIC (Exif) | `SubSecTimeOriginal` (0x9291) | n/a | n/a | Free-length ASCII digits [spec]. iPhones write 3 digits (ms) [observed] | Written by iPhones from about iOS 7 on [observed] |
| JPEG / HEIC (Exif) | `DateTimeDigitized` (0x9004, ExifTool `CreateDate`) + `OffsetTimeDigitized` + `SubSecTimeDigitized` | local | separate tag | 1 s + subsec | Same value as Original on iPhone captures [observed] |
| JPEG / HEIC (Exif) | `DateTime` (0x0132, ExifTool `ModifyDate`) + `OffsetTime` + `SubSecTime` | local | separate tag | 1 s | Last file change. Exif "Update 0", so it is rewritten on every edit [spec] |
| JPEG / HEIC (Exif GPS) | `GPSDateStamp` + `GPSTimeStamp` | **UTC** [spec] | implicitly UTC | GPSTimeStamp is a rational, so it can be sub-second | Time of the GPS fix, not the shutter. Present only when location is on |
| JPEG / HEIC (XMP) | `xmp:CreateDate`, `photoshop:DateCreated` | ISO 8601 subset [spec] | Optional TZD. If absent, the zone is "unknown" [spec] | Fractional seconds allowed [spec] | iPhones sometimes write these, with or without offset [observed] |
| PNG | `tIME` chunk | UTC "should" be used [spec] | No | 1 s | **Last modification**, explicitly not creation time [spec] |
| PNG | `tEXt`/`iTXt` keyword `Creation Time` | RFC 3339 preferred, but "actual format … undefined" [spec] | Depends on the writer | Depends on the writer | Free text |
| PNG | `eXIf` chunk | Exif semantics as above | as Exif | as Exif | One chunk at most. "Historical value only" if the image was edited [spec] |
| PNG | XMP (`iTXt` `XML:com.adobe.xmp`) | as XMP | as XMP | as XMP | |
| MOV / MP4 | `mvhd`/`tkhd`/`mdhd` creation & modification time | **UTC recommended** [spec]. Historically often local [spec] | No | 1 s (integer seconds since 1904-01-01 00:00) [spec] | ExifTool `CreateDate`, `TrackCreateDate`, `MediaCreateDate` |
| MOV (Apple) | `mdta` key `com.apple.quicktime.creationdate` | local wall clock with offset [community] | Yes in practice, e.g. `2024:02:26 16:20:00+02:00` [community] | 1 s | ExifTool `Keys:CreationDate`. Apple's doc only says "UTF-8 string … date the movie file content was created" and gives the example "4/21/2012" [spec] |
| MOV / MP4 | `udta` `©day` | string | Depends on the writer | | ExifTool `ContentCreateDate`. Apple: "Date the movie content was created" [spec] |

---

## 2. Exif (JPEG, HEIC, and PNG `eXIf`)

Source: CIPA DC-008, *Exif Version 3.0* (CIPA, 2023), §4.6.5.4.5 and §4.6.6.6. Annex H gives the edit rules. The text is the same in Exif 2.31 and 2.32 for these tags.

- **Format.** `DateTimeOriginal`, `DateTimeDigitized` and `DateTime` are 20-byte ASCII strings, `"YYYY:MM:DD HH:MM:SS"`, on a 24-hour clock. If the value is unknown, every character except the colons is blank, or the whole field is blank, "treated as unknown". [spec]
- **Time zone.** The datetime tags carry no zone. Exif 2.31 (July 2016) added `OffsetTime`, `OffsetTimeOriginal` and `OffsetTimeDigitized`: "the offset from UTC (the time difference from Universal Time Coordinated including daylight saving time)", formatted `"±HH:MM"`, 7 bytes. Blank means unknown. Before 2.31, Exif had no standard way to record the offset, so the datetime was in effect a bare local wall-clock time. [spec]
- **Sub-seconds.** `SubSecTime`, `SubSecTimeOriginal` and `SubSecTimeDigitized` are ASCII digits of any length, aligned left and padded with blanks. [spec]
- **Meaning.**
  - `DateTimeOriginal`: "the date and time the picture was taken".
  - `DateTimeDigitized`: when the image was stored as digital data. It equals Original for a camera capture.
  - `DateTime`: "the date and time the file was changed". [spec]
- **Edit rules (Annex H).**
  - `DateTimeOriginal`, `OffsetTimeOriginal` and `SubSecTimeOriginal`, and the Digitized trio, are ranked **Freeze 2**: kept through edits, "to allow correction in case of error".
  - `DateTime`, `OffsetTime` and `SubSecTime` are **Update 0**: "updated with every edit". [spec]
- **GPS.** `GPSDateStamp` is `"YYYY:MM:DD"` and `GPSTimeStamp` is three rationals. Both are "in UTC", taken from the GPS receiver's reference time. [spec] ExifTool describes it as "UTC time of GPS fix". [ExifTool GPS tags]
- **HEIC.** HEIF stores Exif and XMP as metadata items linked to the image item by a `cdsc` reference. [Nokia HEIF technical docs] The Exif payload is the same TIFF structure as in JPEG, so the same tag semantics apply. (Normative source: ISO/IEC 23008-12, which is paywalled and was not read directly.)
- **PNG `eXIf`.** Holds the same Exif TIFF payload. "Only one eXIf chunk is allowed." The PNG spec adds: "If the image has been edited subsequent to creation of the Exif profile, this data might no longer apply … should be considered to be of historical value only." [PNG 3rd ed. §11.3.4.5]

### Apple ImageIO exposure

ImageIO exposes these tags as properties:
- `kCGImagePropertyExifDateTimeOriginal` (iOS 4+).
- `kCGImagePropertyExifSubsecTimeOriginal` (iOS 10+).
- `kCGImagePropertyExifOffsetTimeOriginal` (iOS 13+). [Apple ImageIO docs]

That iOS 13 boundary matches the samples in the next section.

### Observed in real Apple files

Source: the ExifTool sample set `Apple.tar.gz` (66 files, updated 2025-09-09), parsed directly for this note. The 40 iPhone files break down as follows.

- **`OffsetTime*`** appears on every unedited capture from iOS 13.0 on: iPhone 11, 11 Pro, 12 Pro, 13, 13 Pro, 13 Pro Max, 14, 15, 15 Plus, 15 Pro, 15 Pro Max, 16 Pro and SE 2. Values seen include `-07:00`, `+01:00` and `+03:00`. It is absent on every file from iOS 12 or earlier.
  - One iOS 14.2 file (iPhone 12 Pro Max) lacks it, along with any XMP offset. Older-iOS and app-edited files are therefore offset-less.
- **`SubSecTimeOriginal`/`SubSecTimeDigitized`** are always 3 digits (milliseconds) from iOS 7.0 on. iOS 6 and earlier write no sub-seconds.
  - `SubSecTime` (for `DateTime`) appears only on some iOS 17 files.
- **`DateTimeOriginal` == `DateTimeDigitized`** in every sample.
- **`DateTime` differs from `DateTimeOriginal`** in an edited file: iPhone 14 (iOS 16.0.1) has `DateTime 2022:09:23 11:34:07` and `DateTimeOriginal 2022:09:19 15:00:46`. This is the Freeze-vs-Update behaviour in practice.
- **Edited exports keep the original date.** Files whose `Software` is `Photos 2.0`, `Photos 4.0`, `Camera+ 6.1.1` or `ProCam 10.0.3` all keep `DateTimeOriginal` and `SubSecTimeOriginal`.
- **XMP on iPhone files** usually duplicates Exif in `xmp:CreateDate` and `photoshop:DateCreated`, in inconsistent forms:
  - with ms and offset: `2019-09-21T14:43:51.705-07:00` (iOS 13)
  - with ms, no offset: `2018-10-09T15:12:26.283` (iOS 12)
  - no ms, no offset: `2023-10-13T11:46:41` (iOS 17)
  - mixed within one file: iPhone 15 Pro Max has `xmp:CreateDate` without an offset and `photoshop:DateCreated` with `-08:00`.

  The XMP form cannot be relied on to carry the offset, even when Exif does.
- **GPS UTC agrees with local + offset on modern files.**
  - iPhone 15: `DateTimeOriginal 08:11:15`, `-04:00`, GPS `12:11:13` UTC.
  - iPhone 16 Pro: `2024:09:21 20:04:47`, `-07:00`, GPS `2024:09:22 03:04:45`.

  The two differ by a couple of seconds, because the GPS fix time is not the shutter time. Some 2008–2010 files (original iPhone, iPhone 3GS, iPhone 4) have `GPSTimeStamp` equal to the local `DateTimeOriginal`. A single file cannot show whether that means a UTC+0 location or local time written into a UTC field, so treat GPS time on very old iOS files with suspicion.

---

## 3. XMP

Sources: Adobe XMP namespace docs (`xmp`, `photoshop`, data types). ExifTool XMP tag tables.

- **Date type.** Allowed forms are `YYYY`, `YYYY-MM`, `YYYY-MM-DD`, `YYYY-MM-DDThh:mmTZD`, `YYYY-MM-DDThh:mm:ssTZD` and `YYYY-MM-DDThh:mm:ss.sTZD`. "The time zone designator need not be present in XMP. When not present, the time zone is unknown, and an XMP processor should not assume anything about the missing time zone." Partial dates, such as year only, are legal. [spec]
- **`xmp:CreateDate`**: "The date and time the resource was created. For a digital file, this need not match a file-system creation time … Later file transfer, copying, and so on, can make the file-system time arbitrarily different." [spec]
- **`xmp:ModifyDate`**: last modification, and "not necessarily the same as the file's system modification date". `xmp:MetadataDate`: when the metadata last changed. [spec]
- **`photoshop:DateCreated`**: "The date the intellectual content of the document was created, rather than the creation date of the physical representation." [spec]
- **`exif:DateTimeOriginal`/`exif:DateTimeDigitized`** also exist in the XMP `exif` namespace, defined by CIPA DC-010. [ExifTool XMP tags]
- **Metadata Working Group (MWG) mapping, as implemented by ExifTool:**
  - "DateTimeOriginal" (when the photo was taken) is derived from `EXIF:DateTimeOriginal` (with SubSec), `IPTC:DateCreated`+`TimeCreated` and `XMP-photoshop:DateCreated`.
  - "CreateDate" (digitized) is derived from `EXIF:CreateDate`, IPTC digital creation date/time and `XMP-xmp:CreateDate`.
  - "ModifyDate" is derived from `EXIF:ModifyDate` and `XMP-xmp:ModifyDate`. [ExifTool MWG tags]

---

## 4. PNG

Source: W3C *PNG Specification (Third Edition)*, W3C Recommendation, 24 June 2025.

- **`tIME`.** "Gives the time of the last image modification (**not** the time of initial image creation)." Fields are year (2 bytes), month, day, hour, minute and second (0–60). "Universal Time (UTC) should be specified rather than local time." The chunk is "intended for use as an automatically-applied time stamp that is updated whenever the image data are changed." It has no offset field and no sub-seconds. [spec]
- **`Creation Time` text keyword** means "Time of original image creation". The date format "SHOULD be in the RFC 3339 date-time format or in the date format defined in section 5.2.14 of RFC 1123 … The actual format of this field is undefined." [spec]
- **`eXIf`.** See §2. Its data is "of historical value only" unless the decoder knows independently that it is still valid. [spec]
- **XMP** is carried in `iTXt` with keyword `XML:com.adobe.xmp`. [spec]
- Apple ImageIO exposes `kCGImagePropertyPNGCreationTime` ("date and time the image was created") and `kCGImagePropertyPNGModificationTime`. [Apple ImageIO docs]
- **iOS/macOS screenshots** are PNG. [unverified] Which date fields they carry was not confirmed in this research. They are believed to carry an XMP packet with a date and `exif:UserComment = "Screenshot"`, and macOS screenshots have a localised, date-bearing filename such as `Screenshot 2024-05-01 at 10.15.32.png`. Confirm both with sample files.

---

## 5. QuickTime MOV and ISO-BMFF MP4

Sources: Apple *QuickTime File Format* docs (Movie header atom, Calendar date and time values, QuickTime metadata keys, User data atoms). AVFoundation metadata key docs. ExifTool QuickTime tag tables.

- **Header dates (`mvhd`, `tkhd`, `mdhd`).** Each holds a creation time and a modification time.
  - Encoding: a 32-bit integer in "Macintosh date format", "the number of seconds that have passed since midnight January 1, 1904". [spec] (The 64-bit form for version-1 atoms is defined in ISO/IEC 14496-12, which was not read directly.)
  - Time zone: "This value does not specify a time zone. Common practice is to use local time … It is strongly recommended that all calendar date and time values be stored using UTC time." [spec]
  - The `mvhd` page says: "Set the creation and modification date with coordinated universal time (UTC). In prior versions of the QuickTime file format, this was not specified, and these fields were commonly set to local time for the time zone where the movie was created." [spec]
  - Precision is whole seconds, with no offset field.
- **ExifTool on header dates**: "According to the specification, integer-format QuickTime date/time tags should be stored as UTC. Unfortunately, digital cameras often store local time values instead … by default ExifTool does not assume a time zone for these values", unless the `QuickTimeUTC` option is set. [ExifTool QuickTime tags]
  - This ambiguity matters for MP4 and MOV from non-Apple sources. Apple devices write UTC here. [community]
  - A zero value decodes to 1904-01-01. ExifTool writes zero when the tag is "deleted". [ExifTool QuickTime tags]
- **`com.apple.quicktime.creationdate`** (`mdta` keys, ExifTool `Keys:CreationDate`).
  - Apple's spec: "A UTF-8 string (value type 1) … The date the movie file content was created. For example, '4/21/2012'." [spec]
  - The AVFoundation key `quickTimeMetadataKeyCreationDate` / identifier `quickTimeMetadataCreationDate` is described as "the creation date of the movie file content". [Apple AVFoundation docs]
  - In practice, iPhone and iPad MOVs store local capture time **with** a UTC offset here, and community guidance treats it as the most reliable capture time for iPhone video. [community]
  - Header `CreateDate` is UTC. It has been reported as wrong (from 1 s to weeks later than `CreationDate`) particularly on edited files. [community] (ExifTool forum threads msg=80803 and msg=80470 were down when this note was written, so these reports are via search snippets only.)
- **ExifTool writing note**: "the system time zone is added … if no time zone is specified. This is because Apple software may display crazy values if the time zone is missing for some tags." [ExifTool QuickTime tags] It also notes that Apple Photos "has been reported to show a crazy date/time for some MP4 files" carrying a date tag, "perhaps only if it is missing a time zone". [ExifTool QuickTime tags]
- **Other date-bearing items:**
  - `udta` `©day`: "Date the movie content was created" [spec], ExifTool `ContentCreateDate`.
  - ItemList `IDIT` (`DateTimeOriginal`).
  - Keys `creation_time` and `location.date`. [ExifTool QuickTime tags]
- **AVFoundation `AVAsset.creationDate`** returns a metadata item. It is a `Date` "if the asset contains metadata that the framework can convert", otherwise a string, and `nil` "if no creation date metadata exists". [Apple AVFoundation docs]
- **Live Photos**: the HEIC still and the MOV are paired files with independent timestamps. [unverified] No primary source on how their dates align was consulted.

---

## 6. When the capture date is missing or wrong

| Situation | What happens | Confidence |
|---|---|---|
| Older iOS captures (12 and earlier), and pre-2016 cameras generally | No offset tag, so the datetime is a bare local time. A UTC instant can be recovered only by inference, for example from `GPSTimeStamp` | [spec] + [observed] |
| Edited or re-exported in Apple Photos or third-party camera apps | `DateTimeOriginal` and SubSec are kept. `DateTime`/`ModifyDate` and `Software` change. The offset may be dropped, and the XMP copy may lose its offset | [observed] |
| Edited iPhone videos | Header `CreateDate` can drift from `Keys:CreationDate` | [community] |
| Shared through the iOS share sheet | Apple lets users remove location before sharing (Share → Options → turn off Location) [Apple Personal Safety guide]. Whether dates survive each share path (AirDrop, Messages, Mail, iCloud links, "All Photos Data" toggle) was not confirmed | [spec] for location; dates [unverified] |
| Received through third-party messengers or social apps | Widely reported to strip Exif and re-encode, leaving no capture date | [unverified] |
| Screenshots and screen recordings | No camera Exif. Any date comes from XMP or a PNG text chunk, if written at all. See §4 | [unverified] |
| User changed the device clock or time zone, or travelled | The Exif wall-clock time and the offset reflect device settings at capture. GPS time is independent and UTC | [spec] for GPS; rest is inference |
| Exif "unknown" encodings | All blanks, or blanks with colons, means unknown. Zero QuickTime dates decode to 1904 | [spec] |
| Writers that store partial dates | XMP allows `YYYY` or `YYYY-MM` only | [spec] |
| Quantitative missing or wrong rates | **No primary source gives frequencies.** The ticket asked "how often", and this cannot be answered from specs. It needs measurement on a real library | n/a |

---

## 7. Fallbacks

| Fallback | What it gives | Reliability |
|---|---|---|
| `GPSDateStamp`+`GPSTimeStamp` | UTC time of the GPS fix | Good as a UTC anchor on modern iPhones, where the fix is seconds from the shutter [observed]. It can be stale or absent (location off, or stripped on share). Suspect on 2008–2010 iOS [observed]. Comparing it with the local `DateTimeOriginal` gives the offset when no offset tag exists (inference) |
| XMP `photoshop:DateCreated` / `xmp:CreateDate` | Duplicate of the capture time | Same source as Exif on iPhone. The offset is inconsistently present [observed]. A missing offset means "unknown" [spec] |
| PNG `tIME` | UTC last-modified time | Not a creation time by definition [spec] |
| QuickTime header `CreateDate` | UTC on spec-compliant writers | Ambiguous between local and UTC for non-Apple writers [spec][ExifTool]. Reported to drift on edited iPhone video [community] |
| Filename patterns | Some apps embed date and time in the filename | **iPhone camera files (`IMG_####`) carry no date.** [unverified] Other apps (macOS screenshots, some messengers, Android cameras) embed one, but whether the stamp is local time or UTC differs by app and was not verified against vendor docs. Use as a weak hint only |
| Filesystem birth/modification time | Time the file was created or changed on *this* filesystem | Adobe: "file transfer, copying, and so on, can make the file-system time arbitrarily different" [spec]. Download, copy, unzip, sync and restore usually reset or alter it. Weakest signal |
| Photos library (`PHAsset.creationDate`) | "The date and time of the asset's creation" as held by the Photos library [Apple Photos docs] | Only available through Apple's Photos framework, not from the file itself. Users can adjust it in Photos (inference) |

---

## 8. Open questions (not answerable from the sources read)

1. Which date fields iOS and macOS screenshot PNGs and screen-recording MOVs actually carry.
2. Whether iPhone **HEIC** files carry the same field set as the JPEG samples. The ImageIO and Exif semantics say yes, but no HEIC sample was parsed.
3. The exact string format of `com.apple.quicktime.creationdate` written by current iOS. Community reports show `YYYY-MM-DDTHH:MM:SS±HHMM`-style values with an offset. Apple's doc example is "4/21/2012".
4. Which iOS share paths (Messages, AirDrop, Mail, iCloud links, the "All Photos Data" toggle) preserve or strip date metadata.
5. Real-world missing and wrong rates. These need measuring on a representative library.

---

## Sources

- CIPA DC-008-2023, *Exif Version 3.0* (CIPA). <https://www.cipa.jp/std/documents/download_e.html?DC-008-Translation-2023-E> §4.6.5.4.5, §4.6.6.6, §4.6.7.1, Annex H. Revision history covers 2.31 (July 2016).
- W3C, *Portable Network Graphics (PNG) Specification (Third Edition)*, REC 2025-06-24. <https://www.w3.org/TR/2025/REC-png-3-20250624/> §11.3.4.5 eXIf, §11.3.5.1 tIME, predefined keywords.
- Adobe XMP docs: data types <https://developer.adobe.com/xmp/docs/xmp-namespaces/xmp-data-types/>; xmp namespace <https://developer.adobe.com/xmp/docs/xmp-namespaces/xmp/>; photoshop namespace <https://developer.adobe.com/xmp/docs/xmp-namespaces/photoshop/>.
- Apple QuickTime File Format: Movie header atom <https://developer.apple.com/documentation/quicktime-file-format/movie_header_atom>; Calendar date and time values <https://developer.apple.com/documentation/quicktime-file-format/calendar_date_and_time_values>; QuickTime metadata keys <https://developer.apple.com/documentation/quicktime-file-format/quicktime_metadata_keys>; User data atoms <https://developer.apple.com/documentation/quicktime-file-format/user_data_atoms>.
- Apple AVFoundation: `quickTimeMetadataKeyCreationDate` <https://developer.apple.com/documentation/avfoundation/avmetadatakey/quicktimemetadatakeycreationdate>; `quickTimeMetadataCreationDate` <https://developer.apple.com/documentation/avfoundation/avmetadataidentifier/quicktimemetadatacreationdate>; `AVAsset.creationDate` <https://developer.apple.com/documentation/avfoundation/avasset/creationdate>.
- Apple ImageIO: `kCGImagePropertyExifOffsetTimeOriginal`, `kCGImagePropertyExifSubsecTimeOriginal`, `kCGImagePropertyExifDateTimeOriginal`, `kCGImagePropertyPNGCreationTime`, `kCGImagePropertyPNGModificationTime` (developer.apple.com/documentation/imageio/…).
- Apple Photos: `PHAsset.creationDate` <https://developer.apple.com/documentation/photos/phasset/creationdate>.
- Apple Personal Safety guide, "Manage location metadata in Photos" <https://support.apple.com/guide/personal-safety/ips0d7a5df82>.
- ExifTool tag docs: EXIF <https://exiftool.org/TagNames/EXIF.html>; GPS <https://exiftool.org/TagNames/GPS.html>; QuickTime <https://exiftool.org/TagNames/QuickTime.html>; XMP <https://exiftool.org/TagNames/XMP.html>; MWG <https://exiftool.org/TagNames/MWG.html>.
- ExifTool sample images, `Apple.tar.gz` <https://exiftool.org/sample_images.html> (parsed directly for §2 "Observed").
- Nokia HEIF technical overview <https://nokiatech.github.io/heif/technical.html>.
- [community] ExifTool forum threads, seen via search snippets because the forum was offline: <https://exiftool.org/forum/index.php?msg=80803>, <https://exiftool.org/forum/index.php?msg=80470>, <https://exiftool.org/forum/index.php?msg=84877>.
