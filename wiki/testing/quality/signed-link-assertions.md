---
id: testing-quality-signed-link-assertions
domain: testing
category: quality
applies_to: [general]
confidence: verified
sources:
  - https://docs.djangoproject.com/en/stable/topics/signing/
  - https://www.rfc-editor.org/rfc/rfc7519
  - https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
  - https://stryker-mutator.io/docs/mutation-testing-elements/supported-mutators/
last_verified: 2026-08-11
related:
  [
    testing-quality-tests-that-cannot-fail,
    testing-quality-write-path-assertions,
    testing-quality-harness-reverse-controls,
    backend-common-auth-jwt-server-side,
    security-secrets-secrets-in-code,
  ]
---

# Testing Code That Mints a Signed Link

## When this applies

The code under test assembles a URL that carries a signature or token — an
approval/action link, a presigned object-storage URL, a magic sign-in link, a
webhook callback — and you are choosing what the test asserts about that
string. Also when such a test is green while the link is signed with the wrong
key, for the wrong subject, or by a stubbed signer that never ran.

## Do this

1. **Assert by verifying, not by matching.** Pass the token from the assembled
   link to the same verifier the receiver runs, with the correct key and the
   correct subject, and require success. A pattern assertion (`?t=` is present,
   the token matches `[A-Za-z0-9_-]{43}`) is satisfied by any constant of the
   right shape, including a hardcoded fake and a signature over the wrong data.

2. **Add the negative arms that keep the shape intact**, in the same test, one
   per binding the signature carries. Each arm changes one bound element and
   requires rejection:

| The link is bound to | Negative arm |
|----------------------|--------------|
| A signing key | Verify with a second key generated in the fixture; require the verifier's rejection |
| A subject/resource id inside the payload | Verify against a different id; require rejection |
| A purpose or namespace (Django `salt`, JWT `aud`) | Verify under a different purpose; require rejection — "A signature that comes from one namespace (a particular salt value) cannot be used to validate the same plaintext string in a different namespace" |
| An expiry | Advance the injected clock past the expiry; require rejection — `exp` "identifies the expiration time on or after which the JWT MUST NOT be accepted for processing" |
| Request elements the provider signs (method, query string) | Replay with one element altered; require the provider's rejection (`SignatureDoesNotMatch`) |

3. **Where the receiver runs in-process, send the assembled link to it.** Use
   the framework's test client or a local server, `GET`/`POST` the link exactly
   as built, and assert the receiver's success status. In the same test, send
   the same path with the signature parameter removed and assert rejection —
   that pair is what proves the link is both routable and gated
   ([testing-quality-write-path-assertions] for asserting past the status code).

4. **Seed this class of mutation by hand.** Generators do not produce it:
   Stryker's nearest string operator is `FilledStringToEmpty`, `"foo"` becomes
   `""` — and an empty key still yields a correctly shaped signature, so even
   the generated mutant survives a pattern assertion. Seed three edits, and
   require the verification assertion (not the pattern assertion) to redden on
   each: sign with another key, sign for another subject, drop the signature
   parameter.

5. **Run a semantics-preserving control in the same session and require green**
   — reorder unsigned query parameters, or reformat the assembly code. A green
   control separates "the test reads the signature" from "the test reads the
   string layout" ([testing-quality-harness-reverse-controls]).

## Edge cases

| Case | Then |
|------|------|
| The receiver is a third party (object storage, a payment provider) | The negative arms move to the provider: issue the URL, alter one signed element, and assert the provider's error. AWS instructs verifying "that all request parameters—including the HTTP method, headers, and query string—match exactly between URL generation and usage" |
| The token is opaque — a random id the receiver looks up, not a MAC | The wrong-key arm has no meaning; the arms become "an id that was never issued is rejected" and "an id issued for another subject is rejected" |
| The signer is a library you do not own | Assert the arguments reaching it (key id, subject, expiry) from the captured call, and keep one end-to-end arm through the real signer |
| The signing key in the fixture is the production key | Generate the key inside the fixture and inject it, so the test does not carry a secret ([security-secrets-secrets-in-code]) |
| The URL's lifetime is bounded by the credential rather than the parameter | Assert rejection after expiry, not an absolute lifetime — a presigned URL "expires when the credential you used to create it is revoked, deleted, or deactivated … even if the URL was created with a later expiration time" |
| The link is assembled in one module and signed in another | Assert on the string the consumer receives, not on the signer's return value — the defect this page covers is the assembly dropping or re-encoding the parameter |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Assert the built URL matches a regex containing the token parameter | Pass the token to the verifier and require success, plus one wrong-key arm | A hardcoded constant of the right shape satisfies the regex; only the verifier reads the key |
| Assert the built URL equals a golden string committed in the test | Recompute the expectation through the verifier | The golden pins the key and payload of the day it was written, so rotating either reddens correct code |
| Assert only that the signature parameter is present | Add the arm that removes it and requires the receiver to reject | Presence proves assembly ran; rejection-on-absence proves the receiver enforces it |
| Stub the signer and assert the stub was called | Run the real signer with a fixture key and verify its output | A called stub proves the call site exists, not that its output verifies |

## Sources

- https://docs.djangoproject.com/en/stable/topics/signing/ — "Cryptographically signed values can be passed through an untrusted channel safe in the knowledge that any tampering will be detected"; "If the signature or value have been altered in any way, a `django.core.signing.BadSignature` exception will be raised"; salt namespacing: "A signature that comes from one namespace (a particular salt value) cannot be used to validate the same plaintext string in a different namespace that is using a different salt setting", which is what the wrong-purpose arm exercises
- https://www.rfc-editor.org/rfc/rfc7519 — `aud`: "If the principal processing the claim does not identify itself with a value in the 'aud' claim when this claim is present, then the JWT MUST be rejected"; `exp`: "identifies the expiration time on or after which the JWT MUST NOT be accepted for processing"
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html — "verify that all request parameters—including the HTTP method, headers, and query string—match exactly between URL generation and usage" as the cause of `SignatureDoesNotMatch`; "a presigned URL expires when the credential you used to create it is revoked, deleted, or deactivated. This is true even if the URL was created with a later expiration time"
- https://stryker-mutator.io/docs/mutation-testing-elements/supported-mutators/ — String Literal: "FilledStringToEmpty: `\"foo\"` (filled string) becomes `\"\"` (empty string)" — the nearest generated operator, which leaves a well-formed signature and so does not exercise this class
- Field reproduction 2026-08-11 (Python link-assembly module, six hand-seeded mutants): the wrong-key and wrong-subject mutants left the `?t=` parameter structurally intact, so every pattern assertion stayed green; only the verifier-call assertion and three live-receiver `200` assertions reddened, and the no-op control survived
