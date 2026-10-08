# AWS Lambda: Beginner to Advanced Guide

A practical guide to AWS Lambda for backend development, architecture interviews, deployment, and CI/CD.

Examples use:

- Node.js 20.x with JavaScript
- Python 3.12
- AWS SAM (Serverless Application Model)
- Amazon API Gateway, Amazon S3, Amazon SQS, Amazon DynamoDB, Amazon EventBridge, and Amazon CloudWatch

AWS runtime names and managed-service features change over time. Confirm the runtime, region, quotas, and service limits in the AWS documentation before production rollout.

---

# 1. What is AWS Lambda?

AWS Lambda is a managed compute service that runs a function in response to events. You upload code and configuration; AWS manages servers, operating-system patching, capacity provisioning, and basic availability.

```text
Event source -> Lambda service -> Your handler -> AWS services or response
```

Typical uses:

- HTTP APIs and webhooks
- File processing after an S3 upload
- Queue consumers
- Scheduled jobs
- Event-driven integrations
- Stream processing
- Lightweight data transformation
- Automation and operational tasks

Lambda is not “free servers.” You still design for timeouts, retries, concurrency, permissions, observability, idempotency, and cost.

## 1.1 Lambda versus a continuously running server

| Concern | Lambda | VM/container service |
|---|---|---|
| Provisioning | Managed and event-driven | You manage instances/tasks |
| Scaling | Automatic within limits | Configure autoscaling or capacity |
| Billing | Requests and execution duration | Running capacity |
| Startup | Possible cold start | Usually already running |
| Maximum request | Bounded by Lambda limits | Depends on service |
| Best fit | Short, independent, event-driven work | Long-running or highly stateful processes |

Choose Lambda when the work can be split into bounded invocations and the operational benefit is greater than the constraints.

---

# 2. Lambda mental model

## Function

A function is code plus configuration: runtime, handler, memory, timeout, environment variables, IAM execution role, architecture, networking, and optional layers.

## Invocation

An invocation is one execution request. Lambda can invoke a function:

- Synchronously: the caller waits for the result.
- Asynchronously: Lambda queues the event and returns immediately.
- Through an event source mapping: Lambda polls a source such as SQS, Kafka, or DynamoDB Streams and invokes the function.

## Execution environment

Lambda creates an isolated execution environment for a function. The environment can be reused for later invocations, but reuse is not guaranteed.

```text
Create environment
  -> initialize runtime and module-level code
  -> invoke handler
  -> freeze or reuse
  -> eventually dispose
```

Never rely on memory, `/tmp`, open connections, or module-level state being present for the next invocation. Treat reuse as an optimization, not correctness.

## Handler

The handler is the function entry point. A Node.js handler commonly receives `(event, context)`; a Python handler receives `(event, context)`.

## Region and account

A function belongs to an AWS Region and account. Resources such as API Gateway, IAM roles, queues, and tables must be referenced in the correct account and region.

---

# 3. Your first Lambda function

## 3.1 Node.js handler

```js
// src/hello.mjs
export const handler = async (event, context) => {
  console.log("request", {
    requestId: context.awsRequestId,
    event,
  });

  return {
    statusCode: 200,
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ message: "Hello from Lambda" }),
  };
};
```

## 3.2 Python handler

```python
# src/hello.py
import json


def handler(event, context):
    print({"request_id": context.aws_request_id, "event": event})
    return {
        "statusCode": 200,
        "headers": {"content-type": "application/json"},
        "body": json.dumps({"message": "Hello from Lambda"}),
    }
```

`console.log` and `print` output is sent to CloudWatch Logs when the execution role permits it.

## 3.3 Handler configuration

The handler name is usually:

```text
Node.js: hello.handler
Python:   hello.handler
```

For an ES module or a file in a subdirectory, configure the exact module path and exported function. A handler mismatch causes an initialization error before business logic runs.

## 3.4 The event is an input contract

Events differ by source. An API Gateway event is not an S3 event, and an SQS invocation contains a batch of records. Log or inspect a safe sample event during development, then validate input explicitly.

---

# 4. Lambda invocation models

## 4.1 Synchronous invocation

The caller receives the function result or error immediately.

Examples:

- API Gateway
- Application Load Balancer
- Lambda Function URL
- Direct SDK invocation

```text
Client -> API Gateway -> Lambda -> API response
```

The caller owns retry behavior. Do not return a successful HTTP response when the operation failed.

