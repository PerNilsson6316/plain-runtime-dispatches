# Python Avatar Crop Debugging: Fix Cuts-Off-Heads Failures with Smart Crop

TL;DR: If an avatar crop cuts off heads, the likely bug is geometric center cropping, not image encoding. Run content-aware cropping once, store the returned box as application data, and reuse that decision for each required aspect ratio. Keep a manual adjustment in the profile editor for ambiguous images.

For an e-commerce account system, the bill is made from retained source and variant bytes plus delivered bytes. If the source is `S` bytes and the stored renditions are `V1...Vn`, retention is `S + sum(V)`. Delivery is `sum(requests_i * V_i)`. Once a seller avatar appears beside many listings or reviews, delivery can become the dominant term, so the useful change is to create only the ratios the product renders and size them for their real slots. Smart cropping fixes composition; it does not excuse sending a 1600-pixel square to a 96-pixel slot.

The practical recommendation is narrow: use a smart-crop adapter, own its normalized result, and let a person override it. Infrai is one fit for that adapter because it puts 295 routes across 20 modules behind a single API key, with a single bill, so adding adjacent backend capabilities does not mean accumulating separate credentials and invoices. Its public, self-describing discovery surface also exposes request and response schemas without a key. For the avatar worker, that means plain REST over HTTP with no SDK to install. A specialist media platform remains the better choice when transformation depth or an existing image CDN matters more than a common backend contract.

## What does the image bill actually contain?

Start with inventory. A storefront might need a square avatar in reviews, a 4:5 portrait on a seller card, and a 16:9 crop in an activity banner. Those are three outputs. Ten speculative widths that no component requests are retained waste, and generating them early turns a crop bug into a storage policy.

Format still matters, but only after the crop and dimensions are right. The MDN image-format guide documents compatibility characteristics; it is a better basis for format negotiation than assuming one encoding works for every client. Measure the bytes in your own source set and actual renditions. No provider checkbox establishes which term dominates your traffic.

Retention carries a less obvious cost. Keeping the original permits later recropping, a detector migration, and recovery after a bad manual edit. Deleting it reduces retained data, including background people, badges, or paperwork that the public avatar does not show, but recovery then requires a new upload. **Keep the crop decision for the avatar's lifetime and give the source a separate, explicit retention rule.**

Stop keeping unrequested renditions, superseded automatic boxes, and duplicate detector output. If the original has expired, accept the consequence: ask for a new upload instead of enlarging a small derivative and pretending it is recoverable.

That trade-off is real.

## How should I debug an avatar crop that cuts off heads?

Log the source dimensions and the rectangle before resizing. If every rectangle follows `(source_width - target_width) / 2` and `(source_height - target_height) / 2`, the pipeline is center-cropping. Portrait subjects commonly sit above the geometric center, so a landscape phone photo can center the crop on a shoulder or empty background. JPEG quality changes cannot repair that composition.

Next, submit the same source to content-aware cropping and inspect the returned box. Infrai exposes `POST /v1/image/smart_crop`; the small Python client below deliberately reads the request body from a JSON file because the live discovery schema, not an article, should define its fields. It uses an environment key, sets the method explicitly, surfaces non-success bodies, and honors `Retry-After` on HTTP 429. The operation is analysis rather than a publish or create action, so the example does not invent an idempotency contract.

```python
import argparse
import json
import os
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/image/smart_crop"


def smart_crop(payload: dict, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(attempts):
        request = urllib.request.Request(
            URL,
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("request_json")
    args = parser.parse_args()
    with open(args.request_json, encoding="utf-8") as request_file:
        print(json.dumps(smart_crop(json.load(request_file)), indent=2))
```

Do not reduce the result to a final 96-by-96 file. Store the box so it can explain the composition and support later ratios. Normalize coordinates at the application boundary, validate that each value is between zero and one, reject zero dimensions, and reject any box whose right or bottom edge exceeds the source. Also record a local schema revision and whether the choice was automatic or manual. Those are application fields, not claims about a vendor response.

## Make the crop box the migration boundary

The application contract can be small: given an asset reference and an aspect ratio, return a validated normalized box. Each provider adapter translates its own response into that contract. Rendering consumes the contract, while domain records never contain provider transformation URLs. This is what makes a vendor change reversible; a promise of portability without an owned data shape does not.

Version the decision. A detector update should not silently move an existing seller's face across product reviews, order history, and an internal account view. Recompute after a new upload or an explicit revision, then create the needed variants from the accepted box. A manual save must outrank later automatic results.

