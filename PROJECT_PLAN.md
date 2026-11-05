# FileFlow: Serverless Pipeline on LocalStack

<goal>
Build a serverless pipeline that runs entirely on a laptop using LocalStack (a local AWS emulator):
upload a CSV file to an S3 bucket -> S3 event triggers a Python Lambda -> Lambda analyzes the file
and stores a summary record in DynamoDB. No real AWS account and no cost.
This is a beginner-friendly GitHub portfolio project. The owner (Aditya, B.Tech CSE student, Python/SQL/Docker user,
MacBook Air Apple Silicon) wants to learn AWS basics and show clean engineering: Docker, tests, CI, good README.
</goal>

<constraints>
- Python 3.12, boto3, pytest. Docker Desktop and Docker Compose.
- LocalStack only. Never use real AWS credentials. Use dummy creds: access key `test`, secret `test`, region `us-east-1`.
- Keep it simple: no frameworks beyond boto3. One command to start, one command to run the demo.
- Explain each step in plain language; the owner is learning. Prefer small working steps over big rewrites.
</constraints>

<instructions_for_any_ai_continuing_this_project>
1. Read this whole file first, then check "Progress Tracker" below to see what is done.
2. Continue from the first unchecked item. Do not redo finished phases.
3. After finishing a step, tick its box and add one line under "Session Log" (date, what changed, what is next).
4. Verify each phase with its "Done when" check before moving on.
5. If something fails, use the Troubleshooting table before inventing new approaches.
6. Never put secrets in the repo. Keep the README accurate.
</instructions_for_any_ai_continuing_this_project>

## Architecture

```
 you ──upload CSV──> [S3 bucket: fileflow-uploads]
                          │  event: ObjectCreated
                          ▼
                [Lambda: process-upload]  ── reads the file back from S3
                          │  computes row count, columns, empty-cell counts
                          ▼
                [DynamoDB table: FileResults]  (key: file_key)
```
All three services run inside one LocalStack container at http://localhost:4566.

## Repo Layout

```
fileflow/
├── docker-compose.yml
├── lambda/handler.py          # Lambda code
├── scripts/setup.sh           # creates bucket, table, role, function, trigger
├── scripts/demo.sh            # uploads sample file, prints DynamoDB result
├── samples/orders.csv         # small test file
├── tests/test_handler.py      # unit tests (no LocalStack needed)
├── tests/test_integration.py  # end-to-end test (needs LocalStack running)
├── requirements-dev.txt       # boto3, pytest, awscli-local
├── .travis.yml or .github/workflows/ci.yml
├── PROJECT_PLAN.md            # this file
└── README.md
```

## Progress Tracker

- [ ] Phase 0: Prerequisites installed
- [ ] Phase 1: LocalStack running via Docker Compose
- [ ] Phase 2: S3 bucket + DynamoDB table created
- [ ] Phase 3: Lambda written, packaged, deployed
- [ ] Phase 4: S3 trigger connected, end-to-end works
- [ ] Phase 5: Automated tests (unit + integration)
- [ ] Phase 6: CI pipeline passing
- [ ] Phase 7: README with diagram, GIF/screenshot, lessons learned
- [ ] Stretch goals (optional)

## Session Log

- (none yet)

---

## Phase 0: Prerequisites

1. Install Docker Desktop and confirm `docker --version` and `docker compose version` work.
2. Install Python 3.12 and create a virtual environment: `python3 -m venv .venv && source .venv/bin/activate`.
3. Create `requirements-dev.txt` with: `boto3`, `pytest`, `awscli-local`, `awscli`. Then `pip install -r requirements-dev.txt`.
4. Export dummy credentials (put in your shell or a `.env` that is git-ignored):
   `export AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_DEFAULT_REGION=us-east-1`
5. `git init`, add a `.gitignore` (`.venv/`, `build/`, `__pycache__/`, `.env`).

Done when: `awslocal --version` prints a version.

## Phase 1: Start LocalStack

