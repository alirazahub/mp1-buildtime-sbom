# MP1: Build-time SBOM Comparison

## Software Supply Chain Security – Mini Project 1

This repository contains the artifacts and comparison results for a small study of Software Bill of Materials (SBOM) generation at build time.

The goal is to compare SBOMs produced from:
- application source directories,
- built container images,
- different tools and scanning approaches.

This helps demonstrate how source-only SBOMs differ from image-level SBOMs and what runtime dependencies are introduced by container packaging.

---

## Project Overview

A Software Bill of Materials documents the components used by an application. This project compares SBOMs generated from:

- application source code,
- installed build artifacts,
- container image contents.

The comparison highlights:
- missing runtime dependencies in source-only scans,
- OS/runtime packages included in container images,
- transitive dependency differences,
- tool-specific coverage differences.

---

## Repository Contents

This repository stores SBOM and comparison artifacts for three sample languages:

- `apps/nodejs-app/`
- `apps/python-flask-app/`
- `apps/go-app/`

Analysis outputs are grouped under:

- `results/` — summary and raw scan results,
- `sboms/` — generated CycloneDX SBOM files,
- `attestations/` — build attestation JSON artifacts.

---

## How to Use

1. Review the generated SBOM files in `sboms/`.
2. Inspect tool output in `results/<language>/`.
3. Use `results/summary.txt` for a quick counts comparison.

> Note: This repository currently contains comparison outputs and artifacts rather than build scripts.

---

## Results Summary

| Language | Syft Source | Syft Image | Trivy Source | Trivy Image |
|---|---|---|---|---|
| Node.js | 904 | 1084 | 274 | 1080 |
| Python | 9 | 537 | 10 | 521 |
| Go | 108 | 268 | 232 | 268 |

---

## Key Findings

- Image-level SBOMs reveal runtime packages and operating system dependencies that are not present in source-only scans.
- Source scans tend to capture only direct project dependencies, while image scans capture the full deployed environment.
- Syft and Trivy have different coverage characteristics: Syft is more exhaustive on nested dependencies, while Trivy is manifest-focused.
- The Python and Node.js sample applications show the largest gap between source and image SBOMs.

---

## Project Structure

```text
mp1-buildtime-sbom/
├── apps/
│   ├── go-app/
│   ├── nodejs-app/
│   └── python-flask-app/
├── attestation/
│   ├── go-build-attestation.json
│   ├── nodejs-build-attestation.json
│   └── python-build-attestation.json
├── results/
│   ├── go/
│   ├── nodejs/
│   ├── python/
│   └── summary.txt
├── sboms/
│   ├── go-*.cdx.json
│   ├── nodejs-*.cdx.json
│   └── python-*.cdx.json
└── README.md
```

---

## Conclusions

Build-time SBOM generation and image-level analysis provide stronger supply chain visibility than source-only SBOM generation. This repository demonstrates that deployed containers often include additional runtime dependencies and OS packages that are invisible when scanning just source code.

This work reinforces the importance of combining source and image SBOM generation for more complete dependency and security analysis.

---

## References

- https://github.com/anchore/syft
- https://github.com/aquasecurity/trivy
- https://github.com/nickjj/docker-node-example
- https://github.com/app-generator/sample-flask-api-restx
- https://github.com/eddycjy/go-gin-example
