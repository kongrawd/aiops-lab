---
title: AWS MCP Server as an Amazon Quick (Web) OAuth connector
kb_type: how-to
confidence: validated
governance: experience
owner: k0ngrawd
reviewed_by: k0ngrawd
last_reviewed: 2026-08-24
review_by: 2027-02-24
keywords:
  - aws-mcp.us-east-1.api.aws
  - us-east-1.oauth.signin.aws
  - AWSMCPSignInOAuthAccessPolicy
  - signin:AuthorizeOAuth2Access
  - signin:CreateOAuth2Token
  - signin:OAuthRedirectUri
  - aws:ViaAWSMCPService
  - aws:CalledViaAWSMCP
  - aws:SignInSessionArn
  - aws-mcp:CallReadOnlyTool deprecated
  - CreateOAuth2PublicClient
  - CreateOAuth2TokenWithIAM
  - oauth-protected-resource
  - oauth-authorization-server
  - external-client/dcr
  - sn/oauthcallback
  - PKCE S256
  - Public OAuth client
  - HTTP 424
  - insufficient_scope
  - Authentication failed. Unable to verify your user identity
  - aws-mcp.amazonaws.com userAgent
---

# Connect the AWS MCP Server to Amazon Quick (Web) with OAuth

The [AWS MCP Server](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/) is a managed remote MCP server that lets an agent read AWS documentation and operate AWS APIs under an existing IAM identity. Amazon Quick is an MCP client that registers those tools as agent actions. This records the outcome of connecting the two: the configuration that worked, the AWS calls it produced in CloudTrail, and how to constrain them in IAM.

> **Not official documentation.** Community-maintained, not endorsed by any vendor named here, no warranty; vendor documentation is authoritative. Test in a non-production environment and run your own security review first — see [DISCLAIMER.md](../DISCLAIMER.md). The IAM policies below are **illustrative, not least-privilege**.

---

## 1. Objective

**Scope.** Connecting AWS MCP Server to Amazon Quick (Web) over OAuth: the discovery chain, where authorization is enforced, the client configuration that completed the connector, the CloudTrail audit trail it produces, and how to scope both the IAM identity the agent inherits and the MCP path itself.

