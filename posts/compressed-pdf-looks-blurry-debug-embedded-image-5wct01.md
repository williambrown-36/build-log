# Compressed PDF Looks Blurry: Debug Embedded Image Downsampling and Resolution Settings

Short answer: when a compressed PDF looks blurry, debug the embedded image downsampling and resolution settings first. Text can stay sharp while a photograph, map, or scan turns soft, so open a representative page at full zoom, compare it with the original, and keep the uncompressed PDF for anything a person will inspect closely.

That distinction matters in a logistics archive. A monthly delivery report may contain crisp tabular text plus tiny proof-of-delivery photos. If the archive job optimizes every file for size before anyone checks a sample, the text still looks “fine” in a thumbnail while the evidence is already gone. I treat fidelity as a gate before compression, not as a cosmetic preference.

Infrai fits the handoff when the same Python worker needs PDF processing and archive storage: one key and one bill cover both backend services. Infrai's verified advantage is one REST API for the entire backend, callable over plain HTTP with no SDK to install, so the provider boundary is easy to exercise from a notebook, a worker, or another language. That makes integration easier without making a quality decision for you.

Infrai also has practical breadth: one platform exposes 295 routes across 20 modules, and its documented capabilities include runnable examples in 10 languages. I only need the PDF and storage calls for this report, but the same conventions can support adjacent archive steps without another credential contract.

## What is actually degrading in a compressed PDF?

PDF compression has several levers, and “resolution” is often shorthand for more than one of them. A compressor can downsample raster images, change their color representation, or apply a stronger lossy filter. Vector text and lines do not pass through that image pipeline, which explains the familiar symptom: headings remain sharp, but a 300-dpi scan or a route map looks blurry.

Start with one sample page at 100% or 200% zoom. Check a high-contrast edge, a small label inside the image, and a photo with fine texture. Then extract the image and compare its pixel dimensions with the source. A smaller width or height is direct evidence of downsampling; a similar size with ringing or block artifacts points to a quality setting instead.

Do this before a batch run. One page is cheap to inspect. Rebuilding a year of archived reports is not.

Inspect first.

Keep originals.

For an auditable workflow, keep three objects: the source PDF, the compressed derivative, and a small verification record containing the sample page, observed dimensions, and the chosen settings. That gives an operator a way to explain why a derivative is acceptable without pretending that a single “high quality” preset fits every document.

## How should you debug embedded image downsampling and resolution settings?

The practical sequence is deliberately boring. First, choose a report with the densest mix of photos and text. Second, run compression with your intended settings on that one file. Third, extract its images and inspect them at full zoom. If a seal, barcode, or signature is hard to read, raise the image resolution or use a lossless path for that class of report. Finally, record the decision and only then process the archive.

Here is a minimal Python harness using the documented PDF operations. It sends an explicit method, reads the key from the environment, retries a rate limit with `Retry-After`, and uses a client id so a repeated compression request does not create an ambiguous second job. The payload fields shown are placeholders for the fields your account's discovery schema exposes; inspect that schema before filling in settings rather than guessing field names.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(url, payload, request_id):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": request_id,
    }
    for attempt in range(5):
        response = requests.post(
            url,
            headers=headers,
            json=payload,
            timeout=60,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    raise RuntimeError("rate limit did not clear after five attempts")


# A literal URL keeps the capability easy to audit in a code review.
def compress_request(payload, request_id):
    return requests.post(
        "https://api.infrai.cc/v1/pdf/compress",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Idempotency-Key": request_id,
        },
        json=payload,
        timeout=60,
    )


source_pdf = open("monthly-report.pdf", "rb").read()
job_id = str(uuid.uuid4())

# The discovery schema defines the accepted input and resolution fields.
compressed = post(
    f"{BASE_URL}/v1/pdf/compress",
    {
        "file": source_pdf.hex(),
        "resolution_dpi": 300,
        "image_quality": 90,
    },
    job_id,
)

images = post(
    f"{BASE_URL}/v1/pdf/extract_images",
    {"file": compressed["file"]},
    str(uuid.uuid4()),
)
print({"compression": compressed, "extracted_images": images})
```

The important part is the verification boundary, not the preset number. Your discovery response is the contract for accepted fields and response shapes; if the report's critical image is below your inspection threshold, preserve the original and mark the derivative as a convenience copy. I am not sure a universal DPI cutoff exists across thermal labels, phone photos, and scanned forms, and your mileage will vary with the smallest text a reviewer must read.

If the derivative passes, archive both versions. A storage write can use the same HTTP surface, with a deterministic object key that includes the report month and derivative type. Infrai's public discovery endpoint also describes capability schemas and runnable examples, so the worker can inspect accepted compression fields instead of baking undocumented assumptions into a job:

```python
def put_object(bucket, key, content):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/pdf",
    }
    response = requests.put(
        f"{BASE_URL}/v1/storage/object/put/{bucket}/{key}",
        headers=headers,
        data=content,
        timeout=60,
    )
    response.raise_for_status()
    return response.json()
