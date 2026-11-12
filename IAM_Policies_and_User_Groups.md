# Lesson 6: AWS IAM Policies and User Groups Explained with Demo

*AWS Education Notes · Author: Jayesh Sharma*

> **Learning approach:** Understand the concepts, inspect policy JSON, follow an authorization request, and practise in a sandbox. This is an original topic-based learning guide. Verify production behavior against the [official AWS IAM documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/).

## 1. Why IAM policies matter

Authentication establishes **who or what is making a request**. Authorization decides **which actions that identity is allowed to perform**. AWS IAM policies define those permissions. The key principle is **least privilege**: grant only what is required.

~~~mermaid
flowchart LR
    ID["IAM identity or workload"] --> REQ["AWS API request"]
    REQ --> AUTH["Authorization evaluation"]
    POL["Applicable policies"] --> AUTH
    AUTH --> DEC{"Allowed?"}
    DEC -->|"Yes"| ACT["Action proceeds"]
    DEC -->|"No"| DENY["Access denied"]
~~~

| Term | Meaning | Example |
|---|---|---|
| Authentication | Who are you? | Sign in or obtain role credentials |
| Authorization | What can you do? | Read an object from a specific bucket |

## 2. IAM policy basics

An IAM policy is a JSON document describing permissions. A statement generally specifies:

- **Effect:** Allow or Deny.
- **Action:** AWS API operation, such as s3:GetObject.
- **Resource:** AWS resource on which the action applies.
- **Condition:** Optional rules that must be satisfied.

Think of a policy as a rule: allow this identity to perform this action on this resource, provided the specified conditions are met.

## 3. Anatomy of a policy document

This identity-based policy allows reading objects from one S3 bucket.

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadReports",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-reports/*"
    }
  ]
}
~~~

| Field | Meaning |
|---|---|
| Version | Policy language version, not the revision number of your policy |
| Statement | One or more permission statements |
| Sid | Optional statement identifier |
| Effect | Allow or Deny |
| Action | API actions covered by the statement |
| Resource | Resource ARN(s) to which the statement applies |
| Condition | Optional condition operators and keys |