## 4.2 Asynchronous invocation

The event is accepted, then Lambda invokes the function later. Lambda can retry failures and can send events to a dead-letter queue or destination.

Examples:

- S3 notifications
- EventBridge rules
- Direct asynchronous invocation

Design the handler to be idempotent because the same event can be delivered more than once.

## 4.3 Event source mappings

Lambda polls a source and invokes the function with batches.

Common sources:

- Amazon SQS and FIFO queues
- Kinesis Data Streams
- DynamoDB Streams
- Amazon MSK and self-managed Apache Kafka

For these integrations, understand batch size, batching window, visibility timeout, checkpointing, partial batch failure, and concurrency.

---

# 5. Core Lambda limits and configuration

Always verify current quotas for your Region. The important design settings are:

## Memory

Memory is configured per function. CPU and network throughput generally increase with memory. Benchmark representative workloads; the cheapest setting is not always the lowest-memory setting.

## Timeout

Timeout is the maximum duration of one invocation. Set it above normal execution time but below the caller’s timeout where possible. A timeout is a failure, not a retry strategy.

## Ephemeral storage

`/tmp` is temporary per execution environment. It is useful for downloads, decompression, and generated files, but it is not durable storage. Increase ephemeral storage only when the workload needs it.

## Environment variables

Use environment variables for non-secret configuration such as table names, feature flags, and log levels. Do not place long-lived credentials or sensitive secrets in plain environment variables. Use Secrets Manager or Systems Manager Parameter Store and grant narrow read permissions.

## Reserved and provisioned concurrency

- **Reserved concurrency** sets a function concurrency limit and reserves capacity for it. It can protect downstream systems and stop runaway invocations.
- **Provisioned concurrency** keeps execution environments initialized to reduce cold-start latency. It costs money while configured and is useful for latency-sensitive traffic.
- **Account or regional concurrency** is shared capacity across functions and can be increased through quotas.

## Architecture

Choose `arm64` for potential cost/performance benefits when all native dependencies support it. Choose `x86_64` when a dependency or binary requires it. Build dependencies for the selected architecture.

---

# 6. A production Lambda architecture

## 6.1 Synchronous API architecture

```text
                 +------------------+
Browser / Client | Route 53          |
        -------->| CloudFront (opt.) |
                 +--------+---------+
                          |
                          v
                 +------------------+
                 | API Gateway      |
                 | auth, throttling |
                 +--------+---------+
                          |
                          v
                 +------------------+
                 | Lambda API       |
                 | validation       |
                 | business logic   |
                 +---+----------+---+
                     |          |
                     v          v
                DynamoDB    EventBridge
                or RDS      / SQS
```

Use API Gateway for routing, authorization integration, throttling, request validation, and access logs. Keep the Lambda handler thin: parse the event, call application services, and map the result to the transport response.

## 6.2 Asynchronous order-processing architecture

```text
Client -> API Lambda -> DynamoDB order + outbox/event
                                  |
                                  v
                           EventBridge or SQS
                         +----------+----------+
                         |                     |
                         v                     v
                  Payment Lambda         Email Lambda
                         |                     |
                         v                     v
                    provider API       SES / notification
                         |
                         v
                    DLQ + alarms
```

Use a queue between components when work can be delayed, retried, or smoothed. Use a dead-letter queue and an alarm for messages that cannot be processed.

## 6.3 File-processing architecture

```text
User -> pre-signed S3 upload -> S3 bucket
                                      |
                                      v
                              S3 event notification
                                      |
                                      v
                             Lambda validation
                              /            \
                             v              v
                      quarantine bucket  processed bucket
                                             |
                                             v
                                      metadata in DynamoDB
```

Validate object key, size, content type, and ownership. Do not trust a client-provided filename or content type.

## 6.4 Multi-account architecture

```text
                    AWS Organizations
       +----------------+------------------+
       |                                   |
  Development account                 Production account
       |                                   |
  dev stack + logs                    prod stack + alarms
       |                                   |
       +---------- central CI/CD -------+
                    |
              audit/log account
```

Separate environments by account when isolation, billing, or security requirements justify it. Use cross-account deployment roles with short-lived credentials through OIDC rather than storing AWS access keys in CI.

---

# 7. Event sources and event shapes

## API Gateway

An API Gateway event commonly contains HTTP method, path, headers, query parameters, path parameters, request context, and a body. Configure whether the body is base64 encoded and validate the content type.

