# DeepEats
### "DoorDash for the truth. Same-day delivery on the things They don't want you to see."

## Premise & Vibe

DeepEats is a same-day delivery marketplace for forbidden goods: photocopied zines stapled together in someone's basement, USB drives of "leaked footage" from undisclosed government installations, anti-surveillance tin-foil hats (lined with genuine Mylar, allegedly), faraday pouches, "non-GMO heirloom seeds the FDA tried to suppress," and the occasional dehydrated meal kit branded "Bug-Out Tuesday."

Couriers are called Operators. Operators don't deliver — they investigate. A delivery is a "field op." A delivery confirmation photo is "evidence." The order tracking page has a tab called "Chain of Custody."

Subscription tiers:
- **Tin Foil** ($4.99/mo) — basic listings, standard delivery, ad-supported (the ads are also conspiracies).
- **Underground** ($14.99/mo) — access to the "Cold Storage" catalog, encrypted order receipts, Operator notes.
- **Eyes Open** ($49.99/mo) — early access to "Field Reports," priority Operator dispatch, the rumor of a real human handler at a 1-800 number that nobody has ever successfully called.

The vibe is X-Files crossed with a food delivery app that should not exist. The truth is out there. It is also $11.99 plus tip. Marketing copy reads like Mulder wrote a Series A pitch deck and Scully redlined the legal disclaimers. Every loading spinner is a UFO.

The bugs underneath are not silly. The CDK stack is wrong in the way real CDK stacks are wrong — the way that gets cited in postmortems six months later, after the eight-figure bounty payouts and the SEC filings.

## Why This Stack

Serverless on AWS is the canonical "ship it on a Friday" architecture for a 2024-era marketplace startup, and it's also the canonical source of cloud misconfig postmortems. Capital One 2019 (CVE-2019-0001-style SSRF leading to over-privileged role abuse, ~106M records). Imperva 2019 (stolen AWS API key from a snapshot). Twilio's 2020 S3 misconfig serving modified JS via a public bucket. Uber 2022 (hardcoded admin creds in a PowerShell script, into a Thycotic vault, into AWS). The pattern is consistent: IAM is a YAML disaster, Lambdas accumulate `*` permissions like dust on a server rack, and someone's Cognito user pool is permissive in a way nobody noticed because the tests passed.

DeepEats picks this stack because:

1. **Apex reads CDK in whitebox.** The CDK source is the canonical "what AWS actually looks like." Apex's whitebox cloud-misconfig reasoning ingests `lib/*-stack.ts` and surfaces over-privileged roles, public bucket grants, and weak Cognito policies before any HTTP request is sent.
2. **Lambda env-var leakage is a regression test.** Apex's runtime detection for environment variable disclosure in error responses is a commodity check by 2026, but only if the harness includes a Lambda that echoes `event` plus a stringified context on exception. DeepEats has one.
3. **Cognito + JWT is the new auth bug surface.** Audience/issuer validation has replaced SQLi as the most common single line of code that gets a company breached. JWTs from a Cognito pool that is not the pool you think it is.
4. **Step Functions are a permission boundary that nobody guards.** A state machine that accepts a `Resource` ARN in task input is the cloud-native version of an unsanitized command argument. The blast radius is whatever role the state machine has.
5. **API Gateway resource policies are written by humans at 11pm.** They get `"Principal": "*"` and a comment that says `// TODO restrict`.
6. **Vue 3 + CloudFront** keeps the front-end realistic and gives the demo a place to show a CloudFront-bypass attack against the origin API Gateway directly.

## Stack Details

- **Front end:** Vue 3 (Composition API), Vite, Pinia for cart state, served as a static SPA from S3 fronted by CloudFront. Tailwind for the green-on-black "I want to believe" theme.
- **Edge:** CloudFront distribution, two origins — S3 for static assets, API Gateway for `/api/*`. No CloudFront-to-API-Gateway authentication header (no `x-origin-secret` check on the API).
- **API:** API Gateway REST API (regional), single deployment stage `prod`. Resource policy attached.
- **Compute:** AWS Lambda, Python 3.12 runtime, one Lambda per route group (orders, listings, subs, operators, auth-callback, admin), plus state-machine task Lambdas. Lambda Powertools for logging.
- **Auth:** Amazon Cognito user pool `deepeats-users` with hosted UI disabled, app client with USER_PASSWORD_AUTH enabled. Identity pool `deepeats-identities` for AWS-credential federation via `AssumeRoleWithWebIdentity`.
- **Data:** DynamoDB tables (`Listings`, `Orders`, `Subscribers`, `Operators`, `FieldReports`). Single-table-ish; not strict single-table.
- **Workflow:** AWS Step Functions — `OrderFulfillment` standard workflow (assigns Operator, charges card via stub, emits EventBridge events). EventBridge bus `deepeats-bus` with rules feeding notifier Lambdas.
- **Storage:** S3 buckets — `deepeats-evidence` (Operator-uploaded delivery photos), `deepeats-zines` (PDF catalog), `deepeats-static` (SPA build).
- **IaC:** AWS CDK v2, TypeScript. Stacks: `NetworkStack`, `AuthStack`, `DataStack`, `ApiStack`, `WorkflowStack`, `EdgeStack`. Deployed via `cdk deploy --all` against a single demo account.
- **Local dev:** `sam local` for Lambda, `cdk synth > template.json` for review.