**Important S3 detail:** s3:GetObject applies to object ARNs. Listing a bucket is a different operation, s3:ListBucket, and uses the bucket ARN without /*.

To allow both listing and reading, use separate statements:

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListReportsBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::company-reports"
    },
    {
      "Sid": "ReadReportObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-reports/*"
    }
  ]
}
~~~

## 4. Identity-based policies

An identity-based policy is attached to an IAM user, group, or role. It answers: **What can this identity do?**

~~~mermaid
flowchart TD
    DEV["Developer role"] --> POLICY["Identity-based policy"]
    POLICY --> ALLOW["s3:GetObject"]
    ALLOW --> BUCKET["Objects in dev-artifacts bucket"]
~~~

Identity-based policies can be:

| Type | Explanation | Typical use |
|---|---|---|
| AWS managed | Created and maintained by AWS | Common permission sets |
| Customer managed | Created and maintained in your account | Reusable organization-specific permissions |
| Inline | Embedded directly in one identity | Identity-specific policy |

A managed policy is a separate IAM resource and can be attached to multiple identities. Updating it can affect every identity to which it is attached. An inline policy belongs to one identity and is deleted with that identity.

## 5. Resource-based policies

A resource-based policy is attached to a supported AWS resource. It specifies which principal can access that resource and what actions are allowed or denied.

Example S3 bucket policy allowing a specific role to read objects:

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowApplicationRoleToRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:role/ReportsReaderRole"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-reports/*"
    }
  ]
}
~~~

The Principal field identifies who the resource policy applies to.

~~~mermaid
flowchart LR
    ROLE["ReportsReaderRole"] -->|"GetObject request"| S3["S3 bucket"]
    S3 --> BP["Bucket policy checks principal and action"]
    BP --> OBJ["Object access if authorization succeeds"]
~~~

### Cross-account access

Suppose Account A owns an S3 bucket and Account B has an application role. For a typical cross-account S3 request:

1. The role in Account B needs an identity-based policy allowing the requested S3 action on the bucket/object.
2. The bucket policy in Account A must allow that role as a principal.
3. Other controls, such as explicit Deny, an SCP, permissions boundary, or KMS key policy, must not block the request.

~~~mermaid
flowchart LR
    subgraph A["Account B - caller"]
      ROLE["Application role"]
      IDP["Identity policy allows S3 read"]
      ROLE --> IDP
    end
    subgraph B["Account A - resource owner"]
      BP["Bucket policy allows Account B role"]
      BUCKET["S3 bucket and objects"]
      BP --> BUCKET
    end
    ROLE -->|"Cross-account GetObject"| BP
~~~

An Allow on only one side is not enough for the typical cross-account S3 access pattern.

## 6. Comparing policy types

| Aspect | Identity-based policy | Resource-based policy |
|---|---|---|
| Attached to | User, group, or role | Supported resource |
| Main question | What can this identity do? | Who can access this resource, and how? |
| Principal field | Not normally specified | Identifies the principal |
| Example | Role policy allowing s3:GetObject | Bucket policy allowing a role to read objects |

Both policy types can matter to the same request. They are not mutually exclusive alternatives.

## 7. How AWS evaluates permissions

A useful starting model:

1. Requests are implicitly denied unless an applicable permission allows them.
2. An applicable explicit Deny overrides an Allow.
3. Other policy types and service-specific authorization rules can further constrain the result.

~~~mermaid
flowchart TD
    START["AWS API request"] --> DENY{"Applicable explicit Deny?"}
    DENY -->|"Yes"| BLOCK["Denied"]
    DENY -->|"No"| ALLOW{"Applicable Allow?"}
    ALLOW -->|"No"| IMPLICIT["Implicitly denied"]
    ALLOW -->|"Yes"| LIMIT{"Any applicable limiting control?"}
    LIMIT -->|"Blocks request"| BLOCK
    LIMIT -->|"Does not block"| PERMIT["Allowed"]
~~~

| Policy/control | What it does |
|---|---|
| Identity-based policy | Grants or denies permissions to an identity |
| Resource-based policy | Grants or denies access at a supported resource |
| Permissions boundary | Sets the maximum permissions an IAM user or role can receive through identity-based policies |
| Service control policy (SCP) | Sets maximum available permissions for principals in affected organization accounts; does not grant permissions by itself |
| Session policy | Can further restrict permissions for a role session |
| Explicit Deny | Overrides an applicable Allow |

**Important nuance:** Resource-based policies that grant directly to IAM users, role sessions, or other principal forms can interact differently with permissions boundaries and session policies. Do not assume every Allow is combined using one universal rule. See [AWS policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html).

Example: a role has an Allow for s3:GetObject on company-reports/* and an explicit Deny for company-reports/confidential/*. Public report objects can be read if no other control blocks them; confidential objects are denied because explicit Deny overrides Allow.

## 8. IAM users and groups

An IAM user represents a person or application requiring long-term IAM credentials. For human workforce access, AWS generally recommends federation and temporary credentials rather than creating long-lived IAM users wherever possible.

An IAM group is a collection of IAM users. Policies attached to the group grant permissions to its users.

Example: five developers need read access to a development S3 bucket. Create a Developers group and attach the policy once, rather than attaching the same policy to every user.

~~~mermaid
flowchart TD
    GROUP["IAM group: Developers"]
    POLICY["Customer-managed policy: DevS3ReadOnly"]
    U1["Developer A"]
    U2["Developer B"]
    U3["Developer C"]
    GROUP --> POLICY
    U1 --> GROUP
    U2 --> GROUP
    U3 --> GROUP
~~~

**Why groups?**
- Centralized permission management.
- Consistent access for people with similar responsibilities.
- Easier onboarding and offboarding.
- Less repeated policy attachment and configuration drift.

**Important group facts**
- A group contains IAM users, not other groups.
- A group is not a principal that signs in or makes AWS API requests.
- A group cannot be assumed like a role.
- A user can belong to more than one group; applicable permissions are evaluated together.
- A policy attached to a group does not automatically become a resource-based policy.

| IAM group | IAM role |
|---|---|
| Organizes IAM users | Provides an assumable identity |
| Policies grant group members permissions | Policies grant permissions to role sessions |
| Does not provide temporary credentials by itself | Commonly provides temporary credentials through AWS STS |
| Useful for grouping human IAM users | Useful for AWS services, federated users, and cross-account access |

## 9. Conditions and policy variables

A condition makes an Allow or Deny statement dependent on request context, such as Region, source IP, principal tags, resource tags, or time.

Example: deny S3 requests that do not use HTTPS.

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-reports",
        "arn:aws:s3:::company-reports/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
~~~

This bucket-policy-style guardrail denies matching S3 requests that do not use secure transport. Check the condition key's availability and behavior for the service and request type being configured.

| Condition key/category | Possible purpose |
|---|---|
| aws:RequestedRegion | Restrict requests to approved Regions, where supported |
| aws:SourceIp | Restrict access based on source IP, where meaningful for the request path |
| aws:PrincipalTag | Make access depend on tags attached to the principal |
| aws:ResourceTag | Make access depend on resource tags, where supported |
| aws:SecureTransport | Require secure transport for supported service requests |

Policy variables allow a policy to use request or identity context dynamically. For example, the variable \${aws:username} can be used in supported policies to refer to an IAM user's name. Verify that a variable is available in the context where the policy is evaluated.

Conditions are not a substitute for least privilege. Test them with the actual service and request path.

## 10. Hands-on S3 exercise

**Goal:** Create a group whose users can list one S3 bucket and read its objects, but cannot upload or delete objects through the permissions granted by this policy.

### Step 1: Create a test bucket
1. Open the S3 console.
2. Create a uniquely named bucket in your chosen Region.
3. Keep Block Public Access enabled.
4. Upload a harmless test file such as sample.txt.

Do not use real customer or confidential data in a learning account.

### Step 2: Create a customer-managed policy

Create a policy named CompanyReportsReadOnly. Replace the bucket name with your actual bucket name.

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyTheReportsBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::company-reports"
    },
    {
      "Sid": "ReadObjectsInReportsBucket",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-reports/*"
    }
  ]
}
~~~

### Step 3: Create a group and attach the policy
1. In IAM, open **User groups**.
2. Create a group named ReportsReaders.
3. Attach CompanyReportsReadOnly.
4. Create a test IAM user only if your lab specifically requires one. Prefer a temporary/federated identity for real workforce access.
5. Add the test user to ReportsReaders.

### Step 4: Test permissions

Sign in as the test identity or use its approved lab credentials. First verify the active identity:

~~~bash
aws sts get-caller-identity
~~~

List the bucket:

~~~bash
aws s3 ls s3://company-reports/
~~~

Download an object:

~~~bash
aws s3 cp s3://company-reports/sample.txt ./sample.txt
~~~

Try an upload:

~~~bash
aws s3 cp ./sample.txt s3://company-reports/upload-test.txt
~~~

The upload should fail if no other applicable policy grants write access and no resource-side grant changes the effective result. The group policy itself grants no upload permission.

| Test | Expected result from this policy |
|---|---|
| List the bucket | Allowed |
| Download an object | Allowed |
| Upload an object | Not allowed by this policy |
| Delete an object | Not allowed by this policy |

An AccessDenied result is useful evidence, but inspect the full policy context before concluding why a request failed.

### Step 5: Clean up

Remove the test user from the group or delete the lab identity when finished. Delete the test bucket and its objects if no longer needed. Remove the lab policy and group if created only for this exercise.

**Cost and safety:** IAM itself does not have a separate charge for ordinary use, but S3 storage, requests, data transfer, and other resources may incur charges. Use a sandbox, enable MFA, never share access keys, and review your account's billing and Free Tier terms.

## 11. Troubleshooting AccessDenied

When an AWS API call returns AccessDenied, work through the request systematically:

1. **Identify the caller:** Run aws sts get-caller-identity. Confirm the account, ARN, and assumed role/session.
2. **Confirm the action:** Check the exact API action. A console page may need additional list or describe permissions beyond the main operation.
3. **Check the resource ARN:** Confirm bucket ARN versus object ARN, path, Region, and account.
4. **Inspect identity policies:** Look for an applicable Allow and any explicit Deny.
5. **Inspect resource policies:** Check the S3 bucket policy, KMS key policy, or other applicable resource policy.
6. **Check limiting controls:** Review permissions boundaries, SCPs, session policies, and service-specific controls.
7. **Check conditions:** A source IP, Region, tag, MFA, TLS, or other condition may not match the request context.
8. **Check dependent permissions:** Reading an SSE-KMS encrypted S3 object may also require permission to use the KMS key.
9. **Use evidence:** Review the full error message, CloudTrail event where available, and IAM policy tools.

~~~mermaid
flowchart TD
    ERR["AccessDenied"] --> WHO["Who made the request?"]
    WHO --> ACTION["Which API action and resource ARN?"]
    ACTION --> IDPOL["Inspect identity policy"]
    IDPOL --> RES["Inspect resource and key policies"]
    RES --> LIMIT["Inspect boundary, SCP, session and conditions"]
    LIMIT --> TEST["Correct one issue and retest"]
~~~

**Useful tools**
- [IAM Policy Simulator](https://policysim.aws.amazon.com/) — test policy behavior for supported scenarios. It is a diagnostic aid, not a guarantee that every live service request behaves identically.
- [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) — help identify external access and validate policies.
- [AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html) — investigate recorded API activity and relevant authorization errors.

## 12. Security best practices

1. **Least privilege:** Grant only the actions and resources needed.
2. **Prefer roles and temporary credentials:** Avoid distributing long-lived access keys to applications or people.
3. **Use groups for common user permissions:** Manage human IAM user permissions consistently when IAM users are required.
4. **Use customer-managed policies thoughtfully:** Reuse reviewed policies where the same permission set is appropriate.
5. **Avoid broad wildcards:** Action "*" and Resource "*" can grant much more access than intended.
6. **Use explicit Deny guardrails carefully:** They are powerful and can block otherwise valid access.
7. **Review cross-account access:** Verify caller-side permissions and the resource owner's trust/permission configuration.
8. **Use conditions appropriately:** Test context keys and request paths rather than assuming a condition works everywhere.
9. **Audit unused access:** Review policies, users, keys, and roles periodically.
10. **Protect privileged identities:** Use MFA, strong federation controls, and tightly controlled administrative access.

## 13. Interview questions and quick revision

**1. What is an IAM policy?**  
A JSON document that defines permissions using Effect, Action, Resource, and optionally Condition.

**2. Authentication vs authorization?**  
Authentication verifies identity; authorization determines which actions that identity can perform.

**3. What is an identity-based policy?**  
A policy attached to an IAM user, group, or role that defines permissions for that identity.

**4. What is a resource-based policy?**  
A policy attached to a supported AWS resource that specifies which principals can access it and what actions are allowed or denied.

**5. Allow vs implicit Deny vs explicit Deny?**  
Implicit Deny applies when no relevant Allow grants the request. Explicit Deny directly denies a matching request and overrides an applicable Allow.

**6. Why use IAM groups?**  
To manage common permissions for multiple IAM users centrally instead of attaching the same policy to each user separately.

**7. Can an IAM group be assumed like a role?**  
No. A group is a collection of IAM users. It is not an assumable identity and cannot obtain role session credentials.

**8. Managed policy vs inline policy?**  
A managed policy is a separate reusable policy resource. An inline policy is embedded in one identity.

**9. What is a permissions boundary?**  
It defines the maximum permissions an IAM user or role can receive through identity-based policies. It does not grant permissions by itself.

**10. What is an SCP?**  
A Service Control Policy sets permission guardrails for accounts in an AWS Organization. It does not grant permissions by itself.

**11. What should you check for cross-account S3 access?**  
Check the caller role's identity policy and the bucket owner's resource policy, then inspect explicit Deny, SCPs, boundaries, and KMS key permissions.

**12. Why might s3:GetObject work while listing the bucket fails?**  
Object retrieval and bucket listing are separate API actions and use different resource ARN formats. Listing requires s3:ListBucket on the bucket ARN.

### Quick revision

| Concept | Remember this |
|---|---|
| IAM policy | JSON permission document |
| Effect | Allow or Deny |
| Action | API operation, such as s3:GetObject |
| Resource | ARN of the target resource |
| Condition | Context that must match |
| Identity-based policy | Attached to user, group, or role |
| Resource-based policy | Attached to a supported resource; includes Principal |
| IAM group | Collection of IAM users |
| Managed policy | Separate reusable policy |
| Inline policy | Embedded in one identity |
| Explicit Deny | Overrides an applicable Allow |
| Permissions boundary | Maximum permissions available through identity policies |
| SCP | Organization-level permission guardrail; does not grant access |

### One-minute interview explanation

AWS IAM policies define who can perform which API actions on which resources, optionally under specified conditions. Identity-based policies are attached to users, groups, or roles, while resource-based policies are attached to supported resources and identify the principals that can access them. AWS evaluates applicable policies and restrictions; an explicit Deny overrides an Allow. IAM groups help manage common permissions for multiple users. In practice, I would start with least privilege, test the policy against required actions and resources, and investigate AccessDenied by checking the caller identity, policy layers, resource policy, and conditions.

### Official references
- [IAM policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [Identity-based policies and resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)
- [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [IAM user groups](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups.html)
- [IAM JSON policy elements](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html)
- [IAM policy simulator](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html)
- [AWS course playlist](https://www.youtube.com/playlist?list=PL6W8uoQQ2c60g3PAgzVPpi0YWwxITD43p)