## S3

An S3 event contains one or more records with bucket and object information. The object key is URL-encoded in the event. Fetch the object using the AWS SDK and verify the bucket before processing.

## SQS

An SQS event contains `Records`. Each record has a message ID, body, receipt handle, attributes, and event source ARN.

```json
{
  "Records": [
    {
      "messageId": "msg-1",
      "body": "{\"orderId\":\"ord-42\"}",
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:region:account:orders"
    }
  ]
}
```

## EventBridge

An EventBridge event typically includes `source`, `detail-type`, `time`, `region`, `account`, and `detail`. Use event patterns to route only relevant events.

## Kinesis and DynamoDB Streams

Stream events contain batches of records and sequence/checkpoint information. Ordering is maintained within the relevant shard or partition, not globally. Configure parallelization and failure handling carefully.

## Kafka and Amazon MSK

Lambda can consume Kafka records through an event source mapping. Plan networking, authentication, offsets, batch failure handling, consumer lag, and broker connectivity. A Kafka trigger is not the same as a long-running Kafka consumer that you operate yourself.

---

# 8. AWS SDK and permissions

## 8.1 Use the SDK inside the function

Node.js example:

```js
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  PutCommand,
} from "@aws-sdk/lib-dynamodb";

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async (event) => {
  const item = {
    id: event.orderId,
    status: "RECEIVED",
    createdAt: new Date().toISOString(),
  };

  await client.send(new PutCommand({
    TableName: process.env.ORDERS_TABLE,
    Item: item,
    ConditionExpression: "attribute_not_exists(id)",
  }));

  return { statusCode: 201, body: JSON.stringify(item) };
};
```

Python example:

```python
import os
import boto3

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["ORDERS_TABLE"])


def handler(event, context):
    table.put_item(
        Item={"id": event["orderId"], "status": "RECEIVED"},
        ConditionExpression="attribute_not_exists(id)",
    )
    return {"statusCode": 201, "body": "created"}
```

Create SDK clients outside the handler so a reused environment can reuse connections. Keep credentials out of code; the execution role supplies temporary credentials.

## 8.2 Least-privilege execution role

A Lambda execution role should contain only actions and resources required by that function.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT:table/orders"
    }
  ]
}
```

The role also needs CloudWatch Logs permissions, usually through the AWS managed basic execution policy or an equivalent narrowly managed policy.

Distinguish:

- **Execution role:** what the function may do.
- **Resource policy:** who may invoke the function.
- **Deployment role:** what CI/CD may create or update.

---

# 9. A complete SAM example

The following example creates an HTTP API, a Lambda function, a DynamoDB table, and the minimum table permission. SAM transforms the template into CloudFormation resources.

## 9.1 Project structure

```text
lambda-orders/
├── template.yaml
├── src/
│   ├── app.mjs
│   ├── package.json
│   └── package-lock.json
└── tests/
    └── app.test.mjs
```

## 9.2 `template.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Example orders API

Globals:
  Function:
    Runtime: nodejs20.x
    Architectures:
      - arm64
    Timeout: 10
    MemorySize: 512
    Tracing: Active
    Environment:
      Variables:
        LOG_LEVEL: INFO

Resources:
  OrdersFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
      Environment:
        Variables:
          ORDERS_TABLE: !Ref OrdersTable
      Events:
        GetOrder:
          Type: HttpApi
          Properties:
            Path: /orders/{id}
            Method: GET
        CreateOrder:
          Type: HttpApi
          Properties:
            Path: /orders
            Method: POST

  OrdersTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: id
        Type: String
      BillingMode: PAY_PER_REQUEST

Outputs:
  OrdersApiUrl:
    Description: HTTP API endpoint
    Value: !Sub https://${ServerlessHttpApi}.execute-api.${AWS::Region}.amazonaws.com
```

For production, prefer explicit table definitions when you need point-in-time recovery, encryption configuration, indexes, deletion policies, or streams.

## 9.3 `src/package.json`

```json
{
  "name": "orders-function",
  "private": true,
  "type": "module",
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.0.0",
    "@aws-sdk/lib-dynamodb": "^3.0.0"
  }
}
```

Pin and regularly update dependencies according to your organization’s policy. Generate a lock file and build with it in CI.

## 9.4 `src/app.mjs`

```js
import { randomUUID } from "node:crypto";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  GetCommand,
  PutCommand,
} from "@aws-sdk/lib-dynamodb";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));

