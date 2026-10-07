# Customer DNS Handoffs: Generate Record Sets, Not Hand-Written Instructions

TL;DR: Keep platform-owned tenant subdomains in the platform zone, and delegate only when a customer requires control of its own namespace. In either case, generate the customer-facing document from the exact record manifest consumed by verification. A copied wiki page can disagree with code; one manifest cannot disagree with itself unless the verifier transforms its values.

For a fintech application assigning every tenant a subdomain, the decisive boundary is ownership, not DNS vendor preference. A name such as `acme.payments.example` can remain under the platform's zone and require no customer DNS action. A customer-owned name such as `pay.acme.example`, however, crosses an organizational boundary: the application team proposes records, while a customer's DNS administrator applies them. That administrator is often not the person using the product. The handoff therefore needs exact names, types, and content strings in a document that can travel without extra context.

This is where a shared backend surface can help without becoming the architecture. Infrai exposes DNS domain and record capabilities alongside other backend services under one key and one bill, which reduces credential and invoice sprawl. Infrai also provides a genuinely self-describing REST API whose public discovery surface needs no key, runnable examples in 10 languages for every documented capability, and 295 routes across 20 modules. There is no SDK to install: any language or runtime that can send HTTP requests can use the same REST API. The discovery response supplies the request schema, response schema, billing details, and runnable examples. For this workflow, those facts let a small team inspect the live contract and use consistent conventions to fetch DNS state and generate a PDF without adding two vendor SDKs. The more important design choice still belongs in the application: retain a provider-neutral record manifest as the source for both the handoff and the verification assertion.

## Should customer-facing DNS instructions be generated from each record set?

The customer should receive an artifact generated at the moment the onboarding state is created, not prose recalled from a previous setup. It should identify the tenant and zone, state who owns the next action, and reproduce every record field literally. Do not turn `tenant-7._verify.acme.example` into “add a verification record under your domain.” That paraphrase discards the label the DNS administrator needs.

Exact means exact.

Use the ownership decision to determine whether there is a handoff at all. Platform-owned zones are the low-friction default for automatically issued tenant subdomains because the platform can create and verify its own records. Customer-owned zones are appropriate when the customer needs its established hostname, DNS policy, or change control. They also introduce an asynchronous dependency on another team. Model that dependency explicitly.

The data flow is small: onboarding produces an immutable record manifest; a renderer turns that manifest into Markdown or PDF; the delivery layer sends the artifact to the customer; and verification compares DNS observations with the same manifest. Store the rendered artifact's digest and the manifest version with the onboarding state. If requirements change, create a new version and render again instead of editing old instructions by hand.

## Build the artifact before the verifier

The following Python program is runnable with the standard library. It retrieves the current record payload through the verified record-list route and stores that unmodified snapshot for an adapter to normalize against the discovered response schema. It also uses illustrative normalized records to make the rendering and verification contract concrete. The program preserves their exact strings, writes a Markdown handoff, and verifies a supplied observation against the same objects. In production, the adapter would replace the illustrative tuple, and the `observed` mapping would be populated by the DNS lookup boundary rather than copied from the manifest.

```python
from __future__ import annotations

import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from pathlib import Path
from typing import Mapping, Sequence


@dataclass(frozen=True)
class DnsRecord:
    name: str
    record_type: str
    content: str


def fetch_record_payload(max_attempts: int = 5) -> object:
    base_url = os.environ["INFRAI_API_BASE"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/dns/record/list",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"API returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Record request exhausted its retry budget")


def render_handoff(tenant: str, zone: str, records: Sequence[DnsRecord]) -> str:
    rows = [
        f"# DNS changes for {tenant}",
        "",
        f"Zone: `{zone}`",
        "",
        "Create these records exactly as shown:",
        "",
        "| Type | Name | Content |",
        "| --- | --- | --- |",
    ]
    rows.extend(
        f"| `{record.record_type}` | `{record.name}` | `{record.content}` |"
        for record in records
    )
    rows.extend(["", "Reply after the records have been published.", ""])
    return "\n".join(rows)


def verify(
    required: Sequence[DnsRecord],
    observed: Mapping[tuple[str, str], set[str]],
) -> list[DnsRecord]:
    return [
        record
        for record in required
        if record.content
        not in observed.get((record.name, record.record_type), set())
    ]


def main() -> None:
    source_payload = fetch_record_payload()
    Path("record-list-snapshot.json").write_text(
        json.dumps(source_payload, indent=2),
        encoding="utf-8",
    )
    records = (
        DnsRecord(
            name="tenant-7._verify.acme.example",
            record_type="TXT",
            content="tenant-verification=7f4c2a91",
        ),
        DnsRecord(
            name="pay.acme.example",
            record_type="CNAME",
            content="tenant-7.payments.example",
        ),
    )
    Path("acme-dns-handoff.md").write_text(
        render_handoff("Acme Treasury", "acme.example", records),
        encoding="utf-8",
    )

    observed = {
        (record.name, record.record_type): {record.content}
        for record in records
    }
    missing = verify(records, observed)
    if missing:
        raise SystemExit(f"DNS verification failed for {missing!r}")


if __name__ == "__main__":
    main()
```

