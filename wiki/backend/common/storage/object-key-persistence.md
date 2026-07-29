---
id: backend-common-storage-object-key-persistence
domain: backend
category: storage
applies_to: [aws-s3]
confidence: verified
sources:
  - https://github.com/aws/aws-sdk-js/issues/1158
  - https://github.com/aws/aws-sdk-js-v3/issues/5656
  - https://github.com/aws/aws-sdk-php/issues/2933
  - https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html
last_verified: 2026-07-29
related: [backend-common-jobs-idempotent-handlers, backend-node-boundaries-runtime-validation]
---

# Persisting the Result of a Managed Object-Storage Upload

## When this applies

You are writing the result of a high-level ("managed") S3 upload to a database,
message, or any durable store — `s3.upload()` in aws-sdk-js v2, `Upload` from
`@aws-sdk/lib-storage` in v3, `S3Client::upload()` in aws-sdk-php — and choosing
which field of the result becomes the stored reference.

## Do this

1. **Store `Bucket` + `Key`, and derive URLs at read time.** Both upload paths
   return `Key` raw and unencoded: the single-part path sets it from the request
   params, and the multipart path receives it in the `CompleteMultipartUpload`
   response shape. `Key` is the one field whose value does not depend on which
   path ran.

2. **Treat `Location` as display output, not as an identifier.** Its provenance
   switches at the multipart threshold, and the two sources encode differently:

| Upload path | Where `Location` comes from | Encoding of a key containing a space |
|-------------|------------------------------|--------------------------------------|
| Single part (body ≤ `partSize`) | The SDK composes it from the signed request: protocol + host + the escaped request path | Percent-encoded — `%20` |
| Multipart (body > `partSize`) | S3's `CompleteMultipartUpload` XML response, passed through | Form-encoded — `+` for space (aws-sdk-js v2 rewrites `%2F` back to `/`, and nothing else) |

3. **Build the read path from the stored key with the same escaping the SDK
   uses for requests.** Pass `Key` to `getObject`/`getSignedUrl` and let the SDK
   escape it; a URL string that was stored and later re-parsed carries whichever
   encoding its upload path produced.

4. **When a column already holds stored URLs, decode by provenance, not by
   guess.** Rows written through the multipart path need form decoding (`+` is a
   space); rows written through the single-part path need standard percent
   decoding (`+` is a literal plus). Re-deriving each row's key from a bucket
   listing removes the ambiguity for good.

## Edge cases

| Case | Then |
|------|------|
| Keys in this bucket never contain a space, `+`, or non-ASCII character | The two encodings coincide today; store `Key` anyway — the divergence appears the first time a user-supplied filename reaches the key, with no code change to blame |
| `partSize` is set explicitly in the upload options | The boundary moves to that value; aws-sdk-js v2 also raises `partSize` on its own when body size ÷ `maxTotalParts` exceeds it, so the switch point is not a constant you can assert against |
| An existing column mixes both encodings and you cannot re-derive keys | Detect per row: a key that resolves under form decoding but 404s under percent decoding was written by the multipart path; record the verdict rather than re-detecting on every read |
| You need a stable public URL in the response body | Compose it at serialization time from bucket + stored key + your own escaping, so one encoding rule owns every URL your API emits |
| The upload runs against S3-compatible storage (MinIO, SeaweedFS, Storj) | Confirm what its `CompleteMultipartUpload` returns for `Location` before relying on it — implementations differ, including omitting it; `Key` remains the portable field |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Save `uploadRes.Location` as the object reference | Save `uploadRes.Key` (with `Bucket`) and derive URLs on read | `Location`'s encoding depends on whether the upload went multipart, so one column ends up holding two encodings |
| Verify the upload-and-fetch round trip with a small test file | Use a file larger than `partSize` as well | Under the threshold only the single-part path runs, so the multipart encoding never appears in the test |
| Normalize stored URLs with a single `decodeURIComponent` pass | Decode by the provenance of each row, or re-derive keys from a listing | `decodeURIComponent` leaves `+` as a literal plus, so multipart-written rows keep a corrupted key and 404 |

## Sources

- https://github.com/aws/aws-sdk-js/issues/1158 — `ManagedUpload` returns a URI-encoded `Location` for multipart uploads and an unencoded one for single-part; the multipart value comes from the S3 response while the SDK builds the single-part value itself. PR #1420 rewrites only `%2F` → `/`
- https://github.com/aws/aws-sdk-js-v3/issues/5656 — the same divergence in `@aws-sdk/lib-storage`: `__uploadUsingPut` composes `Location` for single-part uploads, and the `CompleteMultipartUploadCommand` path does not
- https://github.com/aws/aws-sdk-php/issues/2933 — the same divergence in aws-sdk-php: `ObjectURL` derives from the request's effective URI for regular uploads and from the server response past the multipart threshold, giving inconsistent encoding of spaces and plus signs
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html — the response includes `Key` alongside `Location`
- Reproducible check against aws-sdk-js 2.829.0: `lib/s3/managed_upload.js` — `minPartSize: 1024 * 1024 * 5`; `finishSinglePart` composes `data.Location` from `endpoint.protocol + '//' + endpoint.host + httpReq.path` and sets `data.Key` from `request.params.Key`; `finishMultiPart` applies `data.Location.replace(/%2F/g, '/')` to S3's value. `lib/util.js` `uriEscape` wraps `encodeURIComponent`, so the request path carries `%20`. `apis/s3-2006-03-01.min.json` lists `Key` in the `CompleteMultipartUpload` output shape
- Field context: a production regression where files larger than 5 MB 404'd on download while smaller ones worked, because stored `Location` values from the multipart path held `+` where the key had a space