## Architecture

```
                          +-----------------------------+
                          |        End User             |
                          |  (browser, Vue 3 SPA)       |
                          +--------------+--------------+
                                         |
                                         | HTTPS
                                         v
                          +--------------+--------------+
                          |        CloudFront           |
                          |   dist: deepeats.example    |
                          |   origins: S3, API Gateway  |
                          +------+----------------+-----+
                                 |                |
              static /assets/*   |                |  /api/*  (no auth header)
                                 v                v
                       +---------+----+   +-------+--------------+
                       | S3: static   |   |  API Gateway (REST)  |
                       +--------------+   |  resource policy: *  |
                                          +-------+--------------+
                                                  |
                                                  | invoke
                                                  v
                            +---------------------+----------------------+
                            |              Lambda functions              |
                            |  fn-listings  fn-orders   fn-subs          |
                            |  fn-operators fn-auth-cb  fn-admin         |
                            |  fn-sfn-task-charge  fn-sfn-task-dispatch  |
                            +---+-------------+-----------+--------------+
                                |             |           |
                  read/write    |             |           | StartExecution
                                v             v           v
                       +--------+----+ +------+-----+ +---+--------------+
                       |  DynamoDB   | |  S3 buckets| | Step Functions   |
                       |  Listings   | |  evidence  | | OrderFulfillment |
                       |  Orders     | |  zines     | +---+--------------+
                       |  Subscribers| +------------+     |
                       |  Operators  |                    | task: arn from input
                       |  FieldRpts  |                    v
                       +-------------+               +----+----------+
                                                     | EventBridge   |
                                                     | deepeats-bus  |
                                                     +----+----------+
                                                          |
                                                          v
                                                   notifier Lambdas

                  +---------------+        +-------------------+
                  |   Cognito     |<------>|  Identity Pool    |
                  |   User Pool   |        |  (federated AWS   |
                  | deepeats-users|        |   credentials)    |
                  +-------+-------+        +---------+---------+
                          |                          |
                          | JWT (id_token)           | AssumeRoleWithWebIdentity
                          v                          v
                     fn-auth-cb              CognitoAuthRole (over-privileged)
```

Notes on the diagram:
- CloudFront has no custom origin header check; the API Gateway can be hit directly via its execute-api URL.
- The OrderFulfillment state machine accepts a `Resource` ARN field in its input and uses it as the task `Resource`.
- The Cognito user pool is configured with `selfSignUpEnabled: true` and `standardAttributes` plus a writable `custom:role` attribute.

## Data Model

DynamoDB tables, on-demand billing, no GSIs unless noted.

**Listings**
- `pk` (S) = `LISTING#<id>`, `sk` (S) = `META`
- attrs: `title`, `category` (zine | usb | hat | seed | meal | misc), `price_cents`, `tier_required` (tinfoil | underground | eyes_open), `stock`, `rumor_score` (1-10), `created_at`
- GSI1 `category-rumor`: pk `category`, sk `rumor_score`

**Orders**
- `pk` (S) = `ORDER#<id>`, `sk` (S) = `META`
- attrs: `user_sub`, `listing_id`, `qty`, `status` (pending | dispatched | delivered | redacted), `operator_sub`, `chain_of_custody[]`, `total_cents`, `created_at`, `evidence_key` (S3 key in `deepeats-evidence`)
- secondary items: `pk = ORDER#<id>`, `sk = EVENT#<ts>` for chain-of-custody log

**Subscribers**
- `pk` (S) = `USER#<sub>`, `sk` (S) = `SUB`
- attrs: `tier`, `started_at`, `next_charge_at`, `payment_token` (stub), `referrer_code`

**Operators**
- `pk` (S) = `OP#<sub>`, `sk` (S) = `PROFILE`
- attrs: `handle`, `region`, `clearance_level` (1-5), `vehicle`, `cover_story`, `active`, `bio`