const response = (statusCode, body) => ({
  statusCode,
  headers: { "content-type": "application/json" },
  body: JSON.stringify(body),
});

export async function handler(event) {
  const tableName = process.env.ORDERS_TABLE;
  const method = event.requestContext?.http?.method;

  if (method === "GET") {
    const id = event.pathParameters?.id;
    if (!id) return response(400, { message: "id is required" });

    const result = await db.send(new GetCommand({
      TableName: tableName,
      Key: { id },
    }));
    return result.Item
      ? response(200, result.Item)
      : response(404, { message: "order not found" });
  }

  if (method === "POST") {
    let input;
    try {
      input = JSON.parse(event.body ?? "{}");
    } catch {
      return response(400, { message: "body must be valid JSON" });
    }

    if (typeof input.customerId !== "string" || input.customerId.length === 0) {
      return response(400, { message: "customerId is required" });
    }

    const order = {
      id: randomUUID(),
      customerId: input.customerId,
      status: "RECEIVED",
      createdAt: new Date().toISOString(),
    };

    await db.send(new PutCommand({
      TableName: tableName,
      Item: order,
      ConditionExpression: "attribute_not_exists(id)",
    }));
    return response(201, order);
  }

  return response(405, { message: "method not allowed" });
}
```

This is an instructional example, not a complete API validation or authorization layer. Add authentication, schema validation, request size limits, structured logs, and domain-specific error handling before production.

## 9.5 Build and deploy with SAM

```bash
sam validate
sam build
sam deploy --guided
```

The guided deployment asks for a stack name, Region, artifact bucket, and whether to save configuration. Later deployments can use:

```bash
sam deploy --config-env default
```

Useful local commands:

```bash
sam local invoke OrdersFunction -e events/create-order.json
sam local start-api
```

Local emulation is helpful, but it does not perfectly reproduce IAM, networking, service quotas, or managed-service behavior. Test against an AWS integration environment.

---

# 10. Deployment strategies

## 10.1 Console or ZIP deployment

Useful for a quick experiment:

1. Create an execution role.
2. Package the function and dependencies.
3. Create or update the function.
4. Configure environment variables, timeout, memory, and triggers.
5. Test and inspect logs.

This is difficult to audit and reproduce, so use Infrastructure as Code for shared or production environments.

## 10.2 AWS CLI

```bash
aws lambda create-function \
  --function-name orders-api \
  --runtime nodejs20.x \
  --handler app.handler \
  --role arn:aws:iam::ACCOUNT:role/orders-lambda-role \
  --architectures arm64 \
  --zip-file fileb://function.zip
```

CLI commands are useful for automation, but a template should remain the source of truth.

## 10.3 AWS SAM

SAM is a CloudFormation extension designed for serverless resources. It supports local testing, packaging, deployment, policies, event sources, and guided configuration.

## 10.4 AWS CDK

CDK defines infrastructure in TypeScript, Python, Java, C#, or Go and synthesizes CloudFormation.

```ts
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";

const fn = new lambda.Function(this, "OrdersFunction", {
  runtime: lambda.Runtime.NODEJS_20_X,
  architecture: lambda.Architecture.ARM_64,
  handler: "app.handler",
  code: lambda.Code.fromAsset("src"),
  timeout: cdk.Duration.seconds(10),
});
```

CDK is useful when you need reusable constructs and general-purpose programming, but review synthesized CloudFormation and IAM before deployment.

## 10.5 Serverless Framework and Terraform

These are valid alternatives. Select one primary IaC tool per service or team to avoid competing ownership of the same resources.

---

# 11. CI/CD for AWS Lambda

Yes. Lambda supports CI/CD through CloudFormation-based tools such as SAM and CDK, or through other IaC tools. A safe pipeline separates validation, build, deployment, and promotion.

## 11.1 Recommended pipeline

```text
Pull request
  -> lint + unit tests + dependency audit
  -> SAM validate + build
  -> optional integration tests in an ephemeral stack
  -> review and merge
  -> deploy to development
  -> smoke tests
  -> approval or automated promotion
  -> deploy to staging
  -> canary/linear production deployment
  -> CloudWatch alarms and rollback
