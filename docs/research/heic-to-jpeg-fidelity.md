# HEIC to JPEG fidelity

Research for [#5](https://github.com/bradenhc/photothing/issues/5), part of map [#1](https://github.com/bradenhc/photothing/issues/1).

**Question.** When a HEIC photo from an Apple device is converted to JPEG, what is preserved, what is lost, and what is at risk? Covers EXIF/XMP/GPS, orientation, colour profiles, HDR gain maps, depth/portrait data, Live Photo linkage, multi-image HEICs (bursts), and quality/size, with Apple's own conversion as a reference point.

**Scope.** Domain facts only. No library or technology selection. Researched 2026-10-18.

**Source trust.** ISO/IEC 23008-12 (HEIF) is paywalled, so HEIF structure is cited from the Nokia HEIF reference site (Nokia was a primary contributor to the standard) and from quotations of the standard. Apple facts come from developer.apple.com and support.apple.com. Tag-level facts come from the ExifTool tag reference. Where a claim rests on a weaker source, it is marked **(lower confidence)**.

---

## Summary

| Aspect | Through HEIC to JPEG | Notes |
|---|---|---|
| EXIF (incl. GPS, capture time, camera) | **Preservable** | Both containers carry EXIF. JPEG puts it in one APP1 segment, which has a 64 KB ceiling. |
| XMP | **Preservable** | Large XMP packets need JPEG "Extended XMP". |
| Apple MakerNote (HDR headroom, Live Photo ID, burst ID) | **At risk** | Lives inside EXIF. It survives only if the converter copies the MakerNote byte-for-byte. |
| Orientation | **At risk** | HEIF rotates using `irot`/`imir` properties. EXIF Orientation is descriptive only, and iPhone writes both. A converter must either bake the rotation into the pixels or set EXIF Orientation, and not do both. |
| Colour profile (Display P3) | **Preservable** | HEIF `colr` box maps to a JPEG APP2 `ICC_PROFILE`. Dropping it shifts colours (P3 gets read as sRGB). |
| Bit depth | **Lost if >8-bit** | Baseline JPEG is 8 bits per sample. |
| HDR gain map | **At risk / format-dependent** | JPEG can carry a gain map (MPF secondary image plus metadata), but Apple's own pipeline has been observed turning ISO 21496-1 maps into Apple-format maps or dropping them. |
| Depth / disparity / portrait mattes | **At risk** | They are HEIF auxiliary images. Apple says it can embed them in JPEG too, but only an aware converter carries them across. |
| Live Photo pairing | **Preservable but fragile** | The link is a shared UUID in the still's Apple MakerNote (tag 0x0011) and in the MOV. Lose the MakerNote and the pair breaks. |
| Bursts | **Not a HEIF multi-image issue on iPhone** | Each burst frame is its own photo, grouped by `BurstUUID` in the MakerNote. Grid/tile HEICs are one picture made of tiles and must be reassembled. |
| Quality / size | **Lossy re-encode** | It is a second generation of lossy compression. Apple says HEIF compresses better than JPEG at the same visual quality, so a JPEG of equal quality is larger. |

Apple does **not** publish what its automatic USB conversion ("Transfer to Mac or PC" → Automatic) preserves. Only the existence of the conversion and the opt-out are documented.

---

## 1. Container background (HEIF vs JPEG)

- In HEIF, a still image is an *item*. "All image items are independently coded and do not depend on any other item in their decoding." Images that are correlated, such as "image bursts or animations like cinemagraphs", can be stored efficiently as sequences/tracks. [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html)
- **Derived images.** A *grid* reconstructs one image "of input images of the same width and height", and an *overlay* composites inputs onto a canvas. [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html)
- **Auxiliary images.** An auxiliary image is one that "complements a master image. For example, an alpha plane or a depth map." [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html)
- **Metadata items.** HEIF metadata items follow "Exif, XMP, or MPEG-7" formats and attach to an image through a `cdsc` item reference. [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html)
- **Transformative properties.** `irot` rotates by 90/180/270° and `clap` crops. Colour is signalled with `colr`, which uses the ISO/IEC 14496-12 ColourInformationBox syntax, wide gamut included. [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html)
- **JPEG side.** EXIF and XMP/ExtendedXMP go in APP1. The ICC profile and MPF (CIPA Multi-Picture Format) go in APP2. ExifTool also lists an APP2 `UniformResourceName` segment "used in Apple HDR images". [ExifTool JPEG tags](https://exiftool.org/TagNames/JPEG.html)
- **MPF.** MPF holds extra images as a JPEG trailer, indexed from APP2. Its MPImageType values include Large Thumbnail, Multi-frame Disparity (0x20002), Multi-angle (0x20003) and **Gain Map Image (0x50000)**. [ExifTool MPF tags](https://exiftool.org/TagNames/MPF.html)

**Implication.** A HEIC is a structured container: a primary item, possibly built from grid tiles, plus auxiliary items, metadata items and properties. A JPEG is one primary image with metadata segments, and extra images can only be bolted on through MPF. Anything beyond "primary pixels + EXIF/XMP/ICC" needs a deliberate mapping.

## 2. EXIF, XMP and GPS

- Both formats carry EXIF and XMP natively (see §1). GPS, capture time and camera/lens data are ordinary EXIF tags, so they carry across whenever EXIF is copied.
- **Size limits in JPEG.** JPEG marker segments top out at about 64 KB. ICC profiles are split across numbered APP2 chunks for this reason, and XMP that outgrows a single APP1 must use the ExtendedXMP mechanism. [ExifTool JPEG tags](https://exiftool.org/TagNames/JPEG.html); the libjpeg `iccjpeg.c` reference implementation describes the 14-byte `ICC_PROFILE\0`+seq+count chunk header and why it is needed ([Chromium copy](https://chromium.googlesource.com/chromium/+/trunk/third_party/iccjpeg/iccjpeg.c)) **(lower confidence: the ICC.1 Annex B text was not read directly)**.
- **Apple MakerNote.** This is an opaque blob inside EXIF that holds several Apple-only facts that matter later in this note ([ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html)):
  - `0x000a HDRImageType`, `0x0021 HDRHeadroom`, `0x0030 HDRGain`: HDR gain-map parameters.
  - `0x0011 ContentIdentifier`: the Live Photo pairing UUID ("called MediaGroupUUID when it appears as an XAttr").
  - `0x0017 LivePhotoVideoIndex`.
  - `0x000b BurstUUID`: "unique ID for all images in a burst".
  - `0x0014 ImageCaptureType`: 1 = ProRAW, 2 = Portrait, 10 = Photo, ...
- **Risk.** A converter that rebuilds EXIF from a parsed tag list, and does not copy the MakerNote verbatim, drops Live Photo pairing, burst grouping and Apple HDR headroom all at once.
- **Photos for Mac export.** "Export Unmodified Original" can write IPTC metadata and keywords "as a sidecar XMP file". The regular Export dialog documents no metadata or location controls for photos. It does for videos ("whether metadata such as a title or caption is included"). [Apple Support: Export photos on Mac](https://support.apple.com/guide/photos/export-photos-videos-and-slideshows-pht6e157c5f/mac)

## 3. Orientation

- The HEIF standard (as quoted by the pillow-heif docs) treats Annex A metadata, EXIF included, as descriptive: it "does not normatively affect the presentation", and rotation is expressed with the `irot` transformative property. [pillow-heif: orientation workaround](https://pillow-heif.readthedocs.io/en/latest/workaround-orientation.html) **(lower confidence: secondary quote of the standard)**
- MIAF (ISO/IEC 23000-22:2024 §7.3.10.1) says files should carry no EXIF-expressed transformations. This is cited in a libavif commit. [libavif commit](https://aomedia.googlesource.com/libavif/+/d3ec9d45116880609b6c0ff38de61a5ab7274836%5E%21/) **(lower confidence)**
- In practice, "iOS creates HEIF files with both Exif orientation and irot property containing equivalent values", and some macOS apps (Preview) rotate according to the EXIF tag. [Siipola, EXIF orientation in different formats](https://zpl.fi/exif-orientation-in-different-formats/); [pillow-heif](https://pillow-heif.readthedocs.io/en/latest/workaround-orientation.html) **(lower confidence)**
- In JPEG, EXIF Orientation is the only rotation signal.
- **Risk.** There are two failure modes. (a) The decoder applies `irot`/`imir` to the pixels, and the converter then also copies EXIF Orientation ≠ 1, so the image rotates twice. (b) The decoder ignores `irot` and the converter drops the EXIF Orientation, so the image is not rotated at all. A correct conversion has exactly one rotation applied, either baked into the pixels with Orientation set to 1, or left unrotated with Orientation set to match.

## 4. Colour profile (Display P3) and bit depth

- HEIF signals colour through `colr`, which can be an ICC profile or nclx code points, and wide colour gamut is in scope. [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html)
- JPEG embeds ICC profiles in APP2 `ICC_PROFILE` segments. [ExifTool JPEG tags](https://exiftool.org/TagNames/JPEG.html)
- Apple's HDR sample writes output with an explicit Display P3 colour space (`CGColorSpace.displayP3`) in developer-reported pipelines. In the Photos export dialog, a "Color Profile" pop-up chooses the output profile. [Apple Developer Forums 791283](https://developer.apple.com/forums/thread/791283); [Apple Support: Export photos on Mac](https://support.apple.com/guide/photos/export-photos-videos-and-slideshows-pht6e157c5f/mac)
- **Risk.** If the ICC profile is dropped, viewers assume sRGB and P3 content looks desaturated. If the converter converts to sRGB, out-of-gamut colours are clipped. That is a permanent loss, but it renders consistently.
- **Bit depth.** Apple's HDR sample writes **10-bit HEIC** (`.RGB10`, `writeHEIF10Representation`). Baseline JPEG is 8-bit per sample, so the extra precision is lost in conversion. [Apple: Supporting HDR images in your app](https://developer.apple.com/documentation/uikit/supporting-hdr-images-in-your-app); [Apple Developer Forums 791283](https://developer.apple.com/forums/thread/791283)

## 5. HDR gain maps

### Two Apple formats

1. **Apple HDR gain map (2020-era).** It is an auxiliary image of type `urn:com:apple:photo:2020:aux:hdrgainmap`. Apple describes it as "an 8-bit, single-channel luminance map", "untagged and formatted as linear data ... encoded using the Rec.709 transfer function", "1/4 the resolution of the original image". It is valid only if metadata has `HDRGainMapVersion`, and the headroom comes from MakerNote keys `33` and `48` (0x21 / 0x30). It is applied as `hdr = sdr * (1 + (headroom - 1) * gainmap)`. [Apple: Applying Apple HDR effect to your photos](https://developer.apple.com/documentation/appkit/applying-apple-hdr-effect-to-your-photos)
2. **ISO 21496-1 "Adaptive HDR" gain map.** It was introduced at WWDC24 and is available as ImageIO `kCGImageAuxiliaryDataTypeISOGainMap` (iOS 18 / macOS 15+). It "encodes the map as the logarithm of the ratio between the HDR and the SDR signal" and "defines new metadata and how to store the new information in common file formats like HEIF and JPEG". "JPEG files also fully support Adaptive HDR, with a slightly different syntax compared to HEIF." [WWDC24 session 10177](https://developer.apple.com/videos/play/wwdc2024/10177/); [kCGImageAuxiliaryDataTypeISOGainMap](https://developer.apple.com/documentation/imageio/kcgimageauxiliarydatatypeisogainmap)

### Backward compatibility

- The file holds "a fully backward-compatible SDR baseline" plus a map and metadata. "The file does not contain an HDR image at all. But rather a set of ingredients." If the gain map is dropped, the result is still a valid SDR picture: you lose the HDR rendition, not the image. [WWDC24 session 10177](https://developer.apple.com/videos/play/wwdc2024/10177/)
- "Once an edit has been applied in HDR, the original Gain Map can no longer be used for tone mapping." [WWDC24 session 10177](https://developer.apple.com/videos/play/wwdc2024/10177/)

### In JPEG

- MPF defines a Gain Map Image type (0x50000). Apple HDR JPEGs use the APP2 `UniformResourceName` segment and the XMP `apdi` namespace ("Used in Apple HDR GainMap images": AuxiliaryImageType, NativeFormat, StoredFormat). [ExifTool MPF](https://exiftool.org/TagNames/MPF.html); [ExifTool JPEG](https://exiftool.org/TagNames/JPEG.html); [ExifTool XMP](https://exiftool.org/TagNames/XMP.html)

### Observed Apple conversion behaviour **(lower confidence: developer report, Apple did not confirm)**

A developer report on Apple's forums (July 2025) found:

- With an ISO 21496-1 source, Core Image HEIC output keeps HDR, but **both JPEG output paths lose HDR**.
- With an Apple-gain-map source, HEIC output is converted to ISO 21496-1, while JPEG output via `.hdrGainMapImage` produces an Apple-gain-map JPEG.
- Cropping an ISO 21496-1 JPEG in Photos produced an Apple-gain-map JPEG.

The only Apple reply asked for code. [Apple Developer Forums 791283](https://developer.apple.com/forums/thread/791283)

**Risk.** Gain-map survival depends on the path and on the gain map format. The gain map can be silently dropped (leaving SDR only) or rewritten into the other format.

## 6. Depth, disparity and portrait mattes

- Depth-capable cameras record "per-pixel scene depth information". "By saving color and depth data separately, you can even apply and change these effects long after a photo has been captured." [Apple: Capturing photos with depth](https://developer.apple.com/documentation/avfoundation/capturing-photos-with-depth)
- With `embedsDepthDataInPhoto` (default true), the depth map is embedded "as an embedded attachment when you flatten the photo data for output in compatible file formats". [Apple: embedsDepthDataInPhoto](https://developer.apple.com/documentation/avfoundation/avcapturephotosettings/embedsdepthdatainphoto)
- The portrait effects matte is embedded "as HEIF and JPEG in the photo", so **Apple supports these attachments in JPEG as well as HEIF**. [Apple: embedsPortraitEffectsMatteInPhoto](https://developer.apple.com/documentation/avfoundation/avcapturephotosettings/embedsportraiteffectsmatteinphoto)
- ImageIO exposes these as auxiliary data types: depth, disparity, portrait effects matte, semantic segmentation mattes (skin and others), and HDR / ISO gain map. They are read with `CGImageSourceCopyAuxiliaryDataInfoAtIndex`, which "returns auxiliary data, such as mattes and depth information, that accompany the image". [Apple ImageIO](https://developer.apple.com/documentation/imageio/cgimagesourcecopyauxiliarydatainfoatindex(_:_:_:))
- Depth maps are "nonrectilinear" and include camera calibration data. Edits to the image require "derivative AVDepthData objects reflecting the edits". [Apple: AVDepthData](https://developer.apple.com/documentation/avfoundation/avdepthdata)
- **Risk.** A converter that only decodes the primary item drops all auxiliary images. After that, Portrait-mode effects cannot be re-adjusted. Resizing, cropping or rotating the primary image without transforming the auxiliary maps leaves them misaligned.

## 7. Live Photo linkage

- A Live Photo is a still plus a paired video. In PhotoKit the video is `PHAssetResourceType.pairedVideo`, "the original video data component of a Live Photo asset". [Apple: pairedVideo](https://developer.apple.com/documentation/photos/phassetresourcetype/pairedvideo)
- Pairing is by a shared identifier. On the still it is Apple MakerNote `0x0011 ContentIdentifier`. On the MOV it is the QuickTime key `com.apple.quicktime.content.identifier`. The MOV also has a `still-image-time` timed-metadata marker. [ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html); [ExifTool QuickTime tags](https://exiftool.org/TagNames/QuickTime.html); community implementations such as [makelive](https://pypi.org/project/makelive/) **(still-image-time detail: lower confidence)**
- Apple's "Export Unmodified Original" exports "a Live Photo ... as two separate files: a still-image file and a video file". A regular Export can "export a Live Photo as a still image". [Apple Support: Export photos on Mac](https://support.apple.com/guide/photos/export-photos-videos-and-slideshows-pht6e157c5f/mac)
- **Risk.** Converting HEIC to JPEG does not touch the MOV. Pairing survives only if the JPEG keeps the MakerNote with the same ContentIdentifier and the MOV is kept alongside. A filename convention alone (IMG_1234.HEIC / IMG_1234.MOV) is not the system's pairing key.

## 8. Multi-image HEICs and bursts

- HEIF *can* hold bursts and sequences (§1). On iPhone, though, burst frames show up in Photos as a burst of individual shots. On Mac you choose "Keep Only Selection" (the unselected shots go to Recently Deleted) or "Keep Everything". [Apple Support: View photo bursts](https://support.apple.com/guide/photos/view-photo-bursts-pht8745d2677/mac)
- Burst membership is recorded per image in Apple MakerNote `0x000b BurstUUID`. [ExifTool Apple tags](https://exiftool.org/TagNames/Apple.html)
- **Grid images.** A HEIC whose primary item is a `grid` derivation is still one picture. A converter has to decode every tile and stitch them together, or it outputs a single tile or fails. [Nokia HEIF technical reference](https://nokiatech.github.io/heif/technical.html) That iPhone primary images are tiled grids is widely observed but was not confirmed in an Apple primary source **(lower confidence)**.
- **Risk.** If a converter encounters a real multi-image HEIF (several top-level items, a sequence track, or entity groups), only the primary item becomes a JPEG and the other images are lost unless they are exported as separate JPEGs. Thumbnails, which are separate items, are normally regenerated rather than carried over.

## 9. Quality and size

- "HEIF and HEVC offer better compression than JPEG and H.264, so they use less storage space ... while preserving the same visual quality." [Apple Support 116944](https://support.apple.com/en-us/116944) The corollary is that a JPEG of comparable visual quality is larger than the source HEIC.
- Going from HEIC to JPEG decodes one lossy codec and re-encodes with another, so it is a generational loss. It cannot be undone, and an original-quality round trip is not possible.
- Photos for Mac exposes a "JPEG Quality" pop-up and a separate Size control. Apple's guide describes the quality setting in terms of resolution. [Apple Support: Export photos on Mac](https://support.apple.com/guide/photos/export-photos-videos-and-slideshows-pht6e157c5f/mac)
- `kCGImageDestinationEmbedThumbnail` (default false) controls whether ImageIO embeds a thumbnail in JPEG/HEIF output. In Apple's own API, thumbnails are not carried over by default. [Apple ImageIO](https://developer.apple.com/documentation/imageio/kcgimagedestinationembedthumbnail)

## 10. Apple's own conversion paths (reference behaviour)

From [Apple Support 116944](https://support.apple.com/en-us/116944) and the [Photos export guide](https://support.apple.com/guide/photos/export-photos-videos-and-slideshows-pht6e157c5f/mac):

| Path | Documented behaviour |
|---|---|
| USB import to Photos, Image Capture, or a PC | "the media might be converted to JPEG or H.264". To opt out: Settings → Apps → Photos → **Keep Originals**. (The other value of this "Transfer to Mac or PC" setting is Automatic.) |
| AirDrop / Messages / email | If the receiver lacks support, "the media might automatically be shared in a more compatible format, such as JPEG or H.264". |
| iCloud Photos | "Preserved in its original format, resolution, and frame rate." |
| Capture setting | Camera → Formats → **Most Compatible** captures JPEG/H.264 directly, so no conversion happens later. |
| Photos/Preview File → Export | User picks JPEG/HEIC/TIFF/PNG, JPEG Quality, Color Profile, Size. A Live Photo exports as a still. |
| Export Unmodified Original | Original format. Live Photo exported as two files. Optional IPTC→XMP sidecar. |

**Gap.** None of these Apple pages says what the automatic conversion keeps: EXIF/GPS, MakerNote, ICC profile, gain map or depth. That would need empirical testing (beyond the scope of this note).

---

## Open questions (not resolved from primary sources)

- What exactly iOS "Automatic" USB transfer writes into the JPEG: MakerNote, Display P3 ICC, gain map (and which format), depth?
- The normative ISO/IEC 23008-12 and ISO 21496-1 text on JPEG gain-map syntax (both paywalled).
- Whether current iPhone HEIC captures ever contain more than one top-level displayable image.