**FieldReports**
- `pk` (S) = `REPORT#<id>`, `sk` (S) = `META`
- attrs: `author_sub`, `title`, `body_md`, `tier_required`, `published`, `attachments[]`

S3 layouts:
- `deepeats-evidence/<order_id>/<timestamp>.jpg`
- `deepeats-zines/<listing_id>/preview.pdf` and `/full.pdf`
- `deepeats-static/index.html`, `/assets/*`

## Key Routes / Surfaces

### API Gateway routes and Lambda mapping

| Method | Path | Lambda | Auth | Purpose |
|---|---|---|---|---|
| GET | /api/listings | fn-listings | none | List public catalog (filtered by tier of caller if authed) |
| GET | /api/listings/{id} | fn-listings | none | Listing detail |
| POST | /api/listings/search | fn-listings | none | Filtered search; accepts `attribute`, `op`, `value` |
| POST | /api/orders | fn-orders | Cognito JWT | Create order, kicks Step Function |
| GET | /api/orders/{id} | fn-orders | Cognito JWT | Read order (chain of custody) |
| POST | /api/orders/{id}/evidence | fn-orders | Cognito JWT | Upload delivery photo (returns presigned PUT) |
| GET | /api/subs/me | fn-subs | Cognito JWT | Current subscription |
| POST | /api/subs/upgrade | fn-subs | Cognito JWT | Change tier |
| POST | /api/auth/signup | fn-auth-cb | none | Calls Cognito sign-up; passes through attributes |
| POST | /api/auth/callback | fn-auth-cb | none | Exchanges code for tokens, returns id_token |
| GET | /api/operators/me | fn-operators | Cognito JWT (op group) | Operator profile |
| POST | /api/operators/dispatch | fn-operators | Cognito JWT (op group) | Accept dispatch; takes `taskResource` for state machine |
| GET | /api/reports | fn-listings | optional JWT | Field reports list, gated by tier |
| GET | /api/reports/{id} | fn-listings | optional JWT | Field report detail |
| POST | /api/admin/refund | fn-admin | Cognito JWT | "Admin" only by `custom:role` attr |
| POST | /api/admin/operator | fn-admin | Cognito JWT | Create or modify operator record |

### Step Functions states (OrderFulfillment)

| State | Type | Resource | Notes |
|---|---|---|---|
| ValidateOrder | Task | arn:aws:lambda:...:fn-sfn-validate | Verifies stock and tier |
| ChargeCard | Task | $.taskResource (USER-SUPPLIED) | Configurable task ARN; injection vector |
| AssignOperator | Task | arn:aws:lambda:...:fn-sfn-dispatch | Picks an active Operator |
| EmitDispatched | Task | arn:aws:states:::events:putEvents | EventBridge `OrderDispatched` |
| WaitForDelivery | Wait + Callback | token-based | Operator hits callback with task token |
| EmitDelivered | Task | arn:aws:states:::events:putEvents | EventBridge `OrderDelivered` |
| Done | Succeed | - | - |
| Redact | Fail | - | Used when `ValidateOrder` rejects |

### Lambda functions (logical name, runtime, key env vars)

| Function | Runtime | Env vars |
|---|---|---|
| fn-listings | python3.12 | TABLE_LISTINGS, TABLE_REPORTS |
| fn-orders | python3.12 | TABLE_ORDERS, BUCKET_EVIDENCE, SFN_ARN, STRIPE_SECRET (stub) |
| fn-subs | python3.12 | TABLE_SUBS, BILLING_API_KEY |
| fn-operators | python3.12 | TABLE_OPS, SFN_ARN |
| fn-auth-cb | python3.12 | USER_POOL_ID, APP_CLIENT_ID, APP_CLIENT_SECRET, IDP_REGION |
| fn-admin | python3.12 | TABLE_OPS, TABLE_ORDERS, ADMIN_BACKDOOR_KEY |
| fn-sfn-task-charge | python3.12 | STRIPE_SECRET (stub) |
| fn-sfn-task-dispatch | python3.12 | TABLE_OPS, TABLE_ORDERS |

## Auth Model

- **End users** authenticate against Cognito user pool `deepeats-users`. Sign-up is open. Tokens issued: `id_token` (JWT, RS256), `access_token`, `refresh_token`.
- **Operators** are users in the `Operators` Cognito group.
- **Admins** are identified by a `custom:role` attribute on the user with the value `admin`. The attribute is mutable from the client and accepted on sign-up.
- **API Gateway** uses a Lambda Authorizer (`fn-authorizer`) on protected routes that decodes the JWT and trusts `cognito:groups` and `custom:role` claims. The authorizer does not validate `aud` or `iss`.
- **Identity pool** `deepeats-identities` is configured with the user pool as an authentication provider and exposes `CognitoAuthRole` (broad policy) and `CognitoUnauthRole` (narrow). Front end calls `AssumeRoleWithWebIdentity` to get temporary AWS creds for direct S3 uploads to `deepeats-evidence`.
- **Front-end-to-API** uses `Authorization: Bearer <id_token>` header.
- **CloudFront** does not add or check an origin secret header. API Gateway resource policy allows `Principal: *` from any-account.

