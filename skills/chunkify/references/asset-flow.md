# Asset integration flow

Use this when the request creates or plays Chunkify Assets. Check [the current public OpenAPI schema](https://chunkify.dev/docs/openapi.json), the official SDK for the application's language, and the configured Project before implementation. The schema defines exact request fields, response types, and event payloads; this reference identifies the decisions that connect them.

## Create and track an Asset

- A browser upload starts with an Upload session created by the application server through the SDK. Set `create_asset: true`, which otherwise defaults to false, and put the application's video record ID in Upload metadata. Return only the fields needed by `@chunkify/uploader`. The uploader handles the session's single PUT or multipart transfer and completion. Transferring bytes alone does not create the Source or Asset.
- For a hosted or server-side input, create a Source through the SDK, then create an Asset from its `source_id`. Use the Project's configured output Storage unless the request requires an override. Input and output Storage selections are independent.
- Persist the application's video record before an upload can complete. Save Upload and Asset IDs as they become available, and resolve the Asset through the SDK when an event needs fresher state. Upload completion can race the browser's completion callback.

## Events and readiness

- Subscribe to the events needed by the flow: `upload.created`, `upload.completed`, `upload.failed`, `upload.expired`, `asset.created`, `asset.playback_ready`, `asset.ready`, `asset.failed`, and `asset.expired`. Add `asset.deleted` when deletion is in scope and the `asset.output.*` events for additional outputs. Read `data.output.type` to distinguish output kinds. `upload.completed` contains `data.upload` and `data.source`; its Upload includes `asset_id` when automatic creation succeeded. Parent Asset events contain `data.asset`; output events contain the canonical output in `data.output`.
- `asset.playback_ready` means the initial HLS stream is playable, including while the Asset is `preparing`. `asset.ready` means playback, the poster, and timeline previews are all ready. It does not mean every HLS rendition has been transcoded. Use the stable master playlist URL; Chunkify transcodes other HLS renditions when the player first requests them. A failed Asset carries error details.
- A configured Workflow starts its `asset.ready` actions when the Asset reaches `ready`. Static MP4 renditions, transcripts, captions and thumbnails have independent statuses and can finish later. Caption translations require an original caption track. Update these outputs as their own events arrive without delaying initial playback.
- Verify the raw webhook body with the official SDK before parsing it. Deduplicate by notification `id`, accept out-of-order delivery, and avoid regressing ready or failed application state. Event payloads are snapshots; get the Asset for current state when reconciling.

## Delivery fields

- The Asset's `cdn_url` is the unsigned HLS master URL. `poster.cdn_url` and `timeline_previews.cdn_url` supply the poster and WebVTT thumbnails. Save these in application fields such as `playback_url`, `poster_url`, and `thumbnails_url` when they become available. Keep `asset_id` for later lifecycle events. The WebVTT refers to sprite images; no row per thumbnail is needed.
- Use `playback_ready` to decide when to show streaming playback. Poster and timeline fields can arrive later, so update them without remounting an active player. Read saved URLs during normal views rather than fetching the Asset from Chunkify for every viewer.
- Keep `preview_url` and signed `url` behavior distinct from unsigned CDN delivery. Follow the main skill's retention and expiry rules when the Project uses Chunkify-managed output Storage.

## Check the completed path

Verify one real input through Asset creation, webhook delivery, saved application state, HLS playback, poster, and timeline previews. Check a failed Upload or Asset, duplicate webhook delivery, and a page reload while the Asset is preparing. Use the actual application's origin for CORS and media checks.