`docker-compose.yml`:
```yaml
services:
  localstack:
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"
    environment:
      - SERVICES=s3,lambda,dynamodb,iam,logs
      - LOCALSTACK_AUTH_TOKEN=${LOCALSTACK_AUTH_TOKEN:-}
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
```
The docker.sock mount is required: LocalStack runs each Lambda inside its own container.

Run `docker compose up -d`, then `curl http://localhost:4566/_localstack/health`.

Note: recent LocalStack versions may require a free account and an auth token (`LOCALSTACK_AUTH_TOKEN`).
If the container exits with a license/token message, create an account at app.localstack.cloud,
get a token (GitHub Student Pack or the free non-commercial plan), and put it in `.env`. Check the
current LocalStack docs because licensing rules change.

Done when: the health endpoint lists s3, lambda and dynamodb as `available` or `running`.

## Phase 2: Bucket and Table

Put these in `scripts/setup.sh` (start with `set -e`):
```bash
awslocal s3 mb s3://fileflow-uploads

awslocal dynamodb create-table \
  --table-name FileResults \
  --attribute-definitions AttributeName=file_key,AttributeType=S \
  --key-schema AttributeName=file_key,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

awslocal iam create-role --role-name lambda-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```
Make the script safe to re-run (ignore "already exists" errors, or restart LocalStack for a clean state).

Done when: `awslocal s3 ls` shows the bucket and `awslocal dynamodb list-tables` shows `FileResults`.

## Phase 3: The Lambda

`lambda/handler.py`:
```python
import csv, io, os
from datetime import datetime, timezone
import boto3

# LocalStack injects the endpoint into Lambda; None falls back to default behaviour.
ENDPOINT = os.environ.get("AWS_ENDPOINT_URL")
s3 = boto3.client("s3", endpoint_url=ENDPOINT)
table = boto3.resource("dynamodb", endpoint_url=ENDPOINT).Table(
    os.environ.get("TABLE_NAME", "FileResults")
)

def summarize_csv(text):
    """Pure function: easy to unit test without AWS."""
    rows = list(csv.DictReader(io.StringIO(text)))
    columns = list(rows[0].keys()) if rows else []
    empty = {c: sum(1 for r in rows if not (r.get(c) or "").strip()) for c in columns}
    return {"row_count": len(rows), "columns": columns, "empty_cells": empty}

def lambda_handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]
        item = {"file_key": key, "bucket": bucket,
                "processed_at": datetime.now(timezone.utc).isoformat()}
        try:
            body = s3.get_object(Bucket=bucket, Key=key)["Body"].read().decode("utf-8")
            item.update(summarize_csv(body))
            item["status"] = "OK"
        except Exception as exc:  # store failures instead of losing them
            item["status"] = "FAILED"
            item["error"] = str(exc)
        table.put_item(Item=item)
    return {"statusCode": 200}
```
Notes: DynamoDB rejects Python floats; use ints/strings only (or `Decimal`). Keep the function idempotent: re-uploading the same key overwrites the record.

Package and deploy (add to `setup.sh`):
```bash
mkdir -p build && (cd lambda && zip -r ../build/function.zip handler.py)
awslocal lambda create-function --function-name process-upload \
  --runtime python3.12 --handler handler.lambda_handler \
  --zip-file fileb://build/function.zip \
  --role arn:aws:iam::000000000000:role/lambda-role \
  --environment "Variables={TABLE_NAME=FileResults}"
awslocal lambda wait function-active-v2 --function-name process-upload
```
To update code later: rezip, then `awslocal lambda update-function-code --function-name process-upload --zip-file fileb://build/function.zip`.

Done when: invoking manually works:
`awslocal lambda invoke --function-name process-upload --payload '{"Records":[]}' --cli-binary-format raw-in-base64-out out.json` returns status 200.

## Phase 4: Connect the Trigger