```

## 11.2 Use OIDC instead of long-lived AWS keys

Configure a GitHub Actions OIDC identity provider in AWS and create a deployment role whose trust policy restricts:

- The GitHub organization and repository.
- The allowed branch or environment.
- The audience `sts.amazonaws.com`.

The workflow exchanges its short-lived GitHub identity token for temporary AWS credentials. Do not put `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in repository secrets when OIDC is available.

Example trust policy shape:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Use GitHub Environments for production approvals and restrict the production role further than the development role.

## 11.3 GitHub Actions with SAM

```yaml
# .github/workflows/lambda.yml
name: Lambda CI/CD

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/setup-sam@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: src/package-lock.json
      - run: npm ci
        working-directory: src
      - run: npm test
        working-directory: src
      - run: sam validate --template template.yaml
      - run: sam build --template template.yaml

  deploy-dev:
    if: github.event_name == 'push'
    needs: test
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: aws-actions/setup-sam@v2
      - run: sam build --template template.yaml
      - run: >
          sam deploy
          --stack-name orders-dev
          --resolve-s3
          --no-confirm-changeset
          --no-fail-on-empty-changeset
          --capabilities CAPABILITY_IAM
          --region "${{ vars.AWS_REGION }}"
      - run: ./scripts/smoke-test.sh "${{ vars.API_URL }}"
```

The example uses `--resolve-s3` for simplicity. In a controlled organization, use a managed artifact bucket, explicit stack configuration, pinned action versions, and a deployment role with only the required permissions. The SAM CLI is installed by `aws-actions/setup-sam` before validation, build, and deployment.

## 11.4 Production deployment with versions and aliases

Use immutable published versions and an alias such as `live`:

```text
orders-function:$VERSION
             \-> live alias
```

The alias can route a percentage of traffic to a new version for a canary. CloudFormation/SAM deployment preferences can connect traffic shifting to CloudWatch alarms. If alarms breach, the deployment rolls back to the previous version.

Do not point clients directly at an unqualified `$LATEST` version for production traffic.

## 11.5 Pipeline safeguards

- Run unit tests and static analysis before deployment.
- Pin action major versions and review action updates.
- Restrict CI role permissions and use separate roles per environment.
- Require approval for production.
- Add CloudWatch alarms before shifting production traffic.
- Run database migrations as an explicit, reviewed step.
- Keep templates and application code versioned together.
- Record the commit SHA in function metadata or an environment variable.
- Make deployment repeatable and safe when no changes exist.

---

# 12. Reliability and failure handling

## Idempotency

Retries and duplicate events are normal. Use an idempotency key such as an event ID or business key and persist it with the result.

```text
if idempotency key exists:
    return stored result
else:
    perform operation
    atomically store key and result
```

For DynamoDB, a conditional write can prevent duplicate creation. For APIs, accept an idempotency key and define its retention period.

## Retry behavior

Understand who retries:

- API clients and SDK callers may retry.
- Asynchronous Lambda invocations retry automatically.
- SQS makes a message visible again after the visibility timeout.
- Stream sources retry batches until success or configured failure handling.

Avoid retrying permanent errors such as invalid input. Route poison messages to a DLQ or failure destination.

## SQS visibility timeout

The visibility timeout should exceed the maximum processing time by a safety margin. If it is too short, the same message can be delivered concurrently while the first invocation is still working.

## Partial batch failure

For batch sources, configure partial batch responses where supported so successful records are not retried unnecessarily. The handler must identify failed item IDs accurately.

## Step Functions for orchestration

Use AWS Step Functions for workflows with multiple steps, waits, branching, retries, compensation, and execution history. Do not build a long workflow by chaining opaque Lambda-to-Lambda calls.

---

# 13. Networking and VPC

By default, Lambda can access public AWS service endpoints through its managed network path. Place a function in a VPC when it must reach private resources such as a private database or internal service.

```text
Lambda ENIs -> private subnets -> NAT Gateway -> public services
                       |
                       +-> VPC endpoints -> AWS services
                       |
                       +-> private database
```

Important considerations:

- Private subnets need appropriate routes and security groups.
- A function in private subnets may need NAT or VPC endpoints for outbound AWS API calls.
- NAT Gateways add cost and can become a bottleneck.
- VPC attachment can increase cold-start complexity.
- Security groups should allow only required traffic.
- Database connection limits and pooling must account for Lambda concurrency.

Do not put every Lambda in a VPC by default. Decide based on actual network requirements.

For RDS or other relational databases, consider RDS Proxy or an appropriate data-access architecture to protect connection limits.

---