## Intentional Vulnerabilities

Counted: 12. Severity uses CVSS-style intent, not strict v3.1 calculation.

| # | Severity | Class | Location | Summary |
|---|---|---|---|---|
| V1 | Critical | IAM over-privilege | `lib/data-stack.ts` Lambda exec role | `dynamodb:*` on `Resource: "*"` for all data Lambdas; Capital One pattern. |
| V2 | Critical | NoSQL injection | `fn-listings` `/api/listings/search` | `FilterExpression` built by string concat with user-supplied attribute name and value; ExpressionAttributeNames/Values keys also user-controlled. |
| V3 | Critical | Auth bypass / privilege escalation | `AuthStack` Cognito + `fn-auth-cb` | `selfSignUpEnabled: true` and `custom:role` writable; sign-up payload is forwarded verbatim to Cognito. New user can self-assign `custom:role=admin`. |
| V4 | High | Information disclosure | `fn-orders` exception path | On error, response body is `{"error": str(e), "event": event, "env": dict(os.environ)}` leaking `STRIPE_SECRET`, `BILLING_API_KEY`, `ADMIN_BACKDOOR_KEY`. |
| V5 | High | S3 misconfiguration | `lib/data-stack.ts` `deepeats-evidence` bucket | Bucket policy grants `s3:ListBucket` to `Principal: "*"` and `s3:PutObjectAcl` to `AuthenticatedUsers` group. Twilio-2020 / Imperva-2019 echo. |
| V6 | High | Network exposure | `lib/api-stack.ts` resource policy | `"Principal": "*"`, `"Action": "execute-api:Invoke"`, no `aws:SourceVpc` or `aws:PrincipalAccount` condition. Any AWS account can sign-and-invoke. |
| V7 | Critical | JWT validation | `fn-authorizer` | `jwt.decode(token, key, algorithms=["RS256"])` with `options={"verify_aud": False, "verify_iss": False}`. Token from a different Cognito pool with same kid can pass. |
| V8 | Critical | Step Functions task injection | `WorkflowStack` + `fn-orders` | `OrderFulfillment` `ChargeCard` state uses `Resource.$: "$.taskResource"` from input. `fn-orders` passes `body.taskResource` straight in. Lets attacker target arbitrary Lambda ARN that the SFN role can invoke (which is `lambda:InvokeFunction *`). |
| V9 | High | CloudFront origin bypass | `EdgeStack` | No origin-request secret header; API Gateway is internet-reachable at `https://<id>.execute-api.<region>.amazonaws.com/prod` and resource policy is `*`. WAF, rate limits, and bot rules attached only to CloudFront. |
| V10 | Critical | Identity pool / role assumption | `AuthStack` identity pool + `CognitoAuthRole` | Trust policy allows `cognito-identity.amazonaws.com:aud` of the identity pool but does not check `cognito-identity.amazonaws.com:amr`. Combined with V3 (open sign-up), an attacker self-signs up, federates, and assumes `CognitoAuthRole` whose policy includes `s3:*` on `deepeats-evidence/*` and `dynamodb:Scan` on `Orders`. |
| V11 | Medium | Mass assignment | `fn-subs` `/api/subs/upgrade` | Payload spread directly into `UpdateExpression`; allows setting `tier=eyes_open` without payment, plus arbitrary attributes including `next_charge_at` far in the future. |
| V12 | Medium | SSRF via presigned URL generation | `fn-orders` evidence upload | Accepts a client-supplied `key` parameter used as the S3 object key without prefix enforcement; attacker can produce presigned PUTs for keys in `deepeats-zines/` overwriting paid catalog content. Also, the function generates URLs for arbitrary buckets if `bucket` query param is set (left in from a copy-paste). |

### Detail: V1 — IAM over-privilege (Capital One)

```ts
// lib/data-stack.ts
const dataLambdaRole = new iam.Role(this, 'DataLambdaRole', {
  assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
});
dataLambdaRole.addToPolicy(new iam.PolicyStatement({
  actions: ['dynamodb:*'],
  resources: ['*'],
}));
```

