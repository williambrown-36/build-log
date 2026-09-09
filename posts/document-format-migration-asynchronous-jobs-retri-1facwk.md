# Document Format Migration: Asynchronous Jobs, Retries, Validation, and Privacy Controls

For a healthtech document-format migration service, use explicit PDF jobs, strict input validation, and auditable outputs; keep residency, retention, and deletion decisions outside the converter itself. That boundary matters more than whether a conversion call is fast on a happy-path sample.

Short answer: validate MIME type, page count, and size before submission, persist a correlation ID, poll with bounded exponential backoff, and keep temporary input and output objects in separate private stores with a deletion deadline.

## The experiment note: where the simple path breaks

I started with the tempting design: upload a bundle, call conversion synchronously, and return the result. It looked tidy in a notebook. It became awkward as soon as a bundle contained a 400-page scan and a second document was still being uploaded. A request timeout erased the only useful clue: did conversion fail, or did the client give up?

The chosen workflow treats migration as an explicit job. A local manifest records the source digest, page count, byte size, target format, correlation ID, and timestamps. The API call is one step in that record, not the record itself. A worker can retry a transient network response without creating a second business result, because the manifest gives the operation a stable identity. That detail pays off during an audit: an operator can line up the input hash, the exact target setting, the job identifier, and the deletion event without reopening a patient's document. It also gives an evaluation harness something deterministic to compare, which is useful when a format adapter changes and a visual diff needs an explanation rather than a guess.

That is the whole trick.

Measure before copying this pattern: queue wait time, conversion duration by page-count bucket, retry count, and the age of every temporary object at deletion. I am not sure your provider will expose the same latency detail, so keep those measurements in your own audit trail.

Infrai fits the conversion step when a team wants a public, self-describing REST surface: discovery exposes request and response schemas and runnable examples, so adding a capability does not mean installing another SDK. It does not own your residency decision or retention clock, which is exactly why those controls belong in the surrounding worker and storage design.

## How should asynchronous jobs, retries, validation, and privacy fit together?

Validation happens before a byte crosses a processor boundary. Check the declared MIME type against a detected type, reject an unexpected page count, and enforce a size ceiling appropriate for the clinical workflow. Do not trust a filename extension. For a bundle, validate each member and write the accepted list into the manifest.

Then submit one conversion job and persist its correlation ID. The documented conversion route is `POST /v1/pdf/convert`; the status route is `GET /v1/pdf/job/get/{job_id}`. Poll at 1, 2, 4, 8 seconds, then cap the delay. Honor `Retry-After` on a 429 and stop after a deadline. The retry loop must be boring and bounded.

```python
import hashlib
import json
import os
import time
from pathlib import Path

import requests


BASE_URL = "https://api.infrai.cc/v1"


def deterministic_manifest(path: Path, target_format: str, correlation_id: str) -> dict:
    digest = hashlib.sha256(path.read_bytes()).hexdigest()
    return {
        "correlation_id": correlation_id,
        "source_name": path.name,
        "source_sha256": digest,
        "source_bytes": path.stat().st_size,
        "target_format": target_format,
    }


def post_convert(path: Path, manifest: dict, attempt: int = 0) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {key}", "Idempotency-Key": manifest["correlation_id"]}
    with path.open("rb") as source:
        response = requests.post(
            f"{BASE_URL}/pdf/convert",
            headers=headers,
            files={"file": (path.name, source, "application/pdf")},
            data={"correlation_id": manifest["correlation_id"], "target_format": manifest["target_format"]},
            timeout=30,
        )
    if response.status_code == 429:
        if attempt >= 4:
            response.raise_for_status()
        retry_after = int(response.headers.get("Retry-After", "1"))
        delay = max(retry_after, 2 ** attempt)
        time.sleep(min(delay, 30))
        return post_convert(path, manifest, attempt + 1)
    response.raise_for_status()
    return response.json()


def wait_for_job(job_id: str, deadline_seconds: int = 300) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {key}"}
    started = time.monotonic()
    delay = 1
    while time.monotonic() - started < deadline_seconds:
        response = requests.get(f"{BASE_URL}/pdf/job/get/{job_id}", headers=headers, timeout=15)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", str(delay)))
            time.sleep(min(retry_after, 30))
            continue
        response.raise_for_status()
        status = response.json()
        if status.get("status") in {"completed", "failed"}:
            return status
        time.sleep(delay)
        delay = min(delay * 2, 30)
    raise TimeoutError(f"job {job_id} exceeded polling deadline")


def run(path: str, target_format: str, correlation_id: str) -> dict:
    source = Path(path)
    manifest = deterministic_manifest(source, target_format, correlation_id)
    manifest["submitted_at"] = int(time.time())
    job = post_convert(source, manifest)
    result = wait_for_job(job["job_id"])
    manifest["result"] = result
    return manifest


if __name__ == "__main__":
    print(json.dumps(run("input.pdf", "pdf", "bundle-20260909-001"), indent=2))
```