# 14. Security checklist

- Use separate IAM roles for deployment and execution.
- Grant least privilege by action, resource, and condition.
- Validate and constrain all event input.
- Authenticate and authorize API requests.
- Store secrets in Secrets Manager or Parameter Store.
- Encrypt data in transit and at rest.
- Use customer-managed KMS keys only when their operational ownership is justified.
- Restrict S3 buckets and block unintended public access.
- Add resource policies only for known principals.
- Keep dependencies patched and scan them.
- Do not log passwords, tokens, full payment data, or secrets.
- Configure CloudTrail and review sensitive API activity.
- Use AWS WAF with API Gateway or CloudFront where appropriate.
- Set reserved concurrency to contain abusive or runaway workloads.
- Use organization SCPs and separate production accounts for stronger boundaries.

Lambda code is not automatically secure because the service is managed. IAM, input handling, dependencies, and downstream services remain your responsibility.

---

# 15. Observability and operations

## Logs

Use structured JSON logs with a request ID, correlation ID, operation, outcome, and duration.

```js
console.log(JSON.stringify({
  level: "info",
  event: "order.created",
  orderId,
  requestId: context.awsRequestId,
}));
```

Set retention on log groups. Avoid logging unbounded payloads.

## Metrics

Monitor:

- Invocations
- Errors
- Throttles
- Duration and p95/p99 latency
- Concurrent executions
- Iterator age for stream sources
- SQS approximate age of oldest message
- DLQ message count
- API 4xx and 5xx responses

## Tracing

AWS X-Ray or OpenTelemetry-compatible instrumentation can show service-to-service latency and errors. Propagate correlation IDs across queues and events.

## Alarms

An alarm should have an action: notify, open an incident, stop deployment, or scale a dependent resource. Alarm on symptoms that matter to users, not only on function invocation count.

---

# 16. Performance and cost optimization

## Cold starts

A cold start includes runtime initialization and module-level code. Reduce it by:

- Keeping deployment packages small.
- Removing unused dependencies.
- Initializing clients outside the handler.
- Choosing an appropriate runtime and architecture.
- Using provisioned concurrency for justified latency requirements.
- Avoiding unnecessary VPC attachment.

Do not optimize cold starts before measuring real latency.

## Memory tuning

Run a representative benchmark at multiple memory sizes. More memory can provide more CPU and finish faster, sometimes reducing total cost.

## Concurrency control

Concurrency affects cost and downstream load. Apply reserved concurrency or queue-based buffering when a database, partner API, or rate-limited service cannot handle unlimited parallelism.

## Cost formula

At a high level:

```text
cost ≈ request charges + (invocations × duration × configured memory rate)
       + related services (API Gateway, logs, NAT, queues, storage, databases)
```

Measure the entire architecture. A cheap Lambda attached to an expensive NAT path is not necessarily a cheap system.

---

# 17. Advanced Lambda patterns

## Lambda layers

Layers package shared libraries or extensions. Use them for genuinely shared, stable dependencies; do not use layers to hide application ownership or create difficult version coupling.

## Lambda extensions

Extensions can provide telemetry, configuration, or security integrations. Test their startup and shutdown behavior because they consume time and resources.

## Lambda SnapStart

Where supported by the runtime and Region, SnapStart can reduce startup latency by restoring an initialized snapshot. Review randomness, network connections, credentials, and other state that must be refreshed after restore.

## Response streaming

For supported integrations and runtimes, response streaming can send data progressively. Confirm the API integration, timeout, buffering, and client behavior before adopting it.

## Event filtering

Filter events at the event source mapping or EventBridge rule when possible. This reduces unnecessary invocations and simplifies handlers.

## Durable workflows

Use Step Functions for durable orchestration, wait states, retries, parallel branches, and human approval. Use EventBridge for routing and loose coupling. Use SQS for buffering and worker-style processing.

## Multi-Region

Multi-Region Lambda is an architecture, not a checkbox. Design:

- DNS or global routing and health checks.
- Replication and conflict resolution for data.
- Regional queues and event routing.
- Secrets and KMS key availability.
- Deployment promotion and rollback.
- Observability across Regions.

---

# 18. Testing strategy

## Unit tests

Test validation, domain logic, error mapping, idempotency, and event parsing without requiring AWS.

## Contract tests

Validate API response shapes and event schemas between producers and consumers.

## Integration tests

