# Migrate from Jobs

Use this when an application creates Chunkify Jobs and should move to Transcodings, Thumbnails, or Assets. Check [the current public OpenAPI schema](https://chunkify.dev/docs/openapi.json) for exact fields; this reference covers the decisions a migration needs.

The Jobs API keeps working during the transition, so migrate one call site at a time and keep unmigrated paths running. Leave existing Job records, files, and stored URLs as they are unless the user asks to reprocess them. Jobs that are still in flight finish and send `job.*` events as before.

## Choose the target

Find every Job creation in the application, whether through the SDK (`client.jobs.create`, `client.Jobs.New`, `$client->jobs->create`) or HTTP `POST /jobs`. Also find Job reads, file listings, `job.*` webhook handlers, and stored Job IDs. Classify each creation by `format.id`:

| Legacy format | Target |
| --- | --- |
| `mp4_h264`, `mp4_h265`, `mp4_av1`, `webm_vp9` | Transcoding with the same `format.id` |
| Any of those with `disable_video: true` | Transcoding with `format.id: "m4a_aac"` |
| `hls_h264` | Asset |
| `hls_h265`, `hls_av1`, or any HLS Job with `hls_enc: true` | Stop and ask the user |
| `jpg` | Thumbnails, or the Asset's timeline previews for player scrubbing |

Transcodings produce standalone files and no HLS. Assets handle streaming with a managed H.264 HLS ladder. Chunkify chooses the CPU type and transcoder count for every request, so `transcoder` is no longer an input.

Report the migration plan per call site before editing: the target, the settings that carry over, and the settings that will be dropped.

## Files: Transcodings

Keep `source_id`, `metadata`, `storage.id`, and `storage.path`. `storage.path` keeps its behavior: it is relative to the Storage `base_prefix`, required for customer Storage, and generated for Chunkify Storage. Its extension must match the output: `.mp4`, `.webm`, or `.m4a`. An audio-only output therefore needs a `.m4a` path.

Remove the fields Transcodings do not accept. Unknown fields are rejected, not ignored, so a leftover legacy field returns a validation error:

- Remove `transcoder` and `hls_manifest_id` at the top level.
- Remove `minrate` from H.264, H.265, and AV1 formats; only `webm_vp9` keeps it. Express the rate limit with `maxrate` and `bufsize`, or drop it.
- Change `channels: 5` to `6` and `channels: 7` to `8`; `6` means 5.1 and `8` means 7.1.
- Keep `pixfmt` only when the target format's schema still lists it. Full-range `yuvJ`, big-endian, `yuv410p`, and `yuv411p` values are gone, and AV1 accepts only `yuv420p` and `yuv420p10le`. Ask before replacing a pixel format, because the closest value can change color sampling or bit depth.
- For `disable_video: true`, switch to `m4a_aac` and keep only `audio_bitrate`, `channels`, `audio_samplerate`, `seek`, and `duration`.

Other format settings keep their names. `seek` and `duration` accept fractional seconds, and `seek` can be `0`. The default `preset` is now `faster` for H.264 and `8` for AV1. Set the old default, `veryfast` or `10`, explicitly if the application relied on it. Check the format's limits in the schema when a value sits near a legacy bound.

## Streaming: Assets

Replace each `hls_h264` Job, or each group of Jobs sharing an `hls_manifest_id`, with one Asset for the Source. Then follow [the Asset flow](asset-flow.md) for events, delivery fields, and playback.

- Set `max_resolution` to `1080p`, or to `2160p` when the highest legacy rung exceeded 1080 pixels. Smaller sources are not upscaled.
- Set `min_resolution` to the smallest of `240p`, `360p`, `540p`, `720p`, `1080p`, `1440p`, or `2160p` that covers the lowest legacy rung, or omit it to use the Project default.
- Replace a folder in `storage.path` with `path_template`, which may reference `{assetID}` and `{metadata.<key>}`. The master playlist is always `manifest.m3u8`. Keep `storage.id` when it differs from the Project default.

List each dropped setting to the user with its Job: per-rung `video_bitrate`, `audio_bitrate`, `crf`, `preset`, `profilev`, `level`, `gop`, keyframe interval, `framerate`, `pixfmt`, `channels`, `width`, `height`, `hls_time`, `hls_segment_type`, `seek`, `duration`, and `per_title`. Chunkify manages these for Assets. Also call out a legacy ladder capped below 1080p, which an Asset cannot reproduce.

Stop and ask before migrating `hls_h265`, `hls_av1`, or any Job with `hls_enc`. Assets stream H.264 without segment encryption, so these changes affect device support, file size, or content protection. Present the options: stream H.264 through an Asset, produce H.265 or AV1 files with Transcodings, or keep that Job path on the Jobs API for now.

When a legacy `mp4_h264` Job made a download copy of the same video, offer a `static_rendition` output on the Asset, selected by `resolution`. Keep a Transcoding when the application needs other codecs or encoding controls.

## Images: Thumbnails

Map a `jpg` Job to one Thumbnails request:

- Replace `interval` with exact `timestamps` when the code path knows the Source duration: `0`, `interval`, `2 × interval`, and so on, below the end of the range and capped at 100 values. When the duration is unknown, use `count = ceil(duration / interval)` at most 100. Note that `count` samples the middle of equal sections rather than fixed intervals.
- When `frames` is set, also cap the number of thumbnails at it.
- Replace `seek` and `duration` with `range.start` and `range.end = seek + duration`.
- Replace `width` and `height` with `size`, the maximum edge from 64 to 1920 pixels. Aspect ratio is preserved without cropping.
- Replace `sprite: true` with `output: "sprites"`. Sprite sheets are laid out automatically, and each sample reports its tile `crop`.
- Remove `chunk_duration`. Keep `format: "jpeg"`, the default, unless the user wants `webp`.
- Keep `storage.id` and `storage.path`. As for Transcodings, the path is relative to the Storage `base_prefix`, required for customer Storage, and generated for Chunkify Storage. Its filename keeps one `%05d` placeholder, numbered from 1 for images or sprite sheets, and ends in `.jpg` for `jpeg` or `.webp` for `webp`. The WebVTT file is named after the template without the placeholder, such as `thumb.vtt` for `thumb-%05d.jpg`. Read `storage_prefix` and the file and sample paths from the response rather than rebuilding keys.

When a sprite feeds a player's scrubbing previews and the same video is moving to an Asset, use the Asset's `timeline_previews.cdn_url` WebVTT instead of a Thumbnails request. For additional images of a video that is already an Asset, create a `thumbnail` output on that Asset.

## Events and results

Subscribe the application's webhook to the new events before switching a call site; keep `job.*` subscribed until no Jobs are in flight. Changing a shared webhook subscription changes remote configuration, so prepare the exact update and get approval.

| Legacy | Replacement |
| --- | --- |
| `job.completed`, `data.job` and `data.files` | `transcoding.ready` with `data.transcoding`, whose `files` are embedded |
| `job.failed`, `job.cancelled` | `transcoding.failed`, `transcoding.cancelled` |
| `job.*` for `jpg` | `thumbnail.created`, `thumbnail.ready`, `thumbnail.failed`, `thumbnail.deleted` with `data.thumbnail` |
| `job.*` for HLS | `asset.*` events, as described in [the Asset flow](asset-flow.md) |

- Transcoding `status` is `preparing`, `ready`, `failed`, `cancelled`, or `expired`; read `progress` and `processing_stage` (including `queued`) for detail. Remove checks for the intermediate Job states such as `ingesting`, `transcoding`, or `merging`.
- Thumbnail `status` is `preparing`, `ready`, `failed`, or `expired`. Its `samples` list each requested time with its measured `timestamp`, `path`, dimensions, and sprite `crop`. Its `files` carry `url` and `cdn_url`.
- Replace `/jobs/{id}/files` and `/files` lookups with the `files` embedded in the resource or its event. Remove reads of Job transcoders and logs; Transcodings expose no per-transcoder detail.
- Save stable `cdn_url` values for durable delivery. Signed `url` values and Chunkify temporary Storage outputs expire; see `expires_at`.
- A Transcoding's price is fixed when it is created and covers the accepted settings and trim interval.

## SDK calls

Upgrade the application's official SDK to the release that provides `transcodings`, `thumbnails`, and `assets`. Method and type names follow the published SDK conventions; verify them against the installed package before finishing.

TypeScript:

```ts
const transcoding = await client.transcodings.create({
  source_id: sourceId,
  format: {
    id: "mp4_h264",
    height: 720,
    crf: 23,
  },
  storage: {
    path: `videos/${videoId}/720p.mp4`,
  },
});

const thumbnail = await client.thumbnails.create({
  source_id: sourceId,
  timestamps: [0, 10, 20, 30],
  output: "sprites",
  size: 320,
});
```

Python:

```python
transcoding = client.transcodings.create(
    source_id=source_id,
    format={
        "id": "mp4_h264",
        "height": 720,
        "crf": 23,
    },
    storage={
        "path": f"videos/{video_id}/720p.mp4",
    },
)

thumbnail = client.thumbnails.create(
    source_id=source_id,
    timestamps=[0, 10, 20, 30],
    output="sprites",
    size=320,
)
```

PHP:

```php
$transcoding = $client->transcodings->create(
    sourceID: $sourceId,
    format: [
        'id' => 'mp4_h264',
        'height' => 720,
        'crf' => 23,
    ],
    storage: [
        'path' => "videos/{$videoId}/720p.mp4",
    ],
);

$thumbnail = $client->thumbnails->create(
    sourceID: $sourceId,
    timestamps: [0, 10, 20, 30],
    output: 'sprites',
    size: 320,
);
```

Go:

```go
transcoding, err := client.Transcodings.New(
	ctx,
	chunkify.TranscodingNewParams{
		SourceID: sourceID,
		Format: chunkify.TranscodingNewParamsFormatUnion{
			OfMP4H264: &chunkify.TranscodingMP4H264Param{
				Height: chunkify.Int(720),
				Crf:    chunkify.Int(23),
			},
		},
		Storage: chunkify.TranscodingNewParamsStorage{
			Path: chunkify.String("videos/" + videoID + "/720p.mp4"),
		},
	},
)
if err != nil {
	return err
}

thumbnail, err := client.Thumbnails.New(
	ctx,
	chunkify.ThumbnailNewParams{
		SourceID:   sourceID,
		Timestamps: []float64{0, 10, 20, 30},
		Output:     chunkify.ThumbnailOutputSprites,
		Size:       chunkify.Int(320),
	},
)
if err != nil {
	return err
}
```

The other Go format variants follow the same pattern: `OfMP4H265`, `OfMP4Av1`, `OfWebmVp9`, and `OfM4aAac` with the matching `Transcoding...Param` type. Use the SDK's enum constants for constrained values.

## Verify and report

Run the application's tests for each migrated path. Then create one migrated request against a real Source in a non-production Project and check:

- The request is accepted, with no rejected legacy field.
- The output lands at the expected path with the expected extension, codec, and dimensions.
- The new webhook events reach the handler and update application state, including failure and duplicate delivery.
- Saved URLs load from the application's origin.

For an Asset migration, also verify playback as described in [the Asset flow](asset-flow.md).

Report, for each call site: its new target, the settings carried over, the settings dropped, the decisions the user still has to make, and the paths left on the Jobs API.
