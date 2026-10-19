# What Apple photo dumps look like

Research for [#3](https://github.com/bradenhc/photothing/issues/3), part of map [#1](https://github.com/bradenhc/photothing/issues/1). Researched 2026-10-18.

**Question:** What does a folder "dump" of photos from an Apple device contain across the common export paths? For each path: file naming, formats, companion files, folder structure, and which capture-date metadata survives. Also: what the iPhone "Transfer to Mac or PC" setting does.

**Scope:** Facts about what lands in a folder that could become a **Source**. Nothing here picks a technology for PhotoThing.

**How much to trust each claim.** Every claim is tagged:

- **[A]** Apple or Microsoft first-party documentation, or a published standard. Quoted or closely paraphrased.
- **[O]** Widely observed behaviour that no first-party source documents. It comes from community reports and needs checking against real dumps before the spec depends on it.
- **[?]** Unknown. No reliable source was found.

Apple documents what *formats* each path produces much better than it documents *filenames* or *folder layout*. Most naming facts are therefore [O]. The [O] claims in the summary table should be checked against real dumps.

---

## 1. Background facts that apply to every path

### 1.1 Capture formats on the device

- iPhone 7 and later (and the matching iPads) capture photos in **HEIF** (`.heic`) and video in **HEVC/H.265** when the camera format is "High Efficiency". **[A]** [S1]
- Settings > Camera > Formats > **Most Compatible** makes the camera capture **JPEG** and **H.264** instead. **[A]** [S1][S2]
- **Apple ProRAW** (iPhone 12 Pro and later Pro models, iOS 14.3+) is saved as **DNG**. ProRAW cannot be used with Live Photos, Portrait mode, or video. An edited ProRAW photo is shared as `.jpg`, and the unedited original is shared as `.dng`. **[A]** [S3]
- Since macOS Tahoe 26, a Mac with HDR screenshot capture saves screenshots as HEIF and screen recordings as HEVC. **[A]** [S1] iPhone screenshots are commonly `.png`. **[O]**
- So a dump can contain `.heic`, `.jpg`/`.jpeg`, `.png`, `.dng`, `.mov`, and `.mp4`. The case of the extension varies by path: the camera roll uses upper case (`IMG_1234.HEIC`), while some exports use lower case. **[O]**

### 1.2 DCIM / DCF naming (the device's own file layout)

- The iPhone exposes its camera roll in the camera-industry **DCF** layout: a `DCIM` root, subfolders named with a 3-digit number (100 to 999) plus 5 characters, and files named with 4 characters plus a 4-digit number (0001 to 9999). Files that share the same number form one related "DCF object". **[A]** for the DCF rule itself [S4]. Apple does not document how it applies DCF. **[O]**
- On iPhone this looks like `DCIM/100APPLE/IMG_0001.HEIC`, then `101APPLE`, `102APPLE` and so on. The counter wraps after `IMG_9999`, so **filenames are not unique** over the life of a device, or across devices. **[O]** [S13]
- Videos use the same `IMG_####` prefix with `.MOV`. **[O]**
- Newer iOS versions have also been reported to use date-based DCIM subfolder names such as `202310__`. **[?]** No reliable source was found. Do not depend on the folder name format.

### 1.3 Live Photos

- A Live Photo is **two files**: a still image and a video. Apple says so for Image Capture ("A Live Photo is downloaded as two separate files—a still-image file and a video file") and for Photos' Export Unmodified Original. **[A]** [S6][S7]
- The two files normally share a basename, for example `IMG_1234.HEIC` and `IMG_1234.MOV`. This fits the DCF "same number = related" rule. **[O]** [S4]
- A shared basename is not a guarantee of pairing. Apple's photo framework models the pair as one asset with a `photo` resource and a `pairedVideo` resource [S8]. The video side carries a QuickTime **content identifier** metadata item (`com.apple.quicktime.content.identifier`) [S9]. The still image carries the matching identifier in Apple's maker-note data. **[A]** for the video key; **[O]** for the still image side (it is not in Apple's public docs).
- A device or service that doesn't support Live Photos receives a still photo only. **[A]** [S10]

### 1.4 Edits, `IMG_E` renders and `.AAE` sidecars

- Photos on iOS and macOS edits **non-destructively**. Apple's PhotoKit models an edited asset as separate resources: the original `photo`, a `fullSizePhoto` ("a modified version of the original photo asset"), and `adjustmentData` ("data for use in reconstructing recent edits"). There are matching video resources: `fullSizeVideo`, `fullSizePairedVideo`, and others. **[A]** [S8]
- When those resources are written out as files during a USB transfer, the commonly reported result is: **[O]** [S13][S14]
  - `IMG_1234.HEIC`: the unedited original.
  - `IMG_E1234.HEIC` (or `.JPG`): the edited render. The `E` replaces the `_`, so the name is still 8 characters.
  - `IMG_1234.AAE`: a small XML/plist file describing the edit. Only Apple Photos understands it.
  - An edited Live Photo can also produce `IMG_E1234.MOV`.
- Users report this "duplicate with an E" on Windows imports with Keep Originals, and for Portrait, cropped, and other edited photos (the reports cover iOS 17 and Windows 11). **[O]** [S13][S14]
- `alternatePhoto` ("photo data that isn't the primary form of its asset") covers RAW+JPEG pairs. Such pairs can arrive as two files with one basename. **[A]** for the resource type [S8]. The file layout is **[O]**.

### 1.5 Capture-date metadata

- **Photos (HEIC, JPEG, DNG):** the capture time is the Exif `DateTimeOriginal` tag, written as `YYYY:MM:DD HH:MM:SS` local time with **no time zone**. Exif 2.31 and later add `OffsetTimeOriginal`, which carries the UTC offset. Apple's ImageIO exposes both (`kCGImagePropertyExifDateTimeOriginal`, `kCGImagePropertyExifOffsetTimeOriginal`). **[A]** [S11][S12] iPhones write both tags. **[O]**
- **Videos (MOV):** the QuickTime metadata item `com.apple.quicktime.creationdate` holds the creation date of the content, as an ISO 8601 string with a time zone. The movie header also holds a creation time in UTC. **[A]** for the key [S9]; **[O]** for its format and that iPhones always write it.
- **PNG screenshots:** these may have no Exif capture date. Their date may exist only in XMP or in filesystem timestamps. **[O]**
- **Filesystem timestamps** (file created and modified dates) depend on the copy path. Many transfer tools set them to the copy time, so they are not a reliable capture date. **[O]**
- Apple: "Sharing a photo or video includes sharing its associated metadata, such as date and time, location, device, and captions." **[A]** [S10]

---

## 2. Path by path

### 2.1 Windows: Photos app import or File Explorer copy (USB)

- **Setup:** Apple tells Windows users to install the **Apple Devices** app from the Microsoft Store, connect over USB, unlock the phone, tap Trust or Allow, and then import using the **Microsoft Photos** app. **[A]** [S2] Microsoft gives the same steps: Photos > Import > the connected device, then pick items and a destination. **[A]** [S15]
- **iCloud caveat:** if iCloud Photos is on, "you must download the original, full resolution versions of your photos to your iPhone or iPad before you import to your PC." Microsoft adds that items that exist only in iCloud may not import, and points to iCloud for Windows. **[A]** [S2][S15] A USB dump from a phone with "Optimize iPhone Storage" turned on can therefore be **incomplete**, missing whatever is only in iCloud.
- **Format conversion:** importing HEIF/HEVC "into Photos, Image Capture, or a PC" may convert it to JPEG or H.264 unless **Keep Originals** is set (see §3). **[A]** [S1]
- **Folder structure:** File Explorer shows `Apple iPhone > Internal Storage > DCIM > 1xxAPPLE`, and a copy keeps whatever subfolders the user drags. **[O]** [S13] The Photos app's destination folder and any date subfolders depend on the app version, and Microsoft documents only "select where to save them". **[?]**
- **Naming:** original device names (`IMG_####`, `IMG_E####`, `.AAE`, `.MOV`). **[O]**
- **Metadata:** Exif and QuickTime dates are inside the files, so they survive a byte copy. If Automatic converts to JPEG, the conversion is done on the device. Whether Exif dates survive it is **[O]** (reported yes).
- **Windows viewing:** HEIC and HEVC need Microsoft Store codec extensions to display. This affects viewing only, not what is in the dump. **[O]**

### 2.2 macOS: Image Capture (USB)

- Import To: a folder (or an app), MakePDF, or Build Web Page. Click Download or Download All. There is an optional "Delete after import". By default items remain on the device. **[A]** [S6]
- **Live Photo → two files** (still + video). **[A]** [S6]
- Subject to the same Keep Originals and Automatic conversion as any USB import. **[A]** [S1]
- **Layout:** a flat folder of device filenames. Image Capture does not recreate `DCIM/1xxAPPLE`. **[O]** Whether it brings `IMG_E` renders and `.AAE` files under Keep Originals is **[O]** (reported yes, as on Windows).

### 2.3 AirDrop (iPhone → Mac)

- On a Mac, received photos are either saved to **Downloads** or opened in **Photos**. "Some items are automatically saved in the Downloads folder, but you may also have the option to choose where to save items—for example, whether to save a photo in the Downloads folder or open it in Photos." If both devices use the same Apple Account, the transfer is accepted automatically. **[A]** [S16]
- The sender can set share **Options** first: **[A]** [S10]
  - **Automatic**: "the best file format for the destination". AirDrop may send JPEG or H.264 if the receiver doesn't support HEIF/HEVC. [S1]
  - **Current**: prevents format conversion.
  - **Most Compatible**: "files may convert to JPEG or MOV".
  - **All Photos Data** (AirDrop only): "share the original file with edit history and metadata".
  - Location can be turned off. Keywords can be turned off.
- **Live Photos:** when saved to Downloads rather than Photos, users report that only the still arrives unless All Photos Data is used. **[O]** [S17]
- **All Photos Data** with several items: each item has been reported to arrive as its own subfolder holding the original, the edited render, and an `.AAE`. **[O]** [S17]
- Received files usually keep the `IMG_####` name. Metadata travels with the file unless the sender turned Location off. **[A]** for metadata [S10]; **[O]** for naming.

### 2.4 Photos for Mac: File > Export > Export Unmodified Original

- "Export photos in the original file format in which they were imported." **[A]** [S7]
- **Live Photo → two files** (still + video). **[A]** [S7]
- Optional **Export IPTC as XMP**, which writes a sidecar `.xmp` holding the IPTC metadata and keywords assigned in Photos. **[A]** [S7]
- **File Name:** the photo's Title, its Filename, or **Sequential** with an optional prefix ("Birthday - 1"). **[A]** [S7]
- **Subfolder Format:** None, or grouping into subfolders (for example by Moment or memory, named after the group). **[A]** [S7]
- Edits are **not** applied, because this exports the unmodified original. Whether `.AAE` files are included is **[?]**. Older Photos versions are reported not to export them.
- Embedded Exif and QuickTime dates are in the original files, so they survive. **[O]** (follows from the files being unmodified originals).

### 2.5 Photos for Mac: File > Export > Export N Photos (edited / rendered)

- Photo Kind: **JPEG**, **HEIC**, **TIFF**, or **PNG**, with options for JPEG Quality, 16-bit (TIFF only), Color Profile, and Size. **[A]** [S7]
- **Live Photo → still image only.** **[A]** [S7] ("You can also export a Live Photo as a still image.")
- **Videos:** Movie Quality 480p, 720p, 1080p, or 4K, with an option to include metadata such as title and caption. **[A]** [S7]
- File Name and Subfolder options are the same as §2.4. **[A]** [S7]
- The current Apple page doesn't mention the location-information toggle that older versions had. Whether `DateTimeOriginal` is kept in rendered exports is **[O]** (reported yes).
- Dragging photos from Photos to the Finder also exports them. **[A]** [S7] Whether a drag gives the original or the current edited version is **[?]**.

### 2.6 iCloud.com download

- Downloads are "in the most compatible format for the device you're currently using". The **More Download Options** menu lets the user choose a format: **Unmodified Original** or **Most Compatible**. **[A]** [S5]
- Several items come down as a **`.zip`** archive. **[O]** The current Apple page doesn't say this, although older versions of it did.
- Reports differ on whether Most Compatible is honoured for multi-item downloads. **[O]** / **[?]**
- Live Photo handling, `.AAE` handling, and naming (original `IMG_####` names are reported) are **[O]** / **[?]**.

### 2.7 Related: iCloud Photos sync and a library on disk

iCloud Photos keeps "original, full-resolution photos" in the cloud. A device can hold the full originals or "device-optimized versions". **[A]** [S2] This is why USB dumps from optimized devices can be missing items or contain lower-resolution copies (§2.1). The Mac Photos library on disk is a separate kind of Source and is out of scope for this note.

---

## 3. iPhone "Transfer to Mac or PC": Automatic vs Keep Originals

- **Location:** Settings > Apps > Photos, then scroll to the bottom (labelled TRANSFER TO MAC OR PC). It has two choices: **Automatic** (the default) and **Keep Originals**. **[A]** [S1] (the path is as of the December 2025 article). The section label and the Automatic default are **[O]**.
- **Effect:** "When you import HEIF or HEVC media from an attached iPhone or iPad into Photos, Image Capture, or a PC, the media might be converted to JPEG or H.264. If you don't want it to be converted ... tap Keep Originals." **[A]** [S1]
- **Automatic** checks during the transfer whether the receiving computer can handle HEIF/HEVC, and converts if it can't. **[O]** It applies to USB transfers. AirDrop and Messages are governed by the separate share-sheet Options (§2.3). **[A]** [S1][S10]
- **Consequences for a dump:** **[O]** [S13][S14]
  - With **Automatic**, a Windows dump often contains `.JPG` files where the phone had `.HEIC`, and H.264 `.MOV` where it had HEVC. Edits tend to come across already applied, with no `IMG_E` or `.AAE` files.
  - With **Keep Originals**, the dump has the original `.HEIC` and HEVC `.MOV` files, plus `IMG_E####` renders and `.AAE` sidecars for edited items. This is the main source of the "duplicate photos" reports.
  - So two dumps of the **same** phone can differ in extension, format, and file count depending on this setting.

---

## 4. Summary table

| Path | Typical formats | Naming | Live Photo | Edits | Folders |
|---|---|---|---|---|---|
| Windows Photos / Explorer (USB) | HEIC/MOV (Keep Originals) or JPG/MOV-H.264 (Automatic); PNG, DNG pass through [A/O] | `IMG_####` device names [O] | still + MOV, shared basename [O] | Keep Originals: `IMG_E####` + `.AAE` [O]; Automatic: rendered [O] | Explorer: `DCIM/1xxAPPLE` as copied [O]; Photos app: [?] |
| Image Capture (USB) | Same conversion rules as Windows [A] | Device names [O] | Two files [A] | As Windows [O] | Flat in chosen folder [O] |
| AirDrop → Mac | Automatic / Current / Most Compatible chosen by sender [A] | Device names [O] | Still only unless All Photos Data or opened in Photos [O] | All Photos Data: original + edit history [A]; layout [O] | Downloads [A]; per-item subfolders with All Photos Data [O] |
| Photos: Export Unmodified Original | As imported [A] | Title / Filename / Sequential [A] | Two files [A] | Not applied; `.AAE` [?] | None or grouped subfolders [A]; optional `.xmp` [A] |
| Photos: Export (edited) | JPEG / HEIC / TIFF / PNG; video 480p to 4K [A] | Title / Filename / Sequential [A] | Still only [A] | Applied (rendered) [A] | None or grouped subfolders [A] |
| iCloud.com download | Most Compatible (default) or Unmodified Original [A] | Original names [O] | [?] | [?] | `.zip` for multiple items [O] |

## 5. Facts most relevant to the spec

1. **Filenames collide.** `IMG_####` wraps at 9999 and repeats across devices, so a filename alone cannot identify a photo. **[O]**/[S4]
2. **One logical photo can be several files:** still + MOV (Live Photo), original + `IMG_E` render + `.AAE` (edited), RAW + JPEG, and an optional `.xmp`. **[A]** for the model [S8]; **[O]** for the filenames.
3. **The same phone gives different dumps** depending on Keep Originals vs Automatic, share Options, and iCloud optimized storage. **[A]** [S1][S2][S10]
4. **The capture date lives inside the file** (Exif `DateTimeOriginal` with an optional `OffsetTimeOriginal`; QuickTime `creationdate`). Filesystem dates and folder names are unreliable. PNG screenshots may have no Exif date. **[A]**/[O]
5. **Some paths drop data on purpose:** Export (edited) drops the Live Photo motion [A]; share Options can drop location [A]; Automatic and Most Compatible convert HEIC to JPEG [A].

## 6. Open items: check against real dumps

- Actual `IMG_E`, `.AAE`, and Live Photo filenames in each path on current iOS and Windows versions.
- Whether `.AAE` files appear in Export Unmodified Original and in iCloud.com downloads.
- How iCloud.com handles Live Photos, and how its zip is laid out.
- The Windows Photos app's default destination and subfolder behaviour.
- Whether date-based DCIM folder names (`YYYYMM__`) exist on current iOS.
- How Exif dates and offsets are handled when Automatic converts to JPEG.

---

## Sources

- [S1] Apple Support, "Using HEIF or HEVC media on Apple devices" (published 2025-12-05). https://support.apple.com/en-us/116944
- [S2] Apple Support, "Transfer photos and videos from your iPhone or iPad to your Mac or PC" (published 2026-09-14). https://support.apple.com/en-us/120267
- [S3] Apple Support, "About Apple ProRAW". https://support.apple.com/en-us/119916
- [S4] Design rule for Camera File system (DCF), JEITA CP-3461. Summary: https://en.wikipedia.org/wiki/Design_rule_for_Camera_File_system
- [S5] Apple iCloud User Guide, "Upload and download photos on iCloud.com". https://support.apple.com/guide/icloud/upload-and-download-photos-mmb5dbaf376f/icloud
- [S6] Apple Image Capture User Guide, "Import images from a device". https://support.apple.com/guide/image-capture/imgcp1004/mac
- [S7] Apple Photos User Guide (Mac), "Export photos, videos, slideshows, and memories". https://support.apple.com/guide/photos/pht6e157c5f/mac
- [S8] Apple Developer, `PHAssetResourceType`. https://developer.apple.com/documentation/photos/phassetresourcetype
- [S9] Apple Developer, `AVMetadataIdentifier.quickTimeMetadataContentIdentifier` and `.quickTimeMetadataCreationDate`. https://developer.apple.com/documentation/avfoundation/avmetadataidentifier/quicktimemetadatacontentidentifier , https://developer.apple.com/documentation/avfoundation/avmetadataidentifier/quicktimemetadatacreationdate
- [S10] Apple iPhone User Guide, "Share photos and videos on iPhone". https://support.apple.com/guide/iphone/iphf28f17237/ios
- [S11] Apple Developer, `kCGImagePropertyExifDateTimeOriginal` and `kCGImagePropertyExifOffsetTimeOriginal`. https://developer.apple.com/documentation/imageio/kcgimagepropertyexifdatetimeoriginal
- [S12] CIPA DC-008, Exif 2.3x standard (DateTimeOriginal, OffsetTimeOriginal). https://www.cipa.jp/e/std/std-sec.html
- [S13] Apple Community and Microsoft Q&A threads on `IMG_E` duplicates, `.AAE` files, and DCIM folders on Windows (low trust, observational): https://communities.apple.com/de/thread/252786513 , https://discussions.apple.com/thread/255510437 , https://spy.macrumors.com/threads/too-many-picture-folders-under-dcim.1961212/
- [S14] Microsoft Q&A on `.AAE` files (low trust): https://learn.microsoft.com/en-us/answers/a/4534467
- [S15] Microsoft Support, "Import photos and videos" and "Transfer photos from an iPhone or Android phone to your PC". https://support.microsoft.com/help/4027134 , https://support.microsoft.com/help/4558756
- [S16] Apple macOS User Guide, "Use AirDrop to send items to nearby Apple devices". https://support.apple.com/guide/mac-help/mh35868/mac
- [S17] Apple Community threads on AirDrop to Mac with All Photos Data (low trust): https://discussions.apple.com/thread/254895632 , https://discussions.apple.com/thread/254015591