**Out of scope.** The SigV4 path via the MCP Proxy for AWS — the right choice for IDE and terminal clients and for cross-account work. Concepts already covered upstream are linked, not restated ([§4.2](#42-further-reading)).

**Evidence basis.** Findings are the author's own results in one account, one Region, tagged `[field YYYY-MM-DD]`. Published documentation supplies the surrounding context, tagged `[docs 2026-08-24]`;

---

## 2. Operational context

Endpoints: `https://aws-mcp.us-east-1.api.aws/mcp` and `https://aws-mcp.eu-central-1.api.aws/mcp` — those two Regions. Remote HTTP; Quick does not support local stdio. Requires an Amazon Quick **Enterprise** subscription, listed under *Before you begin* `[docs 2026-08-24]`. `aws___call_aws` was deprecated 2026-07-15 for removal 2026-08-31; `aws___run_script` replaces it. Knowledge tools need no AWS credentials; the API tools do.

**Identity.** OAuth authenticates the human and grants no AWS permissions of its own; the API tools act within that identity's existing IAM permissions. The Quick consent screen is application-level, so the effective boundary is IAM — confirmed in [§2.1](#21-verified-findings). What IAM has to work with is the caller's own identity *plus* two condition keys the MCP server adds ([§3.3](#33-control-the-mcp-path-with-condition-keys)).

**Constraints that shape the design** `[docs 2026-08-24]`, not exercised here — full lists in [§4.2](#42-further-reading):

| Constraint | Consequence |
| --- | --- |
| 60-second fixed operation timeout | Overruns fail **HTTP 424**; poll long work with `aws___get_tasks` |
| No multi-account support on OAuth | One Quick integration per AWS account |
| No read-only *mode* on OAuth | Hiding write tools from the agent is SigV4-only; read-only can still be **enforced** in IAM ([§3.3](#33-control-the-mcp-path-with-condition-keys)) |
| No custom HTTP headers; no scope field | Neither can be added from the connector UI |
| Tool lists static after registration | Delete and recreate to pick up new tools |
| Access token 1 hour, refresh up to 12 hours | Refresh tokens are single-use, and are revoked by any IAM policy change to the principal |
| DCR registrations have a 90-day lifetime | A manually entered Client ID stops working roughly quarterly |
| DCR redirect URIs are allow-listed | Quick's callback does not appear in the published list; registration against it completed here — see [§4.1](#41-provenance) |

### 2.1 Verified findings

**Discovery chain**, as returned `[field 2026-08-24]`:

```
GET https://aws-mcp.us-east-1.api.aws/.well-known/oauth-protected-resource
  → authorization_servers = [ https://us-east-1.oauth.signin.aws/ ]

GET https://us-east-1.oauth.signin.aws/.well-known/oauth-authorization-server
  → authorization_endpoint = https://us-east-1.oauth.signin.aws/v1/authorize
  → token_endpoint         = https://us-east-1.oauth.signin.aws/v1/token
  → registration_endpoint  = https://us-east-1.oauth.signin.aws/v1/register
  → token_endpoint_auth_methods_supported = ["none"]   (public client)
  → grant_types_supported                 = ["authorization_code","refresh_token"]
  → code_challenge_methods_supported      = ["S256"]
```

These URLs are resolved from the metadata document at runtime rather than configured by hand, so treat the values as current state rather than fixed. Discovery, the `401` challenge, public-client metadata, and non-idempotent registration matched RFC 9728, 8414, 7591 and 6749 as returned. Two values differed between the observation dates:

- **`code_challenge_methods_supported`** was absent `[field 2026-07-16]` and returned `["S256"]` `[field 2026-08-24]`, so PKCE S256 is now confirmable from metadata.
- **`client_credentials`** does not appear in `grant_types_supported` on either date. That grant exists as a separate API ([§4.1](#41-provenance)).

**Authorization is enforced at `tools/call`, not at connect.** `initialize` and `tools/list` return `200` without a token; `tools/call` returns `[field 2026-07-16]`:

```
HTTP/2 401
www-authenticate: Bearer realm="https://aws-mcp.us-east-1.api.aws/mcp",
  resource_metadata="https://aws-mcp.us-east-1.api.aws/.well-known/oauth-protected-resource"
```

A connector can therefore pass creation and tool discovery in Quick and still fail on first real use. Successful tool discovery does not indicate working authorization.

**Registration against Quick's callback completed without authentication** `[field 2026-07-16]` — three consecutive calls returned two distinct `client_id`s and then HTTP 500, consistent with rate limiting.

**Server identity** — `AWSMCP` v1.0.0, MCP protocol revision `2025-06-18`, fronted by CloudFront `[field 2026-07-16]`.

**The agent ran as the caller's own identity.** `aws sts get-caller-identity` through the tool returned the user's own federated administrator role session, so the agent held that role's full privilege `[field 2026-07-16]`.

### 2.2 CloudTrail audit trail

**Observed — the downstream data-plane event.** Two `ListBuckets` calls made through the agent, recorded in the caller's own account `[field 2026-07-16]`:

| Field | Value |
| --- | --- |
| `eventName` / `eventSource` | `ListBuckets` / `s3.amazonaws.com`, `readOnly: true` |
| `userIdentity.type` | `AssumedRole` |
| `userIdentity.arn` | `arn:aws:sts::<account>:assumed-role/<Role>/<session-name>` |
| `sessionIssuer.arn` | `arn:aws:iam::<account>:role/<Role>` |
| **`sourceIPAddress`** | **`aws-mcp.amazonaws.com`** |
| **`userAgent`** | **`aws-mcp.amazonaws.com`** |

Three things follow directly:

1. **The recorded origin is an AWS service hostname, not the user's address** — verify anything that reads caller network origin against this path rather than assuming it.
2. **Execution is under the caller's own role session**, not a shared service role. IAM on that role is the boundary.
3. **Audit reaches the individual** via the role-session name, which holds where roles are assumed with per-user session names.

Two `readOnly` events, one Region, one date. The `userAgent` value is a log observation rather than a documented interface — alert on its presence, and do not read its absence as evidence of no MCP activity.

**From the documentation — the control-plane OAuth events**, `eventSource: signin.amazonaws.com` `[docs 2026-08-24]`, not captured in this environment:

| Event | Fields that matter |
| --- | --- |
| `AuthorizeOAuth2Access` | `requestParameters`: `resource` (the MCP endpoint), `redirect_uri`, `code_challenge_method` (`S256`), `client_id` (the DCR client ARN) |
| `CreateOAuth2Token` | `additionalEventData`: `grant_type`, `signInSessionArn`; also `sessionContext.signInSessionArn` |

CloudTrail also records token introspection and revocation, and captures the OAuth client, target MCP server, redirect URI, flow, and sign-in session.

**Correlation.** OAuth-issued API calls carry the `aws:SignInSessionArn` global condition key, and the same session ARN appears in `CreateOAuth2Token`, so the grant and the downstream activity join on it. That makes it both the audit join key and the fastest containment lever: a deny on it stops the session immediately, whereas revoking a refresh token leaves existing access tokens valid for up to an hour `[docs 2026-08-24]`.

To pull the authorization history:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=signin.amazonaws.com \
  --query 'Events[].{t:EventTime,n:EventName,u:Username}'
```

For downstream activity, the observed `userAgent` value `aws-mcp.amazonaws.com` is a useful log filter, with the caveat above. For policy decisions use `aws:CalledViaAWSMCP`, which is a documented condition key ([§3.3](#33-control-the-mcp-path-with-condition-keys)).

---

## 3. Resolution & mitigation

### 3.1 Prerequisites

Every end user's principal needs `signin:AuthorizeOAuth2Access` and `signin:CreateOAuth2Token`; without them, sign-in ends on a `400` page. The account root user needs no additional IAM permission `[docs 2026-08-24]`. Grant per role that will use the connector:

```bash
aws iam attach-role-policy \
  --role-name QuickMcpOperator \
  --policy-arn arn:aws:iam::aws:policy/AWSMCPSignInOAuthAccessPolicy
```

To scope it directly, allow those two actions on `arn:aws:signin:*:*:service-principal/aws-mcp.amazonaws.com`.

### 3.2 Scope a dedicated IAM role

**Executed by:** an administrator, before the first user connects.

This section follows from the observed outcome: the connector bound to the identity the user was already signed in as — a federated administrator role — and the agent inherited all of it `[field 2026-07-16]`. Create a purpose-built role, grant it only the task permissions, pin the Region with `aws:RequestedRegion` (OAuth has no session default Region), and cap it with a permissions boundary.

Then make it the only identity that can complete the grant. An SCP does this, where relying on where the policy is attached does not:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OAuthToAwsMcpOnlyFromTheDedicatedRole",
      "Effect": "Deny",
      "Action": ["signin:AuthorizeOAuth2Access", "signin:CreateOAuth2Token"],
      "Resource": "arn:aws:signin:*:*:service-principal/aws-mcp.amazonaws.com",
      "Condition": {
        "ArnNotLike": { "aws:PrincipalArn": "arn:aws:iam::*:role/QuickMcpOperator" }
      }
    },
    {
      "Sid": "PinTheRedirectUriWhenPresent",
      "Effect": "Deny",
      "Action": "signin:AuthorizeOAuth2Access",
      "Resource": "arn:aws:signin:*:*:service-principal/aws-mcp.amazonaws.com",
      "Condition": {
        "StringNotEquals": {
          "signin:OAuthRedirectUri": "https://us-east-1.quicksight.aws.amazon.com/sn/oauthcallback"
        },
        "Null": { "signin:OAuthRedirectUri": "false" }
      }
    }
  ]
}
```

Four details determine whether these statements behave as intended `[docs 2026-08-24]`:

- **`aws:PrincipalArn` for an assumed role holds the *role* ARN.** The documentation specifies the role ARN rather than the `sts` assumed-role session ARN; matching the session form denies everything.
- **The `Null` guard is load-bearing.** Inverted operators such as `StringNotEquals` match an absent key, so without it the second statement would deny requests that carry no redirect URI at all. As written it applies only when one is present and does not match.
- **`signin:PrincipalArn` does not apply here.** It is available pre-authentication, for `signin:Authenticate`, in resource-based policies and RCPs — not for these two actions.
- **SCPs apply to member accounts, including member-account root, and not to the management account.** Run the connector from a member account.

**Batch policy edits.** Any IAM policy change to the principal revokes refresh tokens, so each tightening pass signs users out of the connector. Converge the policy before rollout.

### 3.3 Control the MCP path with condition keys

**Executed by:** an administrator, alongside [§3.2](#32-scope-a-dedicated-iam-role).

Scoping the role caps what the human can do everywhere. These two keys cap what the *agent* can do, which is the finer control. AWS MCP Server defines no IAM actions of its own — it authenticates the request, **adds two global condition context keys, and forwards to the downstream service**, which authorizes against existing policies `[docs 2026-08-24]`:

| Key | Type | Value |
| --- | --- | --- |
| `aws:ViaAWSMCPService` | Boolean | `true` for any request through an AWS managed MCP server |
| `aws:CalledViaAWSMCP` | String | Service principal of the specific server — `aws-mcp.amazonaws.com`, `eks-mcp.amazonaws.com`, `ecs-mcp.amazonaws.com` |

This is what makes read-only reachable on the OAuth path: **the same role keeps its write permissions for console and CLI use, while writes arriving through MCP are denied.** Attach to the role, or apply as an SCP:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "NoMutatingCallsThroughAnyAwsManagedMcpServer",
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "s3:DeleteObject",
        "s3:PutBucketPolicy",
        "kms:ScheduleKeyDeletion",
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": { "aws:ViaAWSMCPService": "true" }
      }
    },
    {
      "Sid": "OnlyTheAwsMcpServerNotOtherManagedMcpServers",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "Bool": { "aws:ViaAWSMCPService": "true" },
        "StringNotEquals": { "aws:CalledViaAWSMCP": "aws-mcp.amazonaws.com" }
      }
    }
  ]
}
```

Three things to know:

- **These statements are inert for direct API calls.** `Bool` does not match an absent key, so a console or CLI request — where `aws:ViaAWSMCPService` is not present — is unaffected. That is the inverse of the operator behaviour in [§3.2](#32-scope-a-dedicated-iam-role), and the reason the `Bool` guard belongs in every statement here.
- **A read-only posture needs an enumerated deny.** IAM does not accept a wildcard service prefix such as `*:Put*`, so list the mutating actions that matter, or invert with `NotAction` over an explicit allow-list per service.
- **`aws-mcp:InvokeMcp`, `aws-mcp:CallReadOnlyTool` and `aws-mcp:CallReadWriteTool` are deprecated** preview-era actions with no effect. A Deny written against them does not restrict anything; move it to the keys above `[docs 2026-08-24]`.

Not yet verified in this environment. The keys are documented as added to *all* requests the server forwards, which includes the OAuth path, and the observed `userAgent` of `aws-mcp.amazonaws.com` ([§2.2](#22-cloudtrail-audit-trail)) matches the `aws:CalledViaAWSMCP` value for this server. **Confirm with a write call through the agent before relying on it.**

### 3.4 Procedure

**Executed by:** a Quick administrator, once per AWS account. Connectors → **Create for your team** → **Model Context Protocol (MCP)** → name, endpoint, **Public network** → **User authentication (OAuth)** → **Create and continue**, then enable the discovered actions.

The `?oauth=initialize` parameter is not used with Quick. It exists for MCP clients that cannot start the OAuth flow natively; Quick starts it from the connector UI.

**The connector completed with a manually configured client** `[field 2026-07-16]`. The documented default is for Quick to register itself by DCR when the authorization server advertises a `registration_endpoint`, which this one does `[docs 2026-08-24]`. Whether Quick attempted that was not captured here, so try the default path first and use the manual path below if the connector does not complete. To configure it manually, register a public client against Quick's callback:

```bash
curl -sS -X POST "https://us-east-1.oauth.signin.aws/v1/register" \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "quick-aws-mcp",
    "redirect_uris": ["https://us-east-1.quicksight.aws.amazon.com/sn/oauthcallback"],
    "grant_types": ["authorization_code","refresh_token"],
    "response_types": ["code"],
    "token_endpoint_auth_method": "none"
  }'
```

Then complete the form with the values that worked `[field 2026-07-16]`:

| Field | Value |
| --- | --- |
| **Authentication method** | User authentication → **Custom user based OAuth** |
| **Public OAuth client** | **On** — the authorization server returns `["none"]`, so there is no secret |
| **Client ID** | `arn:aws:signin:us-east-1::external-client/dcr/<uuid>` from the response |
| **Client Secret** | *(blank)* |
| **Authorization URL** | `https://us-east-1.oauth.signin.aws/v1/authorize` |
| **Token URL** | `https://us-east-1.oauth.signin.aws/v1/token` |
| **Redirect URL** | *(read-only, pre-filled by Quick)* `https://us-east-1.quicksight.aws.amazon.com/sn/oauthcallback` |

The Redirect URL is generated by Quick and read-only — copy it out rather than typing one in, and re-check it per Region, since the host is Region-specific. Registration is not idempotent, so store the `client_id` and reuse it; the documented lifetime is 90 days, so diary a replacement `[docs 2026-08-24]`.

At consent each user signs in with their own AWS identity and the agent binds to it; an existing AWS Sign-In session can be reused without a fresh prompt `[docs 2026-08-24]`. Direct users to assume the role from [§3.2](#32-scope-a-dedicated-iam-role) *before* consenting — the SCP, rather than the instruction, is what enforces it.

### 3.5 Failure modes

| Symptom | Cause | Fix |
| --- | --- | --- |
| Connector creates and discovers tools, then `401` on first tool call — `"Authentication failed: Unable to verify your user identity"`, JSON-RPC `-32001`, with the `WWW-Authenticate` challenge | The caller's IAM principal lacks the OAuth permissions; it surfaces at first tool call because authorization is enforced at `tools/call` `[field 2026-07-16]` | Attach `AWSMCPSignInOAuthAccessPolicy` to the role, or the equivalent in [§3.1](#31-prerequisites), then check the integration's OAuth configuration |
| Quick does not complete automatic client registration | DCR is documented as available to approved agents `[docs 2026-08-24]`; a manually configured client completed the connector here `[field 2026-07-16]` | Configure the client manually ([§3.4](#34-procedure)) |
| HTTP 500 on repeated registration | Third consecutive call, consistent with rate limiting `[field 2026-07-16]` | Back off; reuse the existing `client_id` |
| Agent acts with more privilege than expected | It bound to the user's everyday identity at consent `[field 2026-07-16]` | [§3.2](#32-scope-a-dedicated-iam-role) and [§3.3](#33-control-the-mcp-path-with-condition-keys). Check with `aws sts get-caller-identity` through the agent |
| A Deny intended to block MCP has no effect | It targets the deprecated `aws-mcp:*` actions `[docs 2026-08-24]` | Rewrite against `aws:ViaAWSMCPService` / `aws:CalledViaAWSMCP` ([§3.3](#33-control-the-mcp-path-with-condition-keys)) |
| `400` page after OAuth sign-in | Principal lacks the two `signin:` actions `[docs 2026-08-24]` | [§3.1](#31-prerequisites); if the SCP is deployed, confirm the user assumed the dedicated role |
| Authorization worked, then stopped after about 90 days | DCR client lifetime `[docs 2026-08-24]` | Register a replacement client and update the integration |
| `HTTP 424` on real work | The 60-second Quick operation timeout rather than an AWS throttle `[docs 2026-08-24]` | Poll with `aws___get_tasks` |
| `403` with `insufficient_scope` | Step-up authorization is not available in Quick `[docs 2026-08-24]` | Re-authorize the whole connection |
| Agent stops using AWS tools mid-session | Token expired, or a policy change revoked the refresh token `[docs 2026-08-24]` | Re-authorize; batch policy edits |

---

## 4. References

### 4.1 Provenance

Findings are attested by the author in one test account, one Region, on the dates tagged.

**Registration outcome and the documented allow-list.** The documentation states that DCR is available to approved agents and that only redirect URIs matching allow-listed patterns are accepted, and the published list does not include Quick's callback `[docs 2026-08-24]`. Registration against that callback completed without authentication `[field 2026-07-16]`, five weeks before that documentation was read. Both are recorded as-is; the likely explanations are a change to the allow-list between the two dates, or enforcement occurring at a different point than registration. **Verify in your own account before depending on either behaviour.**

**Revised 2026-08-24 — read-only enforcement.** *Earlier statement:* because read-only mode is a SigV4 feature, the IAM role was the only available read-only control. *Current statement:* hiding write-capable tools from the agent is SigV4-only, and read-only can be enforced on any path with `aws:ViaAWSMCPService` ([§3.3](#33-control-the-mcp-path-with-condition-keys)). *Why it matters:* the current control leaves the human's write access intact while denying writes that arrive through the agent, which role-scoping alone cannot express.

**Revised 2026-08-24 — service-to-service OAuth.** *Earlier statement:* AWS Sign-In offers no service-to-service path, reasoning from `client_credentials` being absent from `grant_types_supported` — still absent `[field 2026-08-24]`. *Current statement:* the grant exists as `CreateOAuth2TokenWithIAM` `[docs 2026-08-24]`. *Why it matters:* an AWS-specific API surface is not necessarily discoverable from RFC 8414 metadata, so reasoning from metadata alone to "this capability does not exist" is unsound. The grant remains unusable from Quick, though for narrower reasons than first given: Quick does offer a Service authentication method taking a Client ID, Secret, and Token URL, but the API requires SigV4-signing the request to `POST /v1/token?x-amz-client-auth-method=iam`, and DCR clients are excluded from `client_credentials`.

**Revised 2026-08-24 — DCR client lifetime.** *Earlier statement:* a registered `client_id` is permanent, inferred from two registrations minutes apart. *Current statement:* the documented lifetime is 90 days `[docs 2026-08-24]`. *Why it matters:* the observation window was too short to show expiry, so the connector needs a scheduled replacement.

**Not yet verified.** Whether the [§3.3](#33-control-the-mcp-path-with-condition-keys) condition keys are populated on the OAuth path, and whether a deny built on them applies. Also open: the control-plane `signin:` CloudTrail events ([§2.2](#22-cloudtrail-audit-trail)); per-user isolation across two different users, which follows from the authorization-code flow but was observed for one identity only; how the OAuth identity becomes the downstream STS role session, where the evidence establishes only that the call is made server-side under the user's own role; whether an IAM denial surfaces from IAM rather than the MCP layer; session lifetime across refresh; and the SigV4 path to the same endpoint.

### 4.2 Further reading

Every link below resolved on 2026-08-24 and supports a claim in this document.

**AWS MCP Server**

- [Setting up the AWS MCP Server](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/getting-started-aws-mcp-server.html) — endpoints, Regions, OAuth versus SigV4, token validity, multi-account support
- [How AWS MCP Server works with IAM](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/security_iam_service-with-iam.html) — `aws:ViaAWSMCPService`, `aws:CalledViaAWSMCP`, and the deprecated `aws-mcp:*` actions
- [OAuth 2.1 authentication for AWS MCP Server](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/oauth-authentication.html) — token lifecycle and binding
- [Understanding the MCP Server tools](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/understanding-mcp-server-tools.html) — tool inventory, `aws___call_aws` deprecation dates

**AWS Sign-In**

- [Configure OAuth access to AWS MCP Server](https://docs.aws.amazon.com/signin/latest/userguide/aws-mcp-server.html) — authorization models, DCR redirect-URI list, CloudTrail event examples, `aws:SignInSessionArn`
- [AWS Sign-In condition keys reference](https://docs.aws.amazon.com/signin/latest/userguide/reference-signin-condition-keys.html) — the `signin:` keys and where `signin:PrincipalArn` applies
- [`CreateOAuth2PublicClient`](https://docs.aws.amazon.com/signin/latest/APIReference/API_dataplane-signin_CreateOAuth2PublicClient.html) — 90-day registration lifetime, allow-listing, `client_credentials` exclusion
- [`CreateOAuth2TokenWithIAM`](https://docs.aws.amazon.com/signin/latest/APIReference/API_dataplane-signin_CreateOAuth2TokenWithIAM.html) — SigV4 requirement and `x-amz-client-auth-method=iam`

**Amazon Quick**

- [MCP integration in Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html) — Enterprise prerequisite, the three authentication methods and their fields, discovery behaviour, limitations

**IAM policy mechanics**

- [AWS global condition context keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) — `aws:PrincipalArn` holds the role ARN, not the session ARN
- [Policy variables with no value](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html) — inverted operators match an absent key; the basis for the `Null` guard
- [SCP effects on permissions](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) — member accounts including root, not the management account
- [CloudTrail `userIdentity` element](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-user-identity.html) — reading `AssumedRole` and `sessionIssuer`

**Specifications**

- [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728.html) Protected Resource Metadata · [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414.html) Authorization Server Metadata · [RFC 7591](https://www.rfc-editor.org/rfc/rfc7591.html) Dynamic Client Registration · [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636.html) PKCE · [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html) OAuth 2.0
