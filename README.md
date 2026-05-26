# MP1: Build-time SBOM Comparison

**Software Supply Chain Security – Mini Project 1 · USEEN1 · CNAM Paris**

This repository contains the artifacts, SBOM outputs, build attestations, and comparison results for a study of Software Bill of Materials (SBOM) generation at different levels of the software supply chain.

---

## Project Goal

Evaluate the differences between SBOMs generated at the source level and the container image level across three OSS projects in different language ecosystems. The comparison focuses on:

- **Missing dependencies** — packages present in the image but absent from source declarations
- **Unexpected components** — runtime libraries introduced by container packaging
- **Build/runtime mismatches** — divergence between Syft and Trivy on the same artifact
- **Tool coverage differences** — how binary fingerprinting (Syft) compares to manifest resolution (Trivy)

> OS-level dependencies (Alpine `pkg:apk` packages) are excluded from the primary comparison per project scope. Raw counts including OS packages are provided separately for reference.

---

## Selected Projects

| Language | Project | Package Manager |
|---|---|---|
| Node.js | [docker-node-example](https://github.com/nickjj/docker-node-example) | npm / `package.json` |
| Python | [sample-flask-api-restx](https://github.com/app-generator/sample-flask-api-restx) | pip / `requirements.txt` |
| Go | [go-gin-example](https://github.com/eddycjy/go-gin-example) | Go modules / `go.mod` |

---

## Tools Used

| Tool | Purpose |
|---|---|
| [Syft](https://github.com/anchore/syft) | SBOM generation via binary fingerprinting |
| [Trivy](https://github.com/aquasecurity/trivy) | SBOM generation via manifest resolution |
| [Witness](https://github.com/in-toto/witness) | Cryptographically signed build attestation |
| Docker | Build Alpine-based container images |
| jq | Parse CycloneDX JSON, filter OS packages |

**SBOM format:** CycloneDX 1.6 (`.cdx.json`)  
**Attestation predicate type:** `https://witness.testifysec.com/attestation-collection/v0.1`

---

## Results

### Application Dependencies (OS Packages Excluded)

OS packages filtered using:
```bash
jq '[.components[] | select(.purl | startswith("pkg:apk") | not)
  | select(.purl | startswith("pkg:deb") | not)] | length' sbom.cdx.json
```

| Language | Syft Source | Syft Image | Trivy Source | Trivy Image | Syft Δ |
|---|---:|---:|---:|---:|---:|
| Node.js | 902 | 1,319 | 273 | 1,418 | +46% |
| Python (Flask) | 8 | 51 | 9 | 520 | +538% |
| Go (Gin) | 107 | 57 | 231 | 267 | **−47%** |

### Raw Counts (Including OS Packages — Reference Only)

| Language | Syft Source | Syft Image | Trivy Source | Trivy Image |
|---|---:|---:|---:|---:|
| Node.js | 906 | 5,337 | 273 | 1,418 |
| Python (Flask) | 9 | 22,236 | 9 | 520 |
| Go (Gin) | 121 | 13,912 | 231 | 267 |

---

## Key Findings

### Missing Dependencies
- **Python:** 8 source deps → 51 image deps. The container adds CPython runtime and pip toolchain not declared in `requirements.txt`.
- **Node.js:** 902 → 1,319. npm installs transitive dependencies at build time not explicitly in `package.json`.
- **Go:** Unique case — source shows *more* than image (107 → 57). Static compilation bakes all modules into the binary, making them invisible to image-level scanning.

### Unexpected Components
- Alpine base image injects OS packages into all three containers (excluded from primary analysis).
- Python image includes full interpreter + pip toolchain beyond what `requirements.txt` declares.
- Node.js image includes the Node runtime binary and npm, not declared in `package.json`.
- Go image has minimal unexpected application-layer components — static linking keeps it clean.

### Build/Runtime Mismatches (Syft vs Trivy)
- At source level, tools diverge significantly: Node.js 902 (Syft) vs 273 (Trivy). Syft walks `node_modules/`; Trivy reads `package.json` only.
- Go at source is the exception — Trivy finds *more* (231 vs 107) because `go.sum` explicitly lists all indirect dependencies.
- **Neither tool is wrong.** Syft answers: *what is physically present?* Trivy answers: *what was officially installed?* Using both provides more complete coverage than either alone.

---

## Methodology

### Scan Commands

```bash
# Syft — source scan
syft dir:. -o cyclonedx-json > sboms/[app]-syft-source.cdx.json

# Syft — image scan
syft image:<tag> -o cyclonedx-json > sboms/[app]-syft-image.cdx.json

# Trivy — source scan
trivy fs . --format cyclonedx > sboms/[app]-trivy-source.cdx.json

# Trivy — image scan
trivy image <tag> --format cyclonedx > sboms/[app]-trivy-image.cdx.json

# Witness — build attestation
witness run -s build --signer-file-key-path keys/buildkey.pem \
  -- docker build -t <tag> .

# Component count
jq '.components | length' sbom.cdx.json
```

### Workflow

```
Clone OSS Projects → Source Scan → Docker Build → Image Scan → Witness Attestation → Compare
```

1. **Source Level** — Syft and Trivy scan application source directories
2. **Image Level** — Syft and Trivy scan built Alpine-based Docker images
3. **Attestation** — Witness wraps `docker build` and produces signed provenance JSON

---

## Challenges and Limitations

| Challenge | Decision |
|---|---|
| Witness produces attestation envelopes, not CycloneDX lists | Used Witness for build provenance; Syft/Trivy for component comparison |
| OS packages dominate raw image counts | Alpine base images used throughout; `pkg:apk` filtered post-scan |
| Python scanned without `venv` isolation | Documented as methodology limitation; correct approach is to scan within an isolated virtual environment |
| Syft and Trivy diverge on same artifact | Reported both; confirmed this reflects different resolution strategies, not error |
| Go image shows fewer components than source | Identified as static compilation behaviour, not a scanning defect |

---

## Repository Structure

```
mp1-buildtime-sbom/
├── apps/
│   ├── nodejs-app/                    # Express Node.js application
│   ├── python-flask-app/              # Flask REST API
│   └── go-app/                        # Go Gin HTTP service
├── sboms/
│   ├── nodejs-syft-source.cdx.json
│   ├── nodejs-syft-image.cdx.json
│   ├── nodejs-trivy-source.cdx.json
│   ├── nodejs-trivy-image.cdx.json
│   ├── python-syft-source.cdx.json
│   ├── python-syft-image.cdx.json
│   ├── python-trivy-source.cdx.json
│   ├── python-trivy-image.cdx.json
│   ├── go-syft-source.cdx.json
│   ├── go-syft-image.cdx.json
│   ├── go-trivy-source.cdx.json
│   └── go-trivy-image.cdx.json
├── attestations/
│   ├── nodejs-build-attestation.json
│   ├── python-build-attestation.json
│   └── go-build-attestation.json
├── keys/
│   ├── buildkey.pem
│   └── buildpublic.pem
├── results/
│   ├── nodejs/
│   ├── python/
│   ├── go/
│   └── summary.txt
└── README.md
```

---

## Conclusions

1. Source-level SBOMs consistently undercount the real attack surface — sometimes dramatically (Python: 8 → 51 app-layer deps, even after OS exclusion).
2. Image-level SBOMs are the closest proxy for what actually runs in production.
3. Go inverts the expected pattern — static compilation makes source SBOMs more informative than image scans for statically compiled languages.
4. Tool divergence is a finding, not an error. Combining Syft and Trivy provides more complete supply chain visibility than either alone.
5. Build attestation via Witness adds a cryptographic provenance layer independent of component inventory.

---

## Next Steps

- [ ] Re-run Python scans inside an isolated `venv` to eliminate system-wide package contamination
- [ ] Wrap executing application with Witness for true runtime SBOM observation
- [ ] Run SBOMs through [Grype](https://github.com/anchore/grype) or OSV to compare CVE coverage per scan level
- [ ] Use `sbomasm` or `cdxgen diff` for automated component mismatch detection
- [ ] Align Witness attestations with SLSA Level 2/3 provenance requirements

---

## References

- https://github.com/anchore/syft
- https://github.com/aquasecurity/trivy
- https://github.com/in-toto/witness
- https://cyclonedx.org
- https://github.com/nickjj/docker-node-example
- https://github.com/app-generator/sample-flask-api-restx
- https://github.com/eddycjy/go-gin-example