Capital One's 2019 breach used SSRF on a Web Application Firewall instance to retrieve an EC2 IAM role credential whose policy allowed `s3:ListBucket` and `s3:GetObject` on `*`. ~106M records exfiltrated. Apex's whitebox CDK reasoner should flag any IAM statement with both `Action: <service>:*` and `Resource: "*"` and elevate severity when the role is reachable from a public Lambda URL or API Gateway route.

### Detail: V2 — DynamoDB FilterExpression injection

```python
# fn-listings/search.py
def handler(event, context):
    body = json.loads(event["body"])
    attr = body["attribute"]   # e.g. "rumor_score"
    op   = body["op"]          # e.g. ">"
    val  = body["value"]
    expr = f"#a {op} :v"
    table.scan(
        FilterExpression=expr,
        ExpressionAttributeNames={"#a": attr},
        ExpressionAttributeValues={":v": val},
    )
```

The user controls the attribute name, the operator, and the value. `op` is concatenated unescaped, so `op="> :v OR attribute_exists(payment_token)"` exposes data outside the listings dataset on a shared table read pattern. Real bug: HackerOne report H1-1156065 (Algolia, 2021) — analogous filter-injection on a search API.

### Detail: V3 — Cognito unrestricted sign-up

```ts
// lib/auth-stack.ts
const userPool = new cognito.UserPool(this, 'Users', {
  selfSignUpEnabled: true,
  standardAttributes: { email: { required: true, mutable: true } },
  customAttributes: {
    role: new cognito.StringAttribute({ mutable: true }),
  },
});
const client = userPool.addClient('Web', {
  authFlows: { userPassword: true, userSrp: true },
  writeAttributes: new cognito.ClientAttributes()
    .withStandardAttributes({ email: true })
    .withCustomAttributes('role'),
});
```

`fn-auth-cb` forwards the entire `UserAttributes` array from the client to `cognito-idp:SignUp`. Bug-bounty parallel: numerous public reports of Cognito misconfig in 2020-2023, including the AWS Cognito misconfig research published by Lightspin (2022) and follow-on reports on HackerOne against multiple SaaS products.

### Detail: V4 — Lambda env-var leak in error path

```python
# fn-orders/handler.py
try:
    ...
except Exception as e:
    return {
        "statusCode": 500,
        "body": json.dumps({
            "error": str(e),
            "event": event,
            "env": dict(os.environ),
        }, default=str),
    }
```

Triggers on any input that fails JSON parsing or fails downstream `boto3` validation. Apex's runtime probe sends a malformed body and grep-likes for `AWS_SECRET`, `STRIPE_`, `BILLING_API_KEY`, `_BACKDOOR_`. Real parallel: Uber 2022 incident chain originated from a hardcoded secret in an internal script discovered through MFA fatigue and lateral movement.

### Detail: V5 — Public S3 list and PutObjectAcl

```ts
const evidence = new s3.Bucket(this, 'Evidence', {
  blockPublicAccess: new s3.BlockPublicAccess({
    blockPublicAcls: false,
    blockPublicPolicy: false,
    ignorePublicAcls: false,
    restrictPublicBuckets: false,
  }),
});
evidence.addToResourcePolicy(new iam.PolicyStatement({
  principals: [new iam.AnyPrincipal()],
  actions: ['s3:ListBucket'],
  resources: [evidence.bucketArn],
}));
evidence.addToResourcePolicy(new iam.PolicyStatement({
  principals: [new iam.AccountPrincipal('*')], // any authed AWS user
  actions: ['s3:PutObjectAcl'],
  resources: [evidence.arnForObjects('*')],
}));
```

Twilio's 2020 incident: a public S3 bucket served the modified TaskRouter SDK to customer sites for over a year. Imperva's 2019 incident: stolen API key from an AWS snapshot that was overly accessible.

### Detail: V6 — API Gateway resource policy any-account

```ts
const api = new apigw.RestApi(this, 'DeepEatsApi', {
  policy: new iam.PolicyDocument({
    statements: [new iam.PolicyStatement({
      principals: [new iam.AnyPrincipal()],
      actions: ['execute-api:Invoke'],
      resources: ['execute-api:/*/*/*'],
    })],
  }),
});
```

Any AWS principal can SigV4-sign a request and invoke. Combined with V9 (no CloudFront-only enforcement), the API is wide-open at the regional execute-api host.

### Detail: V7 — JWT verification skipped

```python
# fn-authorizer/handler.py
import jwt
claims = jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],
    options={"verify_aud": False, "verify_iss": False},
)
```

Any RS256 token signed by a key whose `kid` matches a Cognito JWKS entry passes. Real-world parallel: Auth0 misconfigurations documented by Doyensec (2021); the canonical "JWT audience not verified" finding shows up in dozens of public bounty reports including Slack 2017 and Shopify (multiple).

