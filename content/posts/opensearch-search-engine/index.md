---
title: "Building a Serverless Search Engine on AWS with OpenSearch"
date: 2026-03-19T10:00:00-07:00
description: "How I built 'MyGoogle' — a full-text document search engine using S3, Lambda, OpenSearch, API Gateway, and CodePipeline."
tags: ["AWS", "OpenSearch", "Lambda", "Serverless", "CI/CD", "S3"]
author: "Chandra Kanth"
showToc: true
TocOpen: false
draft: false
---

## What I Built

The goal was simple but ambitious: build a personal search engine (nicknamed "MyGoogle") that lets you upload PDF documents and search their contents through a REST API. The entire pipeline is serverless — from document ingestion to full-text search.

Here's the workflow at a high level:

**Document Ingestion:**
User uploads PDF → S3 Bucket → Lambda extracts text → Lambda indexes content → OpenSearch

**Search:**
User sends query via curl → API Gateway → Lambda (gateway) → Lambda (search) → OpenSearch returns results

## Architecture & Components

The system uses seven AWS managed services working together:

- **S3 Buckets** — Two buckets: one for raw PDF uploads, one for extracted text (intermediary storage)
- **4 Lambda Functions** — `pdftotxt` (PDF-to-text extraction), `upload-tosearch` (indexes text into OpenSearch), `search-gateway` (handles API Gateway requests), `search-function` (executes search queries)
- **OpenSearch Domain** — Full-text search engine with fine-grained access control
- **API Gateway** — HTTP API with a POST `/search` route
- **CodePipeline + CodeBuild** — CI/CD pipeline pulling from GitHub, building with SAM templates, deploying via CloudFormation
- **CloudWatch** — Monitoring and log verification at every stage

## Implementation Walkthrough

### Phase 1: OpenSearch Domain

I started by creating the OpenSearch domain (`kclite-public`) with public access and fine-grained access control enabled. This was my first mistake — **I created OpenSearch too early**. The domain runs whether you're using it or not, and it's one of the more expensive components. In hindsight, I should have set it up last, right before integration testing.

Configuration highlights:
- OpenSearch 3.1 (latest at the time)
- Public access networking
- Fine-grained access control with an internal user database
- Encryption at rest and in transit enabled

### Phase 2: IAM Roles and OpenSearch Mapping

This was the trickiest part. Lambda functions need the right permissions to talk to both S3 and OpenSearch, and OpenSearch needs to know which IAM roles to trust.

I created two key IAM roles:
- `OpenSearchLambdaPolicy` — with `AWSLambdaBasicExecutionRole`, `AWSLambdaVPCAccessExecutionRole`, and `AmazonOpenSearchServiceFullAccess`
- `LambdaExecutionRole` — auto-created by the CloudFormation stack with `AWSLambdaBasicExecutionRole` and custom OpenSearch/S3 policies

The critical step most guides skip: **mapping these IAM role ARNs inside the OpenSearch Dashboard** under Security → Roles → `all_access` → Mapped Users. Without this, Lambda functions authenticate successfully with AWS but get rejected by OpenSearch's internal access control.

### Phase 3: Lambda Functions via CloudFormation

All four Lambda functions were deployed through a CI/CD pipeline rather than manually:

1. **pdftotxt** — Triggered by S3 PUT events on `.pdf` files. Uses PyPDF to extract text and writes `.txt` files to the intermediary bucket.
2. **upload-tosearch** — Triggered by S3 PUT events on `.txt` files in the intermediary bucket. Reads the text and indexes it into OpenSearch using signed HTTP requests.
3. **search-gateway** — Fronts the API Gateway, handles CORS preflight requests, and routes to the search function.
4. **search-function** — Executes the actual OpenSearch query and returns results.

The Lambda functions use Python 3.9 with Lambda Layers for dependencies like `pypdf` and `requests_aws4auth`.

### Phase 4: S3 Event Triggers

Two S3 buckets with event notifications create the automated pipeline:

- **Document store bucket** — On PUT of any `.pdf` file, triggers `pdftotxt`
- **Intermediary storage bucket** — On PUT of any `.txt` file, triggers `upload-tosearch`

This creates a clean chain: upload a PDF, and it automatically gets extracted, converted, and indexed — no manual steps needed.

### Phase 5: CI/CD with CodePipeline

The entire Lambda deployment is automated:

- **Source:** GitHub repository connected via CodeStar connection
- **Build:** CodeBuild using a SAM template to package the Lambda functions
- **Deploy:** CloudFormation stack creates/updates all Lambda functions, layers, API Gateway routes, and S3 triggers

Every `git push` triggers a full build-and-deploy cycle. This made iterating on Lambda code much faster than manual zip uploads.

### Phase 6: API Gateway

The HTTP API exposes a single `POST /search` endpoint backed by the `search-gateway` Lambda function, deployed to a `prod` stage. No authorization was configured since this was a learning project, but in production you'd want Cognito or API keys.

## Testing It All

The moment of truth: uploading a test PDF (`Games.pdf`) and watching the chain reaction in CloudWatch logs.

1. PDF lands in the document store bucket
2. `pdftotxt` fires, extracts text, writes `Games.pdf.txt` to intermediary bucket
3. `upload-tosearch` fires, indexes the content into OpenSearch
4. Search via curl returns the indexed content:

```bash
curl -X POST https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/search \
  -H "Content-Type: application/json" \
  -d '{"query": "games"}'
```

The response came back with the full extracted text, metadata (title, author, date), and a relevance score. Seeing it work end-to-end was genuinely satisfying.

## Lessons Learned

**Create expensive resources last.** OpenSearch was the costliest component, and I created it first. It sat idle while I spent hours configuring everything else. On a non-free-tier account, that adds up fast.

**IAM role mapping in OpenSearch is a hidden gotcha.** AWS IAM authentication gets you past the front door, but OpenSearch's fine-grained access control is a separate layer. You must explicitly map your Lambda execution role ARNs in the OpenSearch Dashboard. This isn't always obvious from the AWS documentation.

**CI/CD pays for itself quickly.** Initially I considered deploying Lambda functions manually, but setting up CodePipeline with CloudFormation saved enormous time during debugging. Each code fix was a `git push` away from deployment instead of a zip-upload-test cycle.

**S3 event triggers are powerful but need precise configuration.** The suffix filters (`.pdf` and `.txt`) ensure only the right file types trigger the right functions. Without those filters, you'd get infinite loops or misfires.

**CloudWatch is your best debugging friend.** Every Lambda invocation logs to CloudWatch. When the search function wasn't returning results, CloudWatch showed me that the OpenSearch index name was wrong in the environment variables — a quick fix that would have taken much longer without proper logging.

## Final Thoughts

This project tied together more AWS services than any single project I've done before. The serverless architecture means there are no servers to manage, and the CI/CD pipeline means deployments are automatic. For a learner working through cloud projects, building something end-to-end like this — where you can actually *use* the result — makes the concepts stick far better than isolated exercises.
