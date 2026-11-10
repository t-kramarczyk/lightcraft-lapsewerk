# LightCraft for Lapsewerk

Snapshot of [storytold/lightcraft](https://github.com/storytold/lightcraft) at
`5194cc20dea06d725079ac162c6912fc1f0d3a78` (MIT OR Apache-2.0, licences and NOTICE unchanged; upstream `.github/workflows` left out), with one additive
patch used by Lapsewerk. Upstream behaviour is unchanged.

## Patch: one camera look for a sequence

For raws without colour matrices (Sony, Nikon, Canon …) LightCraft fits a look to each file's embedded JPEG. In a
time-lapse each frame then gets a slightly different look (the camera's JPEG processing, e.g. Sony DRO, changes per
frame), which flickers and hides the camera's exposure steps.

- `files::camera_look_of(bytes)` – the look `load_vec` would fit to this file
- `files::load_vec_with_look(bytes, max_edge, &look)` – develop with a given look
- `files::fs_loader_with_look(Arc<CameraLook>)` – a `FileLoader` for `Session::media.file_loader`
- `CameraLook` is public (re-exported from the crate root)
- Test `files::tests::camera_look_of_matches_load`: a file developed with its own look is pixel-identical to `load_vec`

Candidate for an upstream pull request (only with the owner's approval; that would be public).

## Patch 2 – `RawImage::normalized` without per-sample black-level lookups

The black level of each sample (repeat pattern + column delta) is computed once per pattern row instead of with two
divisions and three lookups per sample; the sums are the same in the same order, so the result is bit-identical
(test `normalized_matches_the_per_sample_black_level` compares against the per-sample version for 1 and 3 samples a
pixel, integer and float data with NaN, patterns 1×1 to 3×2, with and without deltas). Sony A7 IV: raw decode
43 → 34 ms a frame. Plain performance change; offered upstream only with the maintainer's consent.

## Patch 3 – hot pixels repaired before the mosaic is read

`RawImage::repair_samples(indices)` replaces listed mosaic samples with the median of the nearest same-colour samples
that are not listed themselves (5 × 5, 7 × 7 if fewer than four). `files::load_vec_for_sequence` /
`fs_loader_for_sequence(look, defects)` load a sequence frame with an optional shared look and the sequence's hot
pixels repaired right after decoding (not for reduced copies, whose sample positions differ). `fs_loader_with_look`
and every existing caller behave as before (`defects: None`). Tests: `repair_samples_takes_the_same_colour_neighbours`;
engine + raw suites 1925 passed. Lapsewerk finds the samples by comparing frames (`lapsewerk_core::defects`).

## Patch 4 – one look for every frame, fitted or not

A night sky (Sony A6700, ISO 800, 15 s) showed a second way per-frame looks flicker: the JPEG fit is refused for most
frames (too few colour pairs) and accepted for a few, which then render about ¾ stop darker than the rest (3 of 169
frames; the raw data agree within 0.05 stops). `files::SequenceLook` says how a sequence's frames get their look:
`Own` (as `load_vec`), `Shared(Arc<CameraLook>)` (as `load_vec_with_look`) or `Unfitted` (no JPEG fit tried for any
frame: the colour matrices, else the spectral ones, else the neutral fallback, with the default tone).
`load_vec_for_sequence` and `fs_loader_for_sequence` take a `SequenceLook` instead of `Option<CameraLook>`; every other
caller behaves as before. Test `sequence_looks_are_one_look_for_every_frame`: with `Unfitted`, a frame whose fit is
refused renders exactly as `load_vec` and one whose fit is accepted gets no fitted tone; `Shared` gives every frame
the shared tone.

## Patch 5 – a crs sidecar's crop on a turned raw

`crs:Crop*` are on the stored image, before the file's orientation; LightCraft's crop is on the oriented photo. A
catalog's crop was already turned (`lightroom_catalog::catalog_crop`), an XMP sidecar's was not: a portrait
raw (Sony A7 IV, `tiff:Orientation="8"`) showed a region a quarter turn off (a time-lapse of a city at sunrise came out
all sky). `parse_sidecar` now turns a `crs:` crop by the sidecar's own `tiff:Orientation` and stored size
(`tiff:ImageWidth`/`ImageLength`, else `exif:PixelX/YDimension`, else 3:2), with the catalog's function (now
`pub(crate)`). Files without an orientation (or 1) are unchanged. Test
`crs_crop_of_a_turned_raw_lands_on_the_oriented_photo`; engine suite 1314 passed.

## Patch 6 – a crs sidecar's masks on a turned raw

Like the crop (patch 5), a crs sidecar keeps a mask's geometry on the stored image. A portrait raw's radial filter over the
horizon (`crs:Left/Right/Top/Bottom` narrow and tall at the stored image's left) landed as a narrow band at the left
edge: a visible stripe in a time-lapse sky, and the horizon not toned. `crs_masks::orient_masks` turns every point of
a mask (linear start/end, radial centre, brush points) by the sidecar's `tiff:Orientation` and swaps a radial's radii
on a quarter turn (a mirror turns its angle). Called from `parse_sidecar` beside the crop. Test
`crs_radial_of_a_turned_raw_lands_on_the_oriented_photo`; engine suite 1315 passed.