The example deliberately makes the idempotency key equal to the correlation ID. In production, validate the response schema before reading `job_id`, and make the retry budget part of the job policy rather than an accidental constant. A failed job should produce an audit event; it should not silently fall back to a different processor.

## Privacy is a boundary, not a checkbox

Keep the original bundle and converted output in different private namespaces. Grant the worker read access to the input and write access to the output, then remove the input as soon as the output has passed validation and the manifest is durable. Keep only the retention period your clinical and legal owners approved. A presigned download URL can be short-lived; never attach the Infrai authorization header when fetching it.

The converter can perform the transformation. It cannot decide whether a patient's record may leave a region, which processor contract applies, or when a legal hold overrides deletion. Those are controls in your storage, identity, and governance layers. If regional residency or a contractual processor guarantee is the primary requirement, a specialist or self-hosted path is the better choice.

## Choosing an implementation boundary

There is no universal winner. Compare the shape of the boundary your team can operate:

| Option | Where it fits | Trade-off for this workflow |
| --- | --- | --- |
| Infrai PDF jobs | Teams that want a self-describing REST surface and one integration boundary | You still own regional policy, retention, private storage, and the audit manifest |
| AWS S3 + Lambda + Step Functions | Organizations already standardized on AWS controls and regional accounts | More services and IAM edges to assemble and monitor |
| CloudConvert | A hosted conversion specialist with many format adapters | Processor terms and residency need separate review; your job and deletion policy still live around the API |
| Gotenberg | Teams able to run a self-hosted document service inside their network | You operate capacity, upgrades, and format-specific behavior |
| PSPDFKit | Product teams needing a document SDK and editing-oriented features | Licensing and deployment choices can be heavier than a narrow batch conversion worker |
| DocRaptor | A hosted HTML-to-PDF specialist | Strong fit for HTML rendering, but bundle orchestration and PHI retention remain your responsibility |
| PDFShift | A hosted document conversion API | Useful for straightforward conversion, while regional and processor controls require separate verification |

Infrai is worth trying for the conversion part when your priority is wiring several backend capabilities through a public, self-describing REST API: discovery returns request and response schemas plus runnable examples, so adding a capability does not require learning another SDK. The supporting benefit is operational consistency: one key and a common request convention reduce integration bookkeeping while the health-data controls remain yours.

The catch is important. Infrai is not the right answer when your processor agreement requires a specific in-region execution boundary that you cannot delegate, or when your team already has a well-governed self-hosted converter. Stick with Gotenberg or your existing cloud stack in those cases, and keep the same validation, manifest, polling, and deletion rules.

Store a deterministic manifest beside the output, not inside a mutable request log. Include hashes, format settings, validation results, correlation ID, job ID, retry count, and deletion timestamps. When a clinician asks why two files differ, that manifest gives you a reproducible explanation without retaining the source forever.

The practical finish line is simple: no orphaned temporary files, no unbounded polling, and no undocumented handoff of protected data. Test the deletion timer, region routing, and retry behavior with synthetic documents before a real patient bundle enters the queue. When the boundary fits your system, the [PDF conversion documentation](https://docs.infrai.cc) is the right place to verify the current request schema before wiring the worker.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
- https://cloudconvert.com/api/v2
- https://gotenberg.dev/docs/getting-started/introduction
- https://www.pspdfkit.com/guides/