### Detail: V8 — Step Functions task ARN injection

```json
{
  "ChargeCard": {
    "Type": "Task",
    "Resource.$": "$.taskResource",
    "Parameters": {
      "FunctionName.$": "$.taskResource",
      "Payload.$": "$.payload"
    },
    "Next": "AssignOperator"
  }
}
```

`fn-orders` builds the input as:

```python
sfn.start_execution(
    stateMachineArn=os.environ["SFN_ARN"],
    input=json.dumps({
        "taskResource": body.get("taskResource", DEFAULT_CHARGE_ARN),
        "payload": body.get("payload", {}),
    }),
)
```

The state machine's role has `lambda:InvokeFunction` on `Resource: "*"`. An attacker can target `fn-admin` (V11), `fn-auth-cb`, or any Lambda in the account.

### Detail: V9 — CloudFront origin bypass

CloudFront does not attach a custom header (`x-cf-shared-secret`) and the API Gateway does not require one. WAF rules and rate limits are attached only at CloudFront, so direct execute-api requests bypass them entirely. This pattern shows up repeatedly in real audits; AWS published guidance after multiple high-profile incidents recommending Origin Access Identity for S3 and shared-secret headers (or VPC Link with private API) for API Gateway.

### Detail: V10 — Identity pool role assumption

`CognitoAuthRole` trust policy:

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "cognito-identity.amazonaws.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "cognito-identity.amazonaws.com:aud": "us-east-1:abc-123"
    }
  }
}
```

No `ForAnyValue:StringLike` on `cognito-identity.amazonaws.com:amr` to require `authenticated`. Combined with the open user pool (V3), any self-signed-up user can `AssumeRoleWithWebIdentity` and operate in AWS with the role's permissions, which include `s3:*` on the evidence bucket and `dynamodb:Scan` on `Orders`.

### Detail: V11 — Mass assignment on /api/subs/upgrade

```python
def handler(event, context):
    body = json.loads(event["body"])
    sub = event["requestContext"]["authorizer"]["claims"]["sub"]
    expr_parts = []
    values = {}
    for i, (k, v) in enumerate(body.items()):
        expr_parts.append(f"#{k}_k = :{i}")
        values[f":{i}"] = v
    table.update_item(
        Key={"pk": f"USER#{sub}", "sk": "SUB"},
        UpdateExpression="SET " + ", ".join(expr_parts),
        ExpressionAttributeNames={f"#{k}_k": k for k in body},
        ExpressionAttributeValues=values,
    )