```bash
awslocal s3api put-bucket-notification-configuration --bucket fileflow-uploads \
  --notification-configuration '{"LambdaFunctionConfigurations":[{"LambdaFunctionArn":"arn:aws:lambda:us-east-1:000000000000:function:process-upload","Events":["s3:ObjectCreated:*"]}]}'
```
`samples/orders.csv` (include one empty cell on purpose):
```
order_id,item,qty,price
1,Dosa,2,40
2,Biryani,1,
3,Tea,3,10
```
`scripts/demo.sh`: upload the file, wait 3 seconds, then `awslocal dynamodb scan --table-name FileResults`.

Done when: after `awslocal s3 cp samples/orders.csv s3://fileflow-uploads/`, the scan shows an item with
`status: OK`, `row_count: 3`, and `price` empty count 1. This is the core milestone.

## Phase 5: Tests

- Unit (`tests/test_handler.py`): test `summarize_csv` with normal CSV, empty CSV, header-only CSV, missing values. Import the function directly; no AWS needed.
- Integration (`tests/test_integration.py`): with boto3 pointed at `http://localhost:4566`, upload a file, poll DynamoDB for up to ~15 seconds, assert the item. Mark it so it can be skipped when LocalStack is not running.

Done when: `pytest` passes.

## Phase 6: CI

Travis CI (from the Student Pack) or GitHub Actions (more common in industry; learning both is a plus).
Pipeline: checkout -> install Python deps -> run unit tests -> start LocalStack with Docker -> run `setup.sh` -> run integration test.
Add the passing badge to the README. If LocalStack needs a token in CI, store it as an encrypted secret and never commit it.

## Phase 7: README and Portfolio Polish

README must contain: one-line pitch, architecture diagram, 30-second demo GIF, quick start (`docker compose up -d`, `./scripts/setup.sh`, `./scripts/demo.sh`),
what each service does, how to run tests, "What I learned", and limitations.
Pin the repo on GitHub and add topics: `aws`, `localstack`, `lambda`, `s3`, `dynamodb`, `serverless`, `python`, `docker`.

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Container exits on startup, mentions license/token | Newer LocalStack requires auth | Set `LOCALSTACK_AUTH_TOKEN`, check current docs |
| `awslocal: command not found` | venv not active or package missing | `source .venv/bin/activate`, `pip install awscli-local` |
| `Unable to locate credentials` | Dummy creds not exported | Export the three AWS env vars from Phase 0 |
| Lambda stuck in `Pending` or fails to start | docker.sock not mounted, or first image pull is slow | Check the compose volume, wait, `awslocal lambda get-function --function-name process-upload` |
| Upload works but DynamoDB stays empty | Trigger missing or Lambda errored | `awslocal s3api get-bucket-notification-configuration --bucket fileflow-uploads`; check Lambda logs (below) |
| Lambda logs | n/a | `docker compose logs localstack`, or `awslocal logs describe-log-groups` then `awslocal logs get-log-events` |
| `Float types are not supported` | Float written to DynamoDB | Convert to `int`, `str`, or `Decimal` |
| `ResourceNotFoundException` on table | Setup not run, or LocalStack restarted (state lost) | Re-run `scripts/setup.sh` |
| `InvalidParameterValueException` on create-function | Role missing or wrong ARN | Create the role first, use `arn:aws:iam::000000000000:role/lambda-role` |
| Port 4566 already in use | Old container running | `docker compose down`, then `up -d` |
| Works locally, fails in CI | LocalStack not ready yet, or token missing | Poll the health endpoint before running setup |

General approach when stuck: read the exact error, check LocalStack container logs, test one component alone
(S3 only, then Lambda invoke by hand, then the trigger), and search the error text plus "localstack".

## Stretch Goals (after Phase 7)

1. Add an SQS queue between S3 and Lambda, plus a dead-letter queue for failed files.
2. Add API Gateway + a second Lambda: `GET /files/{key}` returns the summary.
3. Define the infrastructure with Terraform (`tflocal`) or AWS CDK instead of shell scripts.
4. Support more file types (JSON, images with metadata) and write a small Streamlit or Bootstrap page showing results.
5. Port the same idea to Azure (Blob Storage + Azure Functions + Cosmos DB) using the student credits, and compare the two clouds in the README.