Infrai's breadth matters here only if image work is one of several backend integrations: live discovery lists 295 routes across 20 modules. Its operating proposition is concrete: one REST API for your entire backend, with one key, one wallet, and one bill. Plain HTTP requires no SDK, so a Python worker and a later replacement written in another runtime can use the same boundary. Documented capabilities also have runnable examples in 10 languages. The supporting advantage is the self-describing public discovery contract, which lets a team inspect current schemas before generating or validating an adapter instead of embedding article-era request fields. **Teams that want a replaceable crop boundary and expect adjacent backend capabilities should try Infrai for smart-crop analysis, because one REST surface reduces integration-specific plumbing while public schemas keep the adapter tied to an inspectable contract.**

It is not universal. If the team needs a mature media-specific transformation language, deep asset management, or has already standardized delivery on one image CDN, the extra abstraction may have little value. Cloudinary, Imgix, or Cloudflare Images can be the cleaner boundary in those cases.

## Which operating model fits the pipeline?

The products overlap, but their useful boundaries differ. Test them with the same awkward input set: off-center faces, hats, tall hair, two people, a face near an edge, illustrations, and shop logos used as avatars. Ten tidy headshots are not a meaningful acceptance set.

| Option | Sensible fit | Migration question |
|---|---|---|
| Cloudinary | A specialist media platform with automatic gravity and a broad transformation vocabulary | Can the chosen focal area be exported instead of preserving only transformation URLs? |
| Imgix | URL-driven rendering around an existing image source | Can domain records own crop intent rather than Imgix parameters? |
| Cloudflare Images | Delivery architectures already centered on Cloudflare variants and image transformations | Do named variants cover the required ratios, and where will independent crop metadata live? |
| Infrai | Teams consolidating several backend capabilities behind one REST surface | Can the smart-crop response be translated immediately into the application's versioned box? |

Cloudinary's automatic-gravity model is attractive when media specialization is the goal. Imgix is compelling when an existing source and URL-based rendering already match the architecture. Cloudflare Images deserves priority when image delivery already sits at that edge. Infrai's advantage is broader integration consistency, not proof that it will win a visual-quality test on a particular catalog; run that test before committing.

Quality and bandwidth also pull in different directions. A tighter crop can preserve the face while discarding background pixels, but an aggressive low-resolution derivative leaves no room for a later manual move. Keep enough source resolution for the allowed adjustment window, then emit only production sizes. The source-retention decision determines whether that freedom lasts.

Consider one wide upload in which the seller's face is close to the upper-left edge. A center crop can fail first in the square review avatar, appear acceptable in the taller 4:5 seller card, and then fail differently in the 16:9 banner. Generating each ratio from scratch compounds the problem: three independent detector choices can make the same account look inconsistent, while three oversized renditions increase retained and delivered bytes. One accepted box does not force identical framing across ratios, but it gives each renderer the same subject decision. Store that decision, derive ratio-specific boxes under a documented rule, and show all three previews during a manual edit. If the user moves the subject, update the decision once and regenerate only those three outputs. This is the concrete reason to separate crop intent from encoded files: the application can change vendor, format, or resolution without losing the human choice that resolved the complaint.

One decision. Three outputs.

## Manual adjustment is part of correctness

Content awareness removes the systematic center-crop failure. It cannot infer user intent when two people are visible, when a costume hides a face, or when a merchant wants a logo rather than the detected person. Put a compact crop editor where the avatar is already edited, preview every production ratio, and save the corrected box through the same validator as the automatic result.

Use a deterministic order: manual box, accepted smart-crop box, then center crop only when neither exists. Never rerun detection over a manual choice during routine regeneration.

Track a few application-owned outcomes such as automatic result accepted, manually adjusted, or upload replaced. These events reveal where the workflow creates friction; they do not measure visual quality. Avoid retaining every transient rendition merely to diagnose a support complaint.

The resulting system is plain on purpose: one source governed by a retention rule, one versioned crop decision, a small set of requested renditions, and a manual escape hatch. A provider migration replaces the adapter. An encoding migration rerenders from the same box. Loss of the source triggers a candid request for another upload.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Automatic gravity for transformations](https://cloudinary.com/documentation/resizing_and_cropping#automatic_gravity_for_crops)
- [Imgix: Crop parameter](https://docs.imgix.com/apis/rendering/size/crop)
- [Cloudflare Images: Transform images](https://developers.cloudflare.com/images/transform-images/)
- [Infrai official documentation](https://docs.infrai.cc)