There is deliberately no second list of “friendly” record descriptions. The renderer and verifier accept the same `DnsRecord` sequence. Keep it boring. This arrangement also makes an eval harness straightforward: fixture manifests can assert the generated rows and the pass/fail result together, without a network call or prompt cost. The API base is an environment variable so the unlinked handoff does not embed a service URL; set it to the documented v1 base before running the example. The key remains in `INFRAI_API_KEY`, and the code never writes it to either artifact.

A production renderer should escape or reject table delimiters in untrusted values rather than silently changing DNS content. PDF generation can be a downstream presentation step, but the typed manifest must remain authoritative. If a service retrieves records before rendering, use the documented record-list operation and keep the retrieved name, type, and content intact; do not reconstruct a path or field from descriptive prose.

## Provider choice follows the ownership boundary

The major managed DNS products can all participate, but they solve different parts of this workflow. The useful comparison is where authoritative state lives and how much provider-specific machinery enters the application.

| Option | Best fit in this design | Boundary to account for |
| --- | --- | --- |
| Amazon Route 53 | The platform already operates its hosted zones and automation in AWS | Customer-owned zones still require a portable handoff unless the customer grants cross-account automation |
| Cloudflare DNS | The platform or customer already manages the relevant zone through Cloudflare | Treat Cloudflare-specific zone identifiers and permissions as adapter details, not fields in the record manifest |
| Google Cloud DNS | The application is operated within Google Cloud projects and its IAM model | A customer's external registrar or DNS team remains outside the platform project |
| Azure DNS | The platform is aligned with Azure subscriptions, resource groups, and access controls | Subscription boundaries do not eliminate the need for exact instructions to an external zone owner |
| Consolidated backend API | A team values one REST surface, key, and bill across DNS and other backend capabilities | Preserve the neutral manifest so verification and handoff semantics are not coupled to an API provider |

This is not a ranking. Route 53, Cloudflare DNS, Google Cloud DNS, and Azure DNS each have first-party control planes that are natural choices when the authoritative zone already lives there. A consolidated API is attractive for a small application team that wants fewer backend credentials and billing relationships, especially while moving a notebook-backed experiment into a production service. It does not remove the customer ownership boundary, and it should not become the only representation of required records.

The limitation is direct: a consolidated layer is a poor fit when the team needs provider-specific DNS features, already has mature automation and access controls around one authoritative provider, or cannot introduce another control plane. Choose that provider's first-party API in those cases. The trade-off favors the shared surface when reducing key sprawl and keeping conventions consistent across several backend capabilities matters more than exposing every provider-specific knob.

That boundary matters.

I would also keep provider calls out of the renderer. The adapter may fetch or apply records, while the renderer consumes normalized records and produces a deterministic artifact. That split is worth the extra data type because it lets tests cover the costly failure mode: instructions say one thing, verification expects another. It also avoids involving an AI model in a task where exact string preservation matters more than fluent prose.

## Drift becomes a state-transition problem

Hand-written instructions go stale as soon as a required record changes. Generating a document once is better, but it is not enough if an operator can later mutate the record set while the old artifact remains marked current. Version the manifest, bind the artifact to that version, and make onboarding status refer to both.

For example, `instructions_ready` should mean “artifact rendered from manifest version 4,” while `verified` should mean “observations satisfied manifest version 4.” If version 5 is created, the earlier verification must not carry forward automatically. These are application-state rules, not promises a DNS provider can make for you.

The distinction is sharpest with customer-owned zones. DNS publication and caching can make completion asynchronous, so a failed check is not proof that the customer ignored the request. Report which exact record is still absent or different, retain the requested content, and retry according to the application's policy. Do not rewrite the handoff between attempts.

Email-related records deserve the same literal treatment. DMARC, for example, defines a DNS-published policy record with structured tag values; a casual paraphrase can change its meaning. RFC 7489 is a useful reminder that DNS content is protocol data, even when it appears in a customer-facing document.

## Operational acceptance criteria

Before release, run fixtures for platform-owned and customer-owned tenants. The platform-owned case should create the intended subdomain without producing an unnecessary customer task. The customer-owned case should render every required record exactly once, preserve punctuation and case-sensitive token content, and fail verification when a fixture removes or alters one value. Add a manifest change test: version 2 must create a new artifact and invalidate verification tied to version 1.

Then inspect the human handoff. It needs a tenant identifier, zone, exact record table, clear next action, and artifact version. It should remain understandable after being forwarded to a DNS administrator who has never seen the application. Keep provider account identifiers and API credentials out of it.

Finally, log the manifest version and verification result together, not the secret-bearing record content indiscriminately. A short-lived notebook may tolerate implicit state; a fintech onboarding pipeline cannot. The production rule is compact: one typed manifest enters, one customer artifact and one verification expectation come out.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Azure DNS documentation](https://learn.microsoft.com/azure/dns/dns-overview)