Run against real AWS resources in a dedicated account or isolated stack. Verify IAM, encryption, network paths, retries, and downstream behavior.

## End-to-end tests

Exercise the complete user flow through the public entry point. Keep these focused because they are slower and more expensive.

## Failure tests

Test timeouts, throttling, duplicate events, malformed payloads, downstream errors, partial batch failure, DLQs, and deployment rollback.

---

# 19. Troubleshooting guide

## `Task timed out`

Check downstream latency, network routes, DNS, database connections, and the function timeout. Do not only increase the timeout without identifying the slow operation.

## `AccessDeniedException`

Identify the caller identity, action, and resource. Check the execution role, resource policy, permissions boundary, SCP, Region, and encryption key policy.

## `Unable to import module`

Check the handler path, package format, dependency installation, runtime version, and architecture-specific native modules.

## High throttles

Check account and function concurrency, reserved concurrency, event source settings, and downstream capacity. Buffer with a queue where appropriate.

## Duplicate processing

Implement idempotency and inspect retry, visibility timeout, batch failure, and client retry settings.

## Function cannot reach a service from a VPC

Check subnet routes, NAT or VPC endpoints, security groups, network ACLs, DNS, and the target service policy.

---

# 20. Interview questions and concise answers

## What is a Lambda cold start?

The initialization latency when AWS creates a new execution environment before running the handler. Warm reuse can avoid most initialization, but it is not guaranteed.

## Is Lambda serverless?

It is serverless from the customer’s infrastructure-management perspective. Servers still exist; AWS operates them.

## What happens when a Lambda function fails?

The result depends on invocation type. Synchronous callers receive an error. Asynchronous invocations can be retried and routed to a destination. Event source mappings retry records or batches according to source configuration.

## How do you prevent duplicate side effects?

Make the operation idempotent using a durable idempotency key, conditional writes, and safe retry behavior.

## When should you use SQS before Lambda?

When you need buffering, controlled concurrency, asynchronous work, retry isolation, or protection for a slower downstream service.

## Why can putting Lambda in a VPC cause problems?

It adds network routing and dependency requirements and may increase startup complexity. It is necessary only when private network access is required.

## How do you deploy Lambda safely?

Use IaC, immutable versions, aliases, environment-specific roles, automated tests, CloudWatch alarms, and canary or linear traffic shifting with rollback.

## How do you implement Lambda CI/CD?

A CI pipeline validates and tests code; a deployment job authenticates with AWS using OIDC, builds an artifact, deploys the IaC stack, runs smoke tests, and promotes through environments using approvals and alarms.

---

# 21. Production checklist

## Application

- [ ] Handler validates event shape and input.
- [ ] Business operations are idempotent.
- [ ] Errors are classified as retryable or permanent.
- [ ] SDK clients are reused safely.
- [ ] Dependencies are locked and scanned.

## Infrastructure

- [ ] Resources are defined in IaC.
- [ ] Execution and deployment roles are separate.
- [ ] Timeouts, memory, concurrency, and architecture are measured.
- [ ] Queues have visibility timeouts and DLQs.
- [ ] Data has the required backup, retention, and encryption settings.

## Operations

- [ ] Structured logs and retention are configured.
- [ ] Metrics, traces, and alarms have owners.
- [ ] Dashboards show errors, latency, throttles, and backlog.
- [ ] Runbooks cover rollback and common failures.
- [ ] A smoke test runs after deployment.

## CI/CD

- [ ] Pull requests run tests and template validation.
- [ ] CI uses short-lived OIDC credentials.
- [ ] Production requires an approval or equivalent control.
- [ ] Deployments use versions and aliases.
- [ ] Alarm-driven rollback is tested.
- [ ] The deployment can be repeated safely.

---

# 22. Summary

AWS Lambda is most effective when a system is decomposed into bounded, event-driven operations. The function code is only one part of the solution. A production design must also define event contracts, IAM, retries, idempotency, concurrency, networking, observability, cost controls, and deployment rollback.

A strong default approach is:

1. Start with a small handler and an explicit event contract.
2. Keep state in managed durable services, not in the execution environment.
3. Use queues and workflow services to control asynchronous work.
4. Define infrastructure with SAM, CDK, or another IaC tool.
5. Deploy through CI/CD with OIDC and least-privilege roles.
6. Release immutable versions through aliases and alarm-backed traffic shifting.
7. Measure latency, failure rate, backlog, concurrency, and total architecture cost.
