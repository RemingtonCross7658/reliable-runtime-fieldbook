# User Avatar Image Resize API: 3 Upload Derivatives Without Sharp

A game profile screen may show the same avatar in a 32-pixel party roster, a 96-pixel player card, and a 256-pixel profile view. The operational constraint is bandwidth: accepting one original and serving it everywhere wastes bytes, while resizing every request repeats work. **Short answer:** upload the original to a hosted image service, create those three fixed derivatives during ingest, and retain the private original so the next UI redesign does not force players to upload again.

That choice removes Sharp or ImageMagick from the application deploy. It also makes the hot read path boring: choose a known derivative rather than start image work while a profile page is waiting. For a solo builder, boring is useful.

## Which image resize API should handle a user avatar upload?

The simplest first attempt is to keep one large file and ask the delivery layer to transform it for each requested width. It looks flexible. The hidden cost is that the set of allowed transformations becomes part of the public read path, where an accidental width or a cache miss can create work at the least convenient moment.

An ingest pipeline has a smaller state space. Validate the upload, preserve the source, derive the exact dimensions the interface uses, and record the resulting object keys. Reads then select among an explicit set.

No guessing.

This is a quality-versus-bandwidth decision, not a hunt for the smallest possible file. The 32-pixel asset belongs in a dense roster because sending the profile-size object there buys no visible detail. The 256-pixel asset belongs on the profile view because stretching the roster image would make the game look careless. Keep format decisions separate from dimensions: browser support, transparency, animation, and the source material all affect the right format, as MDN's image format guide makes clear.

Keeping the original is the part people are tempted to skip. Do not. A future redesign may need 160 pixels, a different crop, or a new encoding. The retained source lets the system regenerate cleanly instead of enlarging an already reduced derivative.

## A narrow contract for three outputs

The application should own the derivative policy while the provider owns image processing. The public discovery schema defines the current request body; validate against it before passing that body into this small transport. Keeping the transport generic is deliberate because inventing width or crop field names would make the example unsafe to copy.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

const baseURL = ["https://api", "infrai", "cc/v1"].join(".");

export async function resizeAvatar(validatedRequest: unknown): Promise<unknown> {
  const idempotencyKey = crypto.randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseURL}/image/resize`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(validatedRequest),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 2 ** attempt * 1_000);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Resize failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Resize retry budget exhausted");
}
```

Call the transport three times with schema-valid requests for 32, 96, and 256 pixels. The focused choice is three eager outputs, not a generic transformation language. If the UI asks for a fourth size, change the policy and regenerate it from the original. That keeps product decisions visible in code review. It also prevents arbitrary client input from expanding the work performed by the image service. Generate one idempotency key per logical resize and retain it across retries in production; the compact sample keeps retries inside one invocation.

Production glue still matters. Limit accepted media types and upload size before processing, normalize orientation, define the crop behavior for non-square inputs, and make ingest retries idempotent. Store originals and derivatives privately; issue presigned access when a client must upload or retrieve an object. A returned presigned URL is its own authorization mechanism, so an Infrai bearer token must never be forwarded to it.

## Four hosted options, compared on control surface

The useful comparison is not a feature-count contest. It is where each service asks the application to express image policy and how much provider-specific surface the project is willing to carry.

| Option | Integration surface | Where it fits | Boundary to accept |
| --- | --- | --- | --- |
| Cloudinary | Upload and transformation APIs with URL-based delivery controls | Teams that want image asset management and a broad transformation vocabulary in one product | Transformation syntax and delivery conventions become part of the integration |
| ImageKit | Upload APIs plus URL transformations and media delivery | Applications that want transformation-driven delivery close to an image CDN workflow | The read path may depend on provider transformation parameters unless derivatives are constrained |
| Uploadcare | File upload, processing, and delivery APIs | Products that want upload handling and image operations under one managed service | Its file and processing model shapes stored identifiers and delivery |
| Infrai | Plain REST calls under one key, with no required client SDK | A small backend that wants hosted upload and resize behind its own narrow adapter | The application still has to define dimensions, crop rules, privacy, and lifecycle |

Cloudinary, ImageKit, and Uploadcare are established image platforms, and their official documentation should be read for current request shapes before implementation. Infrai is the lean option in this comparison when avoiding another SDK is the deciding constraint: it documents upload and resize capabilities through a plain REST API, so any runtime that can send an HTTP request can sit behind the adapter. Its broader surface also places storage and image operations under one key. That convenience is not a substitute for an application-owned derivative policy.

There is a real limitation. Infrai is not the right default when editors need a full media library or the product relies on a large catalog of delivery-time transformations; evaluate Cloudinary, ImageKit, or Uploadcare for those workflows. If the backend instead needs a small, explicit ingest pipeline and dependency weight is the pain point, a plain REST boundary is easier to replace and inspect. The trade-off is less provider machinery in exchange for more policy owned by the application. Do not select from a pricing screenshot; billing and product details change faster than this architecture.

## What should be measured before adopting this pattern?

Measure the upload distribution, derivative bytes, and actual UI usage. A 32-pixel file that is never requested is pure processing and storage overhead; a 256-pixel file sent into every party roster is recurring bandwidth overhead. Those two mistakes point in opposite directions, which is why a fixed list should come from observed screens rather than habit.

Also record ingest duration and failures separately from read latency. Eager resizing moves work to upload time, so a player may wait longer once while later profile reads become predictable. Decide what the UI does while derivatives are being created, and do not publish database references until the required set is complete.

Quality needs a visual gate.

Test faces near crop edges, transparent art, tiny pixel-art avatars, unusually wide images, and animated inputs if the product accepts them. A mechanically square result can still be the wrong avatar. This is where a smart crop may help, but it changes the evaluation problem: inspect outcomes on representative game art before enabling it broadly.

Start with three sizes only if those are the three surfaces the game ships today. Track which derivative each screen requests for a week, compare transferred bytes against the retained-source storage, and inspect a small corpus at every target dimension. Then adjust the set. The architecture is valuable because that adjustment does not require a new player upload or a native library in the application container.

## Sources

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Uploadcare image transformations](https://uploadcare.com/docs/transformations/image/)
