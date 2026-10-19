# Identifying the same photo across imports

Research for [#6](https://github.com/bradenhc/photothing/issues/6) (part of map [#1](https://github.com/bradenhc/photothing/issues/1)).

**Question:** What approaches exist for deciding that a photo or video from a **Source** is "the same" as one already in a **Workspace**, and where does each fail?

**Scope:** This covers facts and failure modes only. It does not choose a library and does not set PhotoThing's policy, which the duplicates ticket decides.

Each claim cites the source it comes from. Where a claim is my own inference and not something a source states, it is marked _(inference)_.

---

## 1. The approaches

There are three families of approach, plus combinations of them.

| Approach | What it compares | Answers the question |
|---|---|---|
| Exact content hash | Every byte of the file | "Is this the same file?" |
| Metadata identity | Embedded identifiers, or a key built from capture fields | "Did the camera record this as the same capture?" |
| Perceptual similarity | A compact fingerprint of the decoded pixels | "Does this look like the same picture?" |

These answer different questions. No one of them answers "is this the same photo" in every case covered below.

---

## 2. Exact content hash

**How it works:** Hash the whole file with a cryptographic hash. Equal hashes mean byte-identical files.

**What it catches:** Re-imports of the same file, including renamed files and files in different folders, because the filename and filesystem metadata are not part of the file's bytes.

**Where it fails:**

- **Automatic HEIC-to-JPEG conversion.** Apple says media transferred to a Mac or PC "might be converted to JPEG or H.264" unless the user turns on Settings > Photos > Keep Originals. Media shared by AirDrop, Messages or email "might automatically be shared in a more compatible format". Sharing through iCloud Photos keeps the original format. ([Apple: Using HEIF or HEVC media on Apple devices](https://support.apple.com/en-us/116944)) A converted copy of a shot shares no bytes with its HEIC original.
- **Edited versus original.** PhotoKit models an edited asset as separate resources. `.photo` "provides the original photo data", `.fullSizePhoto` "provides a modified version of the original photo asset", and `.adjustmentData` holds the data used "in reconstructing recent edits". ([PHAssetResourceType.photo](https://developer.apple.com/documentation/photos/phassetresourcetype/photo), [.fullSizePhoto](https://developer.apple.com/documentation/photos/phassetresourcetype/fullsizephoto), [.adjustmentData](https://developer.apple.com/documentation/photos/phassetresourcetype/adjustmentdata)) A Source may hold the original, the rendered edit, or both, and those files hash differently.
- **Metadata-only changes.** Any tool that writes metadata into the file, such as keywords, GPS removal or a date fix, changes the bytes and so changes the hash, even though the pixels are unchanged _(inference: this follows from hashing the whole file)_. Hashing only the decoded pixel data avoids this, but then any re-encode still breaks the match.
- **Live Photos.** A Live Photo is two files, a still and a paired video (`.pairedVideo` "provides the original video data component of a Live Photo asset", [Apple](https://developer.apple.com/documentation/photos/phassetresourcetype/pairedvideo)). Hashing each file on its own says nothing about the pairing.

**False positives:** Effectively none. Equal hashes mean identical bytes.

---

## 3. Metadata identity

### 3a. Apple-specific identifiers

| Identifier | Where it lives | What it identifies | Source |
|---|---|---|---|
| **ContentIdentifier** | Apple MakerNote tag `0x0011` in the still image (HEIC/JPEG) | Links the still and video of a Live Photo. Called MediaGroupUUID when it appears as an extended attribute (XAttr) | [ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html) |
| **ContentIdentifier** | QuickTime Keys `com.apple.quicktime.content.identifier` in the MOV | The video side of the same Live Photo link | [ExifTool QuickTime tags](https://exiftool.org/TagNames/QuickTime.html), [AVMetadataIdentifier.quickTimeMetadataContentIdentifier](https://developer.apple.com/documentation/avfoundation/avmetadataidentifier/quicktimemetadatacontentidentifier) ("represents the content identifier in QuickTime") |
| **BurstUUID** | Apple MakerNote `0x000b` | Shared by "all images in a burst" | [ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html) |
| **ImageUniqueID** | Apple MakerNote `0x0015` | Not described in ExifTool's docs | [ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html) |
| **PhotoIdentifier** | Apple MakerNote `0x002b` | Not described in ExifTool's docs | [ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html) |
| **PHAsset.burstIdentifier** | PhotoKit API (Photos library only) | "The unique identifier shared by photo assets from the same burst sequence" | [Apple](https://developer.apple.com/documentation/photos/phasset/burstidentifier) |
| **PHAsset.representsBurst** | PhotoKit API | Whether the asset is the representative photo of a burst | [Apple](https://developer.apple.com/documentation/photos/phasset/representsburst) |
| **PHObject.localIdentifier** | PhotoKit API | "A unique string that persistently identifies the object" | [Apple](https://developer.apple.com/documentation/photos/phobject/localidentifier) |
| **PHCloudIdentifier** | PhotoKit API | Identifies an asset across devices synced through iCloud Photos | [Apple](https://developer.apple.com/documentation/photos/phcloudidentifier) |

What the sources establish:

- **MakerNote tags are reverse-engineered.** Apple does not publish a MakerNote specification. ExifTool's names come from community decoding: one tag is marked tentative (`ImageCaptureRequestID?`), and several have no description. The tag `0x0011` was identified on the ExifTool forum as a "media group UUID" that is filled in for Live Photos ([ExifTool forum, "Decoded another Apple MakerNotes field"](https://exiftool.org/forum/index.php?msg=73175)). Apple's only first-party confirmation is the AVFoundation QuickTime content-identifier key.
- **ContentIdentifier is a pairing key, not a photo ID.** Based on the sources above, it groups the still and video of one Live Photo. They do not show it on ordinary photos that are not Live Photos.
- **localIdentifier is per device.** Apple says a local identifier "is valid for referring to objects only in the context of a local device". A cloud identifier "is universal" across iCloud-synced devices, but mapping between the two "can be an expensive operation" and can fail ([PHCloudIdentifier](https://developer.apple.com/documentation/photos/phcloudidentifier)). Both are PhotoKit API values, not values stored in exported files. A folder of exported files does not carry them.
- **The standard EXIF ImageUniqueID** (tag `0xa420`) also exists. ExifTool lists it with no notes ([ExifTool EXIF tags](https://exiftool.org/TagNames/EXIF.html)). These sources do not establish whether a given camera fills it in or keeps it stable.

**Where Apple identifiers fail:**

- **Conversion and sharing.** These sources do not document whether Apple's HEIC-to-JPEG conversion keeps the Apple MakerNote. ExifTool's author has said that the ContentIdentifier is stored deep in Apple-specific metadata, so writing it means copying something like the whole MakerNote block ([ExifTool forum](https://exiftool.org/forum/index.php?msg=84030)). Any tool that rebuilds EXIF without copying the MakerNote will drop it _(inference)_.
- **Stripped metadata.** Privacy-oriented sharing and many web services remove EXIF. The identifiers go with it.
- **Non-Apple sources and screenshots.** These identifiers are Apple camera metadata. Photos from other cameras, and images saved from elsewhere, will not have them _(inference)_.
- **Bursts.** BurstUUID and `burstIdentifier` are shared by every frame in a burst, so they mark membership in a group, not the identity of a single photo. Matching on them alone would merge distinct frames.

### 3b. Composite capture keys

**How it works:** Build a key from capture fields such as DateTimeOriginal plus SubSecTimeOriginal ("fractional seconds"), OffsetTimeOriginal ("time zone"), and pixel dimensions or camera model ([ExifTool EXIF tags](https://exiftool.org/TagNames/EXIF.html)).

**An example from an established tool:** Adobe Lightroom Classic's "Don't Import Suspected Duplicates" treats a photo as a duplicate when it has the same original filename, the same EXIF capture date and time, and the same file size as one already in the catalog ([Adobe: Photo and video import options](https://helpx.adobe.com/lightroom/help/photo-video-import-options.html)). If any of the three differs, the photo is imported again. A converted copy has a different size and often a different extension, so it is not flagged.

**Where composite keys fail:**

- **Bursts and fast shooting.** Bursts produce several frames within the same second. Without sub-second time they collide. Even with it, a key built only from capture time cannot tell frames apart if the sub-second field is missing _(inference)_.
- **Re-encoded copies.** File size and extension change, and pixel dimensions may change if the copy was downscaled for sharing.
- **Clock and time-zone edits.** Changing the date in Photos, or a "shift dates" fix in another tool, changes the key. A missing OffsetTimeOriginal makes local times ambiguous across time zones.
- **Stripped metadata.** Without capture fields, the only fallback is filesystem timestamps, which copying and syncing rewrite.
- **Filename as a key component.** Renaming breaks it. iPhone `IMG_####` counters also wrap around and repeat across devices, so a filename is weak evidence _(inference)_.

---

## 4. Perceptual similarity

**How it works:** Decode the pixels, reduce them to a small fingerprint, and compare fingerprints by distance.

| Method | How it is built | Robust to | Not robust to | Source |
|---|---|---|---|---|
| aHash | 8×8 grayscale; each bit is pixel ≥ mean | Scaling, aspect ratio, brightness and contrast | Gamma and histogram changes, added text or spliced elements | [Krawetz, "Looks Like It"](https://www.hackerfactor.com/blog/index.php?/archives/432-Looks-Like-It.html) |
| pHash (DCT) | 32×32 grayscale, DCT, low-frequency 8×8, each bit is coefficient ≥ mean | Scaling, gamma and color-histogram changes, "minor modifications (minor being less than 25% of the picture)" | Not stated for crops or rotation | Krawetz, same |
| PDQ (Meta) | DCT-based 256-bit hash with a quality score | Designed to match "images which people perceive as similar" | Not rotation-invariant (see below) | [PDQ README](https://github.com/facebook/ThreatExchange/blob/main/pdq/README.md) |
| digiKam fingerprint | Haar wavelet signature, after Jacobs et al., "Fast Multiresolution Image Querying" | Adjustable "Similarity Range" in percent | — | [digiKam manual: Similarity View](https://docs.digikam.org/en/left_sidebar/similarity_view.html), [Find Duplicates](https://docs.digikam.org/en/maintenance_tools/maintenance_duplicates.html) |

**Thresholds published by these sources:**

- Krawetz, for 64-bit hashes: a Hamming distance of 0 means very similar, about 5 means probably similar, and 10 or more means probably a different picture.
- PDQ's recommended starting point: a distance of 31 or less (of 256 bits) is a match, and hashes with a quality of 49 or less should be discarded.
- Both sources present these as starting points to tune, not as guarantees.

**Where perceptual similarity fails:**

- **Rotation and orientation.** PDQ "does not guarantee exact rotational invariance". It can compute eight dihedral hashes (rotations and flips), but small bit differences between them make picking a canonical one unreliable ([PDQ README](https://github.com/facebook/ThreatExchange/blob/main/pdq/README.md)). EXIF Orientation (values 1 to 8, [ExifTool](https://exiftool.org/TagNames/EXIF.html)) means one file may store rotated pixels while another stores a rotation flag. Fingerprinting without first applying orientation compares differently rotated images _(inference)_.
- **Bursts and near-identical frames.** Perceptual hashes are built to match images that look alike, so consecutive frames of a burst, or several shots of the same scene, can fall under a match threshold even though they are different photos _(inference: follows from the design goal)_.
- **Edits.** Small exposure or color edits usually still match. Crops, heavy filters and added text may not (Krawetz). Whether an edit counts as "the same photo" is a policy question that the hash cannot decide.
- **Featureless images.** PDQ adds a quality score so that "junky/featureless images" can be filtered out. Hashes of near-uniform images, such as dark frames or skies, are unreliable.
- **Adversarial collisions.** Struppek et al. showed that small, deliberate image changes can force or prevent collisions in Apple's NeuralHash deep perceptual hash ([FAccT 2022, arXiv:2111.06628](https://arxiv.org/abs/2111.06628)). This matters little for personal photos, but it shows that perceptual matches are evidence, not proof.
- **Cost.** Every image must be decoded, including HEIC. Comparing every pair grows quadratically. digiKam warns that Find Duplicates "will take a long time since it has to compare every image with every other image" and that fingerprinting a large collection "may take hours".
- **Videos.** None of the image hashes above cover video. Video needs its own approach.

---

## 5. Established photo managers

| Product | Documented criteria | What it does with matches | Source |
|---|---|---|---|
| Apple Photos (Mac, iOS 16 and later) | **Not documented.** Apple only says Photos "identifies duplicate photos and videos" | "Merge" keeps "one original photo or video" where the duplicates were. The others go to Recently Deleted for 30 days | [iPhone guide](https://support.apple.com/guide/iphone/merge-duplicate-photos-and-videos-iph1978d9c23/ios), [Mac guide](https://support.apple.com/guide/photos/pht5a3157c1d/mac) |
| Lightroom Classic | Original filename, EXIF capture date and time, and file size must all match | Skips the photo on import if the option is enabled | [Adobe helpx](https://helpx.adobe.com/lightroom/help/photo-video-import-options.html) |
| digiKam | Haar-wavelet perceptual similarity within a user-set percentage range | Groups each match with a "reference" image, chosen by a configurable rule (older or larger, preferred folder, newer creation or modification date) | [digiKam manual](https://docs.digikam.org/en/left_sidebar/similarity_view.html) |

The tools that document their method use either strict metadata matching (Lightroom) or perceptual similarity (digiKam). digiKam's "reference image selection" setting shows that after deciding two items are the same, a tool still has to decide which copy wins. That is a separate decision.

---

## 6. Failure-mode matrix

✔ = usually matches, ✘ = usually misses, ⚠ = false-positive risk, ? = not established by sources.

| Scenario | Exact hash | Apple IDs | Composite key | Perceptual |
|---|---|---|---|---|
| Same file, renamed or moved | ✔ | ✔ | ✔ (✘ if filename in key) | ✔ |
| HEIC vs auto-converted JPEG of the same shot | ✘ | ? (MakerNote survival undocumented) | ✘ if size or extension is in the key; ✔ if capture time only | ✔ usually (check orientation) |
| Metadata stripped | ✔ if bytes otherwise unchanged, ✘ usually | ✘ | ✘ | ✔ |
| Metadata edited (date shift, keywords) | ✘ | ✔ if MakerNote untouched | ✘ for date edits | ✔ |
| Edited vs original | ✘ | ? | ✔ (same capture time) | ✔ for light edits, ✘ for crops or heavy edits |
| Burst frames (distinct photos) | ✔ distinct | ⚠ BurstUUID is shared | ⚠ without sub-seconds | ⚠ near-identical frames |
| Live Photo still + video | — (two files) | ✔ ContentIdentifier pairs them | ? | ✘ for video |
| Non-Apple camera or screenshot | ✔ | ✘ (absent) | depends on EXIF | ✔ |

---

## 7. Open facts not established here

- Whether iOS's HEIC-to-JPEG conversion (on transfer or share) keeps the Apple MakerNote, including ContentIdentifier. This can be tested with ExifTool on sample files.
- Whether iPhone writes a stable EXIF ImageUniqueID, or Apple MakerNote ImageUniqueID or PhotoIdentifier, on every capture, and whether that value survives edits and conversion.
- Apple Photos' actual duplicate criteria, which Apple does not document.
- How a Photos library stores originals on disk. Apple does not document the library package's internal layout. PhotoKit exposes `originalFilename` as "the original filename of the asset resource from when it was created or imported" ([Apple](https://developer.apple.com/documentation/photos/phassetresource/originalfilename)).