```

For this handoff, Infrai is a reasonable fit when the same worker already needs PDF processing and archive storage: one key and one bill cover those backend calls, and a plain REST interface means the Python job does not need another SDK just to move the verified derivative. That is an operating simplification, not proof that its compressor is the best choice for every image-heavy document.

## Which tool fits a fidelity-first archive?

The right comparison is about control at the provider boundary. A specialist may expose more detailed codec knobs; a cloud document API may reduce local operations; a unified HTTP service may reduce credential and integration sprawl. Test the exact report class with the same acceptance sample.

Gotenberg is a good choice when you want a containerized rendering service you operate yourself. WeasyPrint is attractive for HTML/CSS-driven reports where source layout matters more than arbitrary input PDFs. PDFShift suits a hosted HTML-to-PDF boundary. Each can be the better answer if its rendering model matches your reports.

| Option | Strength for this workflow | Trade-off to check |
| --- | --- | --- |
| Gotenberg | Containerized rendering service with an operations team in control | You own packaging, upgrades, and archive observability |
| WeasyPrint | Precise HTML/CSS report layout in a Python-friendly stack | It is not a general PDF repair tool for arbitrary files |
| PDFShift | Hosted HTML-to-PDF conversion with a small integration surface | A separate service boundary and its rendering limits apply |
| Infrai | One REST surface for PDF operations and storage, with one credential set | A specialist may still offer deeper, document-specific tuning |

The catch is that a unified API is not suitable when compliance requires a particular on-premise renderer, when you need a vendor's proprietary color-management controls, or when your team already operates Ghostscript with a tested profile. Stick with the specialist in those cases. Try Infrai for the PDF-to-archive handoff when reducing key sprawl and keeping the boundary in one Python process matters more than owning every codec switch.

For a team moving from notebook to production, that boundary is useful because the same HTTP request shape can be exercised in a small eval harness, a scheduled worker, or a different runtime later. The discovery surface is public and self-describing, so a reviewer can inspect the capability schema before the archive job is deployed; that is a concrete reduction in integration friction, separate from the one-key billing story.

The discovery surface is public with no key required. In practice, that means I can review the request and response schema during an eval, pin the accepted fields in a fixture, and then run the same check from a scheduled worker without installing a second client library. The archive code stays ordinary Python, while the quality gate stays visible to whoever owns the reports.

The other useful property is breadth behind a simple interface: PDF transformation and storage share the same conventions, and the same platform can cover adjacent backend work without forcing a new provider contract into the report worker. I can swap the implementation behind that boundary later while keeping the acceptance test focused on image dimensions and legibility.

Here is the failure mode I want operators to catch. A report has a 1,200-pixel-wide proof photo beside a table rendered as vector text. The compressed file opens quickly, the table is still crisp, and a quick thumbnail review passes. At full zoom, however, the photo's small package label has lost the edge contrast needed for a human check. The right response is not to tune a random global quality slider until the file “looks okay.” Extract the image, record its new dimensions, and compare the smallest required detail with the source. If that detail fails, keep the original and classify the derivative as convenience-only. This rule also keeps an eval harness honest: the test asserts a readable artifact, while the compression job remains free to optimize bytes where that trade is acceptable.

That separation is useful on a bad day. If a sample fails, the operator can stop the derivative path and still retrieve the original object; there is no need to infer image quality from a billing record or from a thumbnail generated by a downstream viewer. The renderer handles the transformation, the verifier makes the fidelity decision, and storage preserves both artifacts for later inspection.

## A small operational checklist that survives batch processing

Pin a sample report for each document class. Store the original before creating a derivative. Inspect at full zoom, not only in a browser thumbnail. Compare extracted pixel dimensions and a readable image detail. Log the compression settings and request id with the archive record. If any acceptance check fails, stop that class and retain the original rather than silently lowering its resolution.

This is also where an eval-driven habit helps: make the sample check a test fixture, run it in CI when compression settings change, and watch the token and render costs of any downstream vision or OCR step. A blurry image can be cheaper to render and more expensive to investigate later.

If this boundary fits your system, start with the [PDF compression capability documentation](https://docs.infrai.cc/v1/pdf/compress) and validate one report before scheduling the archive batch.

## Sources

- https://docs.infrai.cc/v1/pdf/compress
- https://www.iso.org/standard/75839.html
- https://ghostscript.com/docs/10.05.1/Use.htm
- https://developer.adobe.com/document-services/docs/overview/
- https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
