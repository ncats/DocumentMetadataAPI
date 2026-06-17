# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

DocumentMetadataAPI is a Flask REST API that returns bibliographic metadata (title, journal, abstract, pub date, etc.) for biomedical publications, looked up by PMID, PMC ID, or DOI. It targets a p90 SLO of 150ms.

The backend is MongoDB (DocumentDB in production), accessed via the `pymongo` driver. Tracing is done via OpenTelemetry → Jaeger.

## Running locally

```bash
pip install -r requirements.txt
python main.py          # Flask dev server on port 5000
```

For production-style serving (matches Docker):
```bash
gunicorn -b 0.0.0.0:8000 --workers 1 --threads 8 --timeout 0 main:app
```

The app connects to MongoDB. Without a `connection_string` env var it defaults to a local MongoDB instance (`client['local']`). In production, set:
```bash
export connection_string="mongodb://..."
```

## API endpoints

- `GET /` — health check; returns sample IDs from each ID type
- `GET /version` — returns version string
- `GET /publications?pubids=PMID:123,PMC456&request_id=<uuid>` — main endpoint; returns metadata + `_meta` wrapper
- `GET /identifiers?pubids=PMID:123,DOI:10.1/x` — cross-reference lookup; returns synonyms across PMID/PMC/DOI

## Architecture

**`main.py`** — Flask app with two main routes:
- `/publications`: queries the `documentMetadata` MongoDB collection by `document_id`. For missing IDs, falls back to the NCBI PMC ID converter API (`www.ncbi.nlm.nih.gov/pmc/utils/idconv/v1.0/`).
- `/identifiers`: queries the `documentIds` reference collection for cross-ID lookups, then falls back to NCBI PMC ID converter for misses. Processes PMID, PMC, and DOI in separate batches.

**`data_loader.py`** — AWS Lambda handler for bulk-loading data. Reads gzipped TSV files from GCS (via boto3 S3-compatible client), parses them, fetches PMID→PMC/DOI synonyms, and upserts into MongoDB. The same document is stored three times: once per identifier (PMID, PMC, DOI), all pointing to identical metadata.

**`data_checker.py`** — AWS Lambda handler for auditing. Verifies whether documents from a given TSV file are present (or absent for delete files) in MongoDB.

**`query_tester.py`** — Local load-testing script. Hits the production endpoint with batches of 10/50/100 IDs and reports `processing_time_ms` statistics.

## MongoDB collections

- `documentMetadata` — main collection; keyed on `document_id` (e.g., `PMID:30690000`, `PMC1234`, `10.1000/xyz`)
- `documentIds` — reference/synonym collection; fields `PM`, `PMC`, `DOI`

## ID format conventions

The API normalizes input IDs before lookup:
- `PMC:` prefix → `PMC` (no colon)
- `DOI:` prefix → stripped entirely
- Lookups are case-insensitive for prefixes

## Deployment

Jenkins CI (`Jenkinsfile`) builds a Docker image, pushes to AWS ECR (`853771734544.dkr.ecr.us-east-1.amazonaws.com/translator-docmetadataapi`), and deploys to AWS EKS. The pipeline polls SCM every 5 minutes and targets the `translator-eks-ci-blue-cluster`.

`rds-combined-ca-bundle.pem` is the TLS CA bundle for AWS DocumentDB (MongoDB-compatible); the Dockerfile now fetches the updated global bundle from AWS at build time instead.
