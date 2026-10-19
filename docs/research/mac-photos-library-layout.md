# Mac Photos library on-disk layout

Research for [#2](https://github.com/bradenhc/photothing/issues/2) (part of map [#1](https://github.com/bradenhc/photothing/issues/1)). This covers facts only, with no technology selection. Researched 2026-10-18.

**Source trust.** Apple documents the library only at the user and PhotoKit level. Apple publishes nothing about the internal folder layout or the database schema. Those details come from [osxphotos](https://github.com/RhetTbull/osxphotos), an established open-source reverse-engineering project. Its maintainers test it against every macOS release from 10.12 through macOS 26 (Tahoe) and 27, and its source code is the closest thing to a primary reference for the format. Each claim below is tagged:

- **[Apple]** comes from Apple's own documentation.
- **[osxphotos]** comes from osxphotos source or docs. Apple documents none of these details, so they can change between macOS releases.

## 1. What the library is

- The library is a macOS package (a directory that Finder shows as one file) named `*.photoslibrary`. The System Photo Library lives in `~/Pictures` by default and can be moved elsewhere, including to an external drive. A user can have several libraries, but iCloud Photos syncs only the System Photo Library. [Apple: [System Photo Library overview](https://support.apple.com/guide/photos/pht211de786/mac), [Designate a System Photo Library](https://support.apple.com/en-us/104946)]
- Apple warns: *"To avoid accidentally deleting or corrupting a Photos library, do not alter the contents of a library in the Finder."* [Apple: [overview, Photos 6.0](https://support.apple.com/en-ie/guide/photos/pht211de786/6.0/mac)]

## 2. Internal layout (Photos 5+, i.e. macOS 10.15 and later)

All of the following is [osxphotos], from [`photoinfo.py`](https://github.com/RhetTbull/osxphotos/blob/main/osxphotos/photoinfo.py), [`photosdb.py`](https://github.com/RhetTbull/osxphotos/blob/main/osxphotos/photosdb/photosdb.py), [`_constants.py`](https://github.com/RhetTbull/osxphotos/blob/main/osxphotos/_constants.py) and [`adjustmentsinfo.py`](https://github.com/RhetTbull/osxphotos/blob/main/osxphotos/adjustmentsinfo.py).

| What | Path inside `X.photoslibrary/` |
|---|---|
| Library database (SQLite, Core Data) | `database/Photos.sqlite` (Photos 4 and earlier used `database/photos.db`) |
| Originals ("masters") | `originals/<dir>/<file>`. `<dir>` and `<file>` come from the asset's `ZDIRECTORY` / `ZFILENAME` columns. In practice `<dir>` is the first hex character of the UUID. Photos 4 and earlier used `Masters/`. |
| Live Photo video (original) | Next to the still in `originals/`, as `<stem>_3.mov` |
| RAW of a RAW+JPEG pair | Next to the original in `originals/`, as `<stem>_4.<ext>` |
| Edited photo render | `resources/renders/<UUID[0]>/<UUID>_1_201_a.jpeg` (`.heic` for HEIC originals on Photos 6+) |
| Edited video render | `resources/renders/<UUID[0]>/<UUID>_2_0_a.mov` |
| Edited Live Photo video | `resources/renders/<UUID[0]>/<UUID>_2_100_a.mov` |
| Edit recipe (adjustments sidecar) | `resources/renders/<UUID[0]>/<UUID>.plist`, a plist with `adjustmentData`, `adjustmentFormatIdentifier`, `adjustmentEditorBundleID`, `adjustmentTimestamp`, and others. This is the in-library counterpart of an iPhone `.AAE`. |
| Previews / thumbnails | `resources/derivatives/…`, e.g. `resources/derivatives/masters/<UUID[0]>/<UUID>_4_5005_c.jpeg` |
| Shared-album items | `resources/cloudsharing/data` (Photos 5–7); `scopes/cloudsharing/data` (Photos 8+ / macOS 13+) |
| Shared Library / shared-moment items | `scopes/momentshared/originals`, `scopes/syndication/originals` (with their own `resources/derivatives`) |

Notes:

- **Filenames inside the library are not the user's filenames.** Photos 5+ renames each file to its UUID when it imports it. The original name is stored only in the database, in `ZADDITIONALASSETATTRIBUTES.ZORIGINALFILENAME`. [osxphotos API docs: `original_filename`]
- **The schema changes between releases.** The asset table is `ZGENERICASSET` on Photos 5 (macOS 10.15) and `ZASSET` on Photos 6+ (macOS 11+). Columns have also changed, for example the adjustments flag `ZHASADJUSTMENTS` became `ZADJUSTMENTSSTATE` in 9.9. Photos versions are told apart by the Core Data model version, which osxphotos maps to Photos versions 5–12. Photos 12 has model numbers starting `270…`. [osxphotos `_constants.py`]

### Referenced (non-managed) files

- If "Copy items to the Photos library" is turned off, imported files stay where they were. Apple calls these **referenced files**. They are not uploaded to iCloud, are not included when the library is backed up, and become unreachable if the drive is disconnected or the files are moved or renamed. [Apple: [Change where Photos stores your files](https://support.apple.com/guide/photos/pht1ed9b966d/mac)]
- In the database, a referenced file's location is a volume name plus a path relative to that volume (`ZFILESYSTEMVOLUME.ZNAME` + `ZFILESYSTEMBOOKMARK.ZPATHRELATIVETOVOLUME`, resolved under `/Volumes/`). The asset's `ZDIRECTORY` can then be an absolute path instead of one relative to `originals/`. [osxphotos]

## 3. Where capture dates and other metadata live

- **In the original file.** The file inside `originals/` is the file as imported, so it carries whatever embedded metadata it arrived with, such as EXIF/XMP for photos and QuickTime metadata for video. Photos' own "Export Unmodified Original" exports *"in the original format used when they were imported"*. A Live Photo exports as two files, a still and a video. [Apple: [Export](https://support.apple.com/guide/photos/pht6e157c5f/mac)]
- **In the library database.** The date Photos displays (`ZASSET.ZDATECREATED`, a Core Data timestamp) and its timezone offset (`ZADDITIONALASSETATTRIBUTES.ZTIMEZONEOFFSET`) are database columns. So are title, keywords, location, albums, people, favorites, the original filename, and the edit state. osxphotos reports both a current `date` and a `date_original`. Photos' "Adjust Date and Time" edits the database. [osxphotos]
- **The two can disagree.** Photos' native export *"does not preserve most of this metadata"*. osxphotos offers `--exiftool` and `--sidecar` options precisely to carry database metadata into exported copies. A reader that trusts only the file's embedded metadata will miss the user's date, title, and location changes. A reader that trusts only the database depends on an undocumented schema. [osxphotos README] (Apple does not say explicitly whether Photos ever writes back into originals. osxphotos's model assumes it does not.)

## 4. Enumerating and copying originals without modifying the library

There are two documented routes.

**a) Read the package directly** (osxphotos's approach: *"osxphotos works by copying photos out of the Photos library folder"*).

- Photos *"maintains an exclusive lock on the database file while Photos is open"*. osxphotos checks for that lock. If the database is locked, osxphotos copies `Photos.sqlite` to a temp file, then opens it with a read-only SQLite connection. [osxphotos `photosdb.py`]
- Originals are then copied from `originals/` using the database to map UUID-named files back to assets, original filenames, Live Photo pairs, RAW pairs, and edit state. This requires parsing the undocumented, version-dependent schema (section 2), and Apple warns against touching the package in Finder (section 1). Reading without writing does not violate that warning, but nothing in Apple's documentation guarantees the result.
- macOS privacy controls (TCC) protect `~/Pictures/*.photoslibrary`, so a process may need the user's permission or Full Disk Access to read it. (During this research, a plain `ls` of the library from a terminal hung instead of returning.)

**b) PhotoKit** (Apple's supported API).

- `PHPhotoLibrary` *"represents the entire set of assets and collections that the Photos app manages, including assets stored on the local device and those stored in iCloud Photos"*. Access needs the user's authorization. [Apple: [PHPhotoLibrary](https://developer.apple.com/documentation/photos/phphotolibrary)]
- Each `PHAsset` has one or more `PHAssetResource`s, *"like when backing up or restoring assets"*. A photo can have both JPEG and RAW resources. A Live Photo has a still and a video. An edited asset has resources for before the edit, after the edit, and the adjustment data. [Apple: [PHAssetResource](https://developer.apple.com/documentation/photos/phassetresource)]
- Resource types include `photo` / `video` (original data), `pairedVideo` (Live Photo original video), `alternatePhoto` (non-primary form, e.g. the other half of RAW+JPEG), `fullSizePhoto` / `fullSizeVideo` / `fullSizePairedVideo` (the current edited version), `adjustmentData`, and the `adjustmentBase*` types. [Apple: [PHAssetResourceType](https://developer.apple.com/documentation/photos/phassetresourcetype)]
- `PHAssetResourceManager.writeData(for:toFile:options:)` writes a resource's bytes to a local file and *"may download asset data from the network"*, depending on the options. [Apple: [writeData](https://developer.apple.com/documentation/photos/phassetresourcemanager/writedata(for:tofile:options:completionhandler:))]
- PhotoKit works on the library Photos manages, which is the System Photo Library (see section 1). Apple's documentation does not describe pointing PhotoKit at an arbitrary `.photoslibrary` path.

## 5. When "Optimize Mac Storage" is on

- With iCloud Photos on, Optimize Mac Storage *"stores smaller versions of your photos on your Mac when storage space is limited"* and *"keeps the original, full-size photos in iCloud"*. Choosing "Download Originals to this Mac" brings the originals back. [Apple: [Optimize storage](https://support.apple.com/guide/photos/phta9b4673b4/mac)]
- **On disk:** the database still lists every asset. For an asset whose original is not local, the file in `originals/` is absent. osxphotos marks such an asset `ismissing` when `ZINTERNALRESOURCE.ZLOCALAVAILABILITY != 1` for its original resource (`ZDATASTORESUBTYPE` 1 or 3), and its `path` is `None`. Smaller derivatives under `resources/derivatives` may still be present. `incloud` comes from `ZCLOUDMASTER.ZCLOUDLOCALSTATE == 3`. [osxphotos]
- **Getting the original:** PhotoKit returns an error for non-local data unless `isNetworkAccessAllowed` is `true`. If it is `true`, Photos downloads the data from iCloud and reports progress. [Apple: [isNetworkAccessAllowed](https://developer.apple.com/documentation/photos/phassetresourcerequestoptions/isnetworkaccessallowed)] osxphotos's `--download-missing` drives Photos through AppleScript (which it calls "somewhat buggy"), or through PhotoKit with `--use-photokit`. [osxphotos README]
- Community reports (not Apple documentation) say that a downloaded original may be offloaded again soon after use when space is tight, and that Time Machine does not force downloads. [Apple Community thread [256205582](https://discussions.apple.com/thread/256205582), low trust]

## Facts later decisions depend on

1. Originals live under `originals/` with UUID filenames. The user's original filenames, edits, and the user-visible capture date live only in `database/Photos.sqlite`, whose schema is undocumented and changes between macOS releases.
2. Edited versions are separate renders in `resources/renders/`. Originals are never overwritten. Live Photos are a still plus a `_3.mov`, and RAW+JPEG pairs are two files.
3. Reading without modification is possible: copy or open the database read-only (Photos holds a lock while it is running) and copy files out. PhotoKit is the supported alternative but covers only the System Photo Library.
4. With Optimize Mac Storage, some originals are simply not on disk. Getting them means a download from iCloud, so a fully offline "on-disk only" read of such a library is necessarily incomplete.
5. Referenced files live outside the package (possibly on other volumes) and may be unreachable.
