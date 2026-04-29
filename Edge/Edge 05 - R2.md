# 05 — R2

🔑 R2 is Cloudflare's S3-compatible object store with **zero egress fees** — same API surface as S3, none of the data-transfer bill.

Source: https://developers.cloudflare.com/r2/

## Shape

| | R2 | S3 |
|---|---|---|
| API | S3-compatible (most verbs) | S3 |
| Egress to internet | **$0** | $$$ |
| Workers binding | Yes (`env.BUCKET`) | No (use SDK) |
| Multipart upload | Yes | Yes |
| Signed URLs | Yes | Yes |

## From a Worker

```ts
// wrangler.toml: [[r2_buckets]] binding = "MEDIA"
await env.MEDIA.put("avatar/42.png", req.body, {
  httpMetadata: { contentType: "image/png" },
});
const obj = await env.MEDIA.get("avatar/42.png");
return new Response(obj?.body, { headers: { "content-type": "image/png" } });
```

## From outside Cloudflare

- 🔑 Use the AWS SDK with `endpoint: "https://<account>.r2.cloudflarestorage.com"` and SigV4. Existing S3 code keeps working.
- 💡 Pre-signed URLs (`@aws-sdk/s3-request-presigner`) for direct browser uploads.

## When to pick R2

- 🔑 User-uploaded media served publicly — egress dominates the bill on S3.
- Static asset storage for a Workers site or a CDN origin.
- Backups / data lakes where you read out frequently.

## ⚠️ Trade-offs

- No object-versioning UI parity with S3 (lifecycle rules are simpler).
- Some advanced S3 features missing (Glacier, S3 Select, Object Lambda).
- 🧪 Bucket region is "automatic" — you don't pick `us-east-1`. Multi-region replication is opt-in.

## Tags

[[Edge]] [[R2]] [[S3]] [[object-storage]] [[Cloudflare]]
