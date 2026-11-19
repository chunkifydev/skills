# Browser upload and playback CORS

Use this when browser uploads reach customer storage or playback reads its CDN. Read the current bucket and CDN policies first. Derive the new rules from the actual page origins, upload PUT and `Content-Type` request, HLS playlist and segment reads, poster and timeline requests, and required response headers. Allow only the application's own origins by default.

The Chunkify dashboard needs `https://chunkify.dev` to upload or preview this media. Add it when the user asks for dashboard uploads or previews. Otherwise, ask once while presenting the policy for approval, and add it only if the user agrees.

For S3, a combined console-format example is:

```json
[
  {
    "AllowedOrigins": [
      "http://localhost:3000",
      "https://app.example.com"
    ],
    "AllowedMethods": ["GET", "HEAD", "PUT"],
    "AllowedHeaders": ["Range", "Content-Type"],
    "ExposeHeaders": ["Content-Length", "Content-Range", "Accept-Ranges", "ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

Replace example origins with the real scheme, host, and port. Remove unused origins and methods. Add other request headers only when the browser's preflight requires them. Use separate rules when upload and playback origins differ. The S3 API and CLI wrap this array in `CORSRules`; S3 handles preflight, so do not add `OPTIONS` to `AllowedMethods`. Other providers use different policy formats.

Preserve unrelated existing rules and show the full merged policy and exact change before replacing bucket CORS. Ask approval for that replacement unless the user already authorized it. If provider tools are unavailable, give the approved policy and precise manual steps. Check CORS at the CDN edge and storage origin. Verify a browser PUT preflight, HLS playlists and segments, poster, timeline WebVTT and sprite images, and `Range` responses from the application's origin and CDN hostname.