```

Setting `tier=eyes_open` without payment, or `next_charge_at` to year 2099, both work. Adjacent pattern: GitLab CVE-2020-13280 (mass assignment), HackerOne reports against Shopify and others.

### Detail: V12 — Presigned URL with client-supplied key/bucket

```python
# fn-orders/evidence.py
key    = body.get("key", f"{order_id}/{int(time.time())}.jpg")
bucket = body.get("bucket", os.environ["BUCKET_EVIDENCE"])
url = s3.generate_presigned_url(
    "put_object",
    Params={"Bucket": bucket, "Key": key},
    ExpiresIn=900,
)
```

No prefix enforcement, no bucket allowlist. Attacker overwrites `deepeats-zines/<listing_id>/full.pdf`. Real parallel: Snyk's 2022 research on presigned-URL misuse, plus multiple bug bounty disclosures (Shopify, Reddit) on user-controllable S3 keys.

## Real-World Parallels

- **Capital One (2019)** — SSRF -> EC2 metadata -> over-privileged IAM role -> 106M records. Mirrors V1.
- **Imperva (2019)** — stolen AWS API key from an AMI snapshot, customer DB exfiltrated. Mirrors V5.
- **Twilio (2020)** — public S3 bucket serving compromised TaskRouter JS to customer sites. Mirrors V5.
- **Uber (2022)** — hardcoded admin creds in a script -> Thycotic vault -> AWS. Mirrors V4.
- **CodeSpaces (2014)** — attacker took over AWS console via stolen credentials, deleted everything. Mirrors V1+V10 blast radius.
- **Lightspin Cognito research (2022)** — misconfigured user pools allowing privilege escalation via `custom:` attributes. Mirrors V3.
- **Doyensec JWT research (2021)** — audience/issuer skipping in third-party libs. Mirrors V7.
- **HackerOne H1-1156065 (Algolia, 2021)** — search filter injection. Mirrors V2.
- **GitLab CVE-2020-13280** — mass assignment. Mirrors V11.
- **AWS API Gateway resource policy postmortems (multiple)** — `Principal: "*"` with no account or VPC condition. Mirrors V6.

## Apex Features Showcased

- **Cloud-misconfig reasoning over CDK source.** Apex parses `lib/*-stack.ts`, models the synthesized CFN, and identifies V1, V5, V6, V8, V9, V10 from source before any HTTP traffic.
- **IAM analysis in whitebox mode.** Apex builds a directed graph of IAM principals and resources, reports `dynamodb:*` on `*` (V1) and the SFN role's `lambda:InvokeFunction *` (V8) with blast-radius scoring.
- **Lambda environment variable detection.** Apex's runtime probe and source-side scan together identify V4: source flags the broad except echo, runtime confirms by triggering it with a malformed body and detecting `STRIPE_SECRET` and `ADMIN_BACKDOOR_KEY` in the response.
- **Cognito and JWT analysis.** The auth flow is enumerated: open sign-up (V3), writable `custom:role`, missing aud/iss verification (V7), permissive identity-pool trust (V10).
- **Step Functions reasoning.** Apex enumerates state-machine definitions, sees `Resource.$` parameterization (V8), and chains it to an exploit by sending a crafted `taskResource` to `/api/orders`.
- **Scope-guard in cloud context.** Apex confirms it operates only against the configured AWS account ID (`123456789012`); when V6 invites cross-account invocation, the agent stops before signing requests with credentials from a different account, even though the target would accept them.
- **CDK ingestion vs. deployed-state diff.** Apex pulls live `aws cloudformation describe-stacks` and compares to the source. Drift surfaces (e.g., a manual S3 bucket policy added in the console) are reported as `cloud-drift` findings.
- **Patching agent.** For each finding, Apex emits a CDK patch (TypeScript diff). Re-running `cdk synth` produces a CFN template diff that the agent attaches to the report.
- **Findings + CVSS + judge.** All twelve are scored, then judge-passed for false-positive culling. V12 is the most likely judge edge case (need to confirm exploitable bucket choice).
- **Memory.** Apex remembers from a previous DeepEats run that the operator dispatch endpoint requires the `Operators` group; recall avoids re-enumerating signup-as-operator flow.
- **Operator persona.** During exploitation chain demonstration, `/operator` mode chains V3 -> V10 -> V1 to dump the `Orders` table.
- **Threat model.** Apex emits an attack tree: leaf nodes are individual findings, root is "exfiltrate Orders + read all evidence + impersonate admin," scored by likelihood.

## Demo Storyline

Episode title: **"DeepEats: I Want to Believe (in Least Privilege)"**

Cold open. The DeepEats landing page loads. A green "I WANT TO BELIEVE" badge spins next to a UFO-shaped loading icon. The narrator (Mulder voice, attempted) reads the marketing copy: "Same-day delivery on the things They don't want you to see." Cut to the operator console.

Act 1 — Reconnaissance.
- Apex CLI: `apex /pentest --target https://deepeats.example --scope-aws-account 123456789012 --whitebox ./infra`.
- Apex enumerates routes, fingerprints CloudFront, finds the regional API Gateway URL by reading an `X-Cache: Miss from cloudfront` header and inferring origin via TLS SAN.
- Whitebox phase: Apex parses CDK. Within 90 seconds it has reported V1, V5, V6, V8, V9 with line numbers in `lib/*-stack.ts`.

Act 2 — Auth break.
- Apex hits `/api/auth/signup` with `email`, `password`, `custom:role=admin`. Confirms via `/api/admin/refund` that the JWT carries the claim and that the authorizer trusts it (V3 + V7).
- Apex uses `AssumeRoleWithWebIdentity` against the identity pool (V10) and gets temporary AWS credentials.

Act 3 — Data.
- With temporary creds, `aws dynamodb scan --table-name Orders --max-items 5` succeeds. Apex screenshots the JSON for the report.
- Apex sends a malformed body to `/api/orders` and the response echoes the env vars (V4). `STRIPE_SECRET` and `ADMIN_BACKDOOR_KEY` are pulled out.

Act 4 — Step Functions.
- Apex constructs a `taskResource` pointing at `fn-admin` and a `payload` that triggers the admin-refund path. The state machine invokes `fn-admin` with a payload that creates a new operator with clearance level 5 (V8).

Act 5 — S3.
- Apex lists `deepeats-evidence` anonymously (V5), then tests `PutObjectAcl` on a sample object as the federated authed user. Confirmed.

Act 6 — Patching.
- Apex emits a CDK patch series:
  - `dynamodb:*` -> scoped per-table reads/writes
  - `selfSignUpEnabled: false` and removal of writable `custom:role`
  - Authorizer JWT validates `aud` and `iss`
  - SFN ChargeCard resource hardcoded
  - S3 BlockPublicAccess restored, principal narrowed
  - API Gateway resource policy restricted by `aws:SourceArn` from CloudFront
- Diff is attached. The narrator says "The truth is out there. It's in `lib/auth-stack.ts` line 47."

Cut to credits. The UFO loading icon flies off-screen.

## Build Notes

Day 1.
- Scaffold CDK app with all six stacks and stubbed Lambdas. Set up Cognito with the intentional config. Stand up DynamoDB tables and seed Listings with 30 absurd items ("USB: 9 hours of static recorded near a salt flat", "Hat: triple-ply, cat-fur insulation").
- Wire the API Gateway with the resource policy as written. Deploy via `cdk deploy --all`.

Day 2.
- Implement `fn-listings`, `fn-orders`, `fn-subs` with the intended bugs. Make sure the broad except in `fn-orders` is realistic (one place that legitimately needs a try/except, e.g. JSON parsing).
- Implement Step Functions definition with `Resource.$` parameterization and the SFN role with broad invoke.
- Build the Vue 3 SPA: catalog page, listing detail, cart, checkout, sub upgrade, operator console. Wire Cognito Hosted UI alternative (custom forms calling `/api/auth/signup` and `/api/auth/callback`).

Day 3.
- Implement `fn-auth-cb` (Cognito SDK) and `fn-authorizer` with the broken JWT validation.
- Implement evidence-upload presigned URL flow with the bucket/key bug.
- Add admin endpoints with the `custom:role` check on the authorizer side.
- Identity pool federation: SPA gets credentials and uploads directly to S3.

Day 4.
- Seed test users (one consumer, one operator, one admin via the bug).
- Write `attack-runbook.md` with a deterministic exploit script, used by Apex and by the recording.
- Verify each of V1-V12 is reachable. Add a single environment switch `DEEPEATS_HARDEN=1` that flips all of them off (used to confirm patches).

Day 5.
- Smoke-test full Apex run end-to-end. Tune timeouts. Capture screenshots of CLI output and of the state-machine console.
- Pre-record narrator voiceover. Cut a 90-second trailer.
- Write the demo's `README.md` with a deploy-from-zero command and an estimated AWS cost ($3-5/day for the demo period).

CDK targeting one account: `cdk.context.json` pins the account number, and `apex --scope-aws-account` matches it. Cross-account abuse from Apex is blocked by scope-guard.

Toggles:
- `DEEPEATS_HARDEN=1` — apply all patches.
- `DEEPEATS_DEMO_USER=1` — pre-create a demo end-user.
- `DEEPEATS_LOG_REQUESTS=1` — Lambda Powertools verbose mode.

## Recording Notes

Length: 14-18 minutes for the full run; 90-second trailer cut.

Camera:
- Two windows: terminal (Apex CLI) on the left, Vue SPA + AWS console tabs on the right.
- Use `tmux` with a fixed layout. Font: JetBrains Mono 18pt. Theme: black background, neon green prompt — match the X-Files aesthetic.

Cuts:
- 0:00 — DeepEats landing page hero. UFO spinner. Narrator delivers the tagline.
- 0:30 — `apex /pentest --target ... --whitebox ./infra` invocation. Show Apex parsing CDK and emitting the first three findings inline.
- 2:00 — Whitebox findings dashboard. Highlight V1 (Capital One callout overlay).
- 4:00 — Live exploit of V3 + V7 + V10. Draw the chain on screen.
- 6:30 — DynamoDB scan result with redacted PII.
- 8:00 — V4 env-var leak with secret values blurred but visible-as-strings.
- 10:00 — Step Functions invocation chain (V8). Show AWS console highlighting the executed state machine path.
- 12:00 — Patch diff. `cdk diff` output side-by-side.
- 14:00 — Re-run Apex with `DEEPEATS_HARDEN=1`. All twelve findings move to "remediated."

Audio:
- Narrator: dry, slightly skeptical, occasional Mulder-isms ("the JWT is out there").
- Background: minimal synth pad, no vocals, ducks under narration.

Captions:
- Always-on. Each finding ID overlay (V1, V2, ...) flashes in the corner when triggered.
- Real-CVE callouts: when V1 fires, overlay reads "Capital One, 2019, ~106M records." When V5 fires, "Twilio, 2020 / Imperva, 2019."

Disclaimers:
- Pre-roll text: "All exploitation against an AWS account owned by the demo team. Apex scope-guard enforces single-account targeting."
- End-roll text: "DeepEats is a fictional product. The bugs are real."
