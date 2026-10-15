# Direct Connect Information Sharing Framework – Cross-Account IAM Role

This sample contains an AWS CloudFormation template that lets an AWS Direct Connect customer share Direct Connect telemetry with its WAN Service Provider (WSP) or Enterprise Service Provider (ESP) through a read-only, cross-account IAM role.

The provider's monitoring system assumes the role with temporary AWS Security Token Service (AWS STS) credentials and reads:

- Direct Connect configuration (connections, virtual interfaces, LAGs) through the Direct Connect `Describe*` and `List*` APIs
- Amazon CloudWatch metrics for Direct Connect (the `AWS/DX` namespace), such as connection state, throughput, and optical light levels
- Operational metadata that the customer publishes as resource tags (contacts, circuit IDs, maintenance windows) through the Resource Groups Tagging API

AWS does not replicate or export any data to the provider's account. The provider retrieves data through read-only API calls and may store it in its own systems. The role grants no write permissions, and every call is logged in the customer's AWS CloudTrail.

This template accompanies the AWS Prescriptive Guidance guide *Direct Connect Information Sharing Framework: Enabling Secure, Customer-Controlled Telemetry Sharing with WAN Service Providers Using Cross-Account IAM Roles*. The template uses `AWS::Partition` for every ARN, so it deploys without changes in any AWS partition.

## How it works

1. The customer deploys `dx-info-sharing.yaml` in its account. The stack creates one IAM managed policy and one IAM role.
2. The provider's monitoring system, running as a specific IAM role in the provider's account, calls `sts:AssumeRole` with the customer's role ARN, the customer-specific ExternalId, and a source identity.
3. AWS STS evaluates the role's trust policy and returns temporary credentials that are valid for up to one hour.
4. The monitoring system calls the read-only APIs. It scopes CloudWatch queries to the `AWS/DX` namespace and filters `tag:GetResources` requests to resources tagged `DXInfoSharing=enabled`.
5. CloudTrail in the customer account records the role assumption and every API call, attributed to the source identity.

## What the template deploys

| Logical ID | Type | Default name |
|---|---|---|
| `DXNetworkOperatorPolicy` | `AWS::IAM::ManagedPolicy` | `DXNetworkOperatorPolicy` |
| `DXNetworkOperatorRole` | `AWS::IAM::Role` | `DXNetworkOperatorRole` |

### Trust policy

The role's trust policy:

- Trusts exactly one IAM role in the provider's account (`arn:<partition>:iam::<ESPAccountId>:role/<ESPRoleName>`), not the provider's whole account.
- Requires `sts:ExternalId` to match the `ExternalId` parameter on every `AssumeRole` call. This mitigates the [confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html).
- Requires a [source identity](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html) when `RequireSourceIdentity` is `true` (the default). CloudTrail records it for every action in the session, and it can't be changed for the life of the session.
- Allows `sts:SetSourceIdentity`, so the provider can set a source identity or carry one over from its own session through role chaining.

With the default parameters, the deployed trust policy looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowESPAssumeRole",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/DXMonitoringRole" },
      "Action": ["sts:AssumeRole", "sts:SetSourceIdentity"],
      "Condition": {
        "StringEquals": { "sts:ExternalId": "<ExternalId>" },
        "Null": { "sts:SourceIdentity": "false" }
      }
    }
  ]
}
```

### Permissions

All permissions are read-only. The policy grants no `iam:PassRole` and no IAM create, update, delete, or attach permissions, so the provider can't change or extend its own access.

| Statement | Actions | Resource scope |
|---|---|---|
| `DirectConnectReadOnly` | `directconnect:Describe*`, `directconnect:List*` | All |
| `CloudWatchMetricsRead` | `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics`, `cloudwatch:ListMetrics` | All (see [Limitations](#limitations)) |
| `CloudWatchAlarmDiscovery` | `cloudwatch:DescribeAlarms`, `cloudwatch:DescribeAlarmsForMetric` | All |
| `TagBasedResourceDiscovery` | `tag:GetResources`, `tag:GetTagKeys`, `tag:GetTagValues` | All (see [Limitations](#limitations)) |
| `IAMRoleSelfAudit` | `iam:GetRole`, `iam:GetRolePolicy`, `iam:ListAttachedRolePolicies`, `iam:ListRolePolicies` | This role only |
| `IAMPolicySelfAudit` | `iam:GetPolicy`, `iam:GetPolicyVersion` | This policy only |
| `ConfigComplianceRead` (optional) | `config:DescribeConfigRules`, `config:DescribeComplianceByConfigRule`, `config:DescribeComplianceByResource`, `config:GetComplianceDetailsByConfigRule`, `config:GetComplianceDetailsByResource` | All. Added only when `EnableConfigAccess` is `true` |
| `VPCReadOnly` (optional) | `ec2:DescribeVpcs`, `ec2:DescribeVpcPeeringConnections`, `ec2:DescribeRouteTables`, `ec2:DescribeVpnConnections`, `ec2:DescribeVpnGateways`, `ec2:DescribeCustomerGateways` | All. Added only when `EnableVPCAccess` is `true` |
| `TGWReadOnly` (optional) | `ec2:DescribeTransitGateways`, `ec2:DescribeTransitGatewayAttachments`, `ec2:DescribeTransitGatewayRouteTables`, `ec2:SearchTransitGatewayRoutes` | All. Added only when `EnableTGWAccess` is `true` |

## Prerequisites

- An AWS account with at least one Direct Connect connection.
- Permissions to create IAM roles and managed policies through CloudFormation.
- CloudTrail enabled in the account.
- The following values from the provider, exchanged over an encrypted channel:
  - The provider's 12-digit AWS account ID. It must be in the same AWS partition as your account.
  - The name of the provider's IAM role that will assume your role, including its path if it has one. **The role must exist before you deploy the stack.** IAM rejects a trust policy that names a role that doesn't exist.
  - The ExternalId that the provider generated for you: 32–256 characters, unique per customer, and random (a version 4 UUID is recommended). The provider generates and manages this value, not the customer.

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `ESPAccountId` | Yes | – | The provider's 12-digit AWS account ID. |
| `ESPRoleName` | Yes | – | The provider's IAM role name, with its path if it has one (for example, `DXMonitoringRole` or `service-role/DXMonitoringRole`). |
| `ExternalId` | Yes | – | The ExternalId from the provider. 32–256 characters from the set `A-Z a-z 0-9 _ + = , . @ : / -`. Hidden in the console and API output (`NoEcho`). |
| `RequireSourceIdentity` | No | `true` | Require the provider to set a source identity when it assumes the role. If `false`, a source identity is accepted but not required. |
| `EnableVPCAccess` | No | `false` | Add read-only access to VPCs, peering connections, route tables, and VPN resources. |
| `EnableTGWAccess` | No | `false` | Add read-only access to Transit Gateway resources. |
| `EnableConfigAccess` | No | `false` | Add read-only access to AWS Config compliance data. AWS Config support for Direct Connect resource types varies by Region and partition. |
| `RoleName` | No | `DXNetworkOperatorRole` | Name of the IAM role to create. |
| `PolicyName` | No | `DXNetworkOperatorPolicy` | Name of the IAM managed policy to create. |

## Deploy (customer)

### AWS CLI

Read the ExternalId into a variable so that it doesn't land in your shell history, then deploy:

```bash
read -rs -p "ExternalId from your provider: " EXTERNAL_ID; echo

aws cloudformation deploy \
  --template-file dx-info-sharing.yaml \
  --stack-name dx-info-sharing \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ESPAccountId=111122223333 \
    ESPRoleName=DXMonitoringRole \
    ExternalId="$EXTERNAL_ID"
```

`CAPABILITY_NAMED_IAM` is required because the template creates IAM resources with custom names.

Get the role ARN and send it to the provider:

```bash
aws cloudformation describe-stacks \
  --stack-name dx-info-sharing \
  --query "Stacks[0].Outputs[?OutputKey=='RoleArn'].OutputValue" \
  --output text
```

### AWS Management Console

1. Open the AWS CloudFormation console and choose **Create stack**, **With new resources (standard)**.
2. Choose **Upload a template file** and select `dx-info-sharing.yaml`.
3. Enter a stack name and the parameter values.
4. Acknowledge that CloudFormation might create IAM resources with custom names, and then choose **Submit**.
5. On the **Outputs** tab, copy the `RoleArn` value and send it to the provider.

### Tag the resources you want to share

The provider reads operational metadata only from resources that you tag with `DXInfoSharing=enabled`:

```bash
aws directconnect tag-resource \
  --resource-arn arn:aws:directconnect:us-east-1:444455556666:dxcon/dxcon-EXAMPLE \
  --tags key=DXInfoSharing,value=enabled \
         key=NOCContact,value=noc@example.com \
         key=TechnicalPOC,value=netadmin@example.com
```

Tag values are visible to the provider, so add only information that you intend to share.

## Assume the role (provider)

The provider's role (`ESPRoleName`) needs an identity-based policy that allows it to assume the customer's role and set a source identity:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sts:AssumeRole", "sts:SetSourceIdentity"],
      "Resource": "arn:aws:iam::444455556666:role/DXNetworkOperatorRole"
    }
  ]
}
```

Assume the role from the provider's role session:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::444455556666:role/DXNetworkOperatorRole \
  --role-session-name dx-monitoring \
  --external-id "<ExternalId>" \
  --source-identity dx-monitoring-system
```

The source identity must be 2–64 characters from the set `A-Z a-z 0-9 _ + = , . @ -` and can't start with `aws:`. Use a value that identifies the operator or system, such as an employee ID or the monitoring system's name.

With the temporary credentials, validate access:

```bash
aws directconnect describe-connections
aws cloudwatch list-metrics --namespace AWS/DX
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=DXInfoSharing,Values=enabled
```

**AWS Management Console access:** **Switch Role** in the console doesn't work with this role, because the console can't pass an ExternalId or a source identity. Use the AWS CLI or an AWS SDK. Operators who need the console can use a [custom identity broker](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html) that exchanges the temporary credentials from `AssumeRole` for a console sign-in URL.

## Operations

### Rotate the ExternalId

1. The provider generates a new ExternalId and sends it to you over an encrypted channel.
2. Update the stack with the new `ExternalId` value.
3. The provider updates its monitoring configuration.

Role assumption fails between steps 2 and 3. Agree on a change window with the provider. Retry logic for `AccessDenied` responses on `AssumeRole` shortens the gap.

### If the provider re-creates its role

IAM pins the trust policy to the trusted role's unique ID. If the provider deletes and re-creates its role, the new role can't assume your role, even with the same name. This is intentional. To restore access, update the stack with a changed parameter value (for example, a rotated ExternalId) so that CloudFormation rewrites the trust policy.

### Monitor access

Review CloudTrail events for the role regularly, and alert on unexpected activity. Useful signals are `AssumeRole` calls that fail with `AccessDenied`, calls from unexpected source IP addresses, unusual call volumes, and unknown source identities.

### Revoke access

From least to most disruptive:

| Goal | Action |
|---|---|
| Stop sharing operational metadata | Remove the `DXInfoSharing` tags from your resources. This doesn't revoke credentials. |
| Invalidate active sessions | In the IAM console, open the role and choose **Revoke sessions**, **Revoke active sessions**. IAM attaches the `AWSRevokeOlderSessions` inline policy, which denies credentials issued before that time. See [Revoke IAM role temporary security credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_revoke-sessions.html). |
| Remove access permanently | Delete the stack. Combine with session revocation to end active sessions too. |

## Limitations

- **Describe APIs aren't tag-scoped.** Direct Connect `Describe*` APIs return every connection and virtual interface in the account. The `DXInfoSharing=enabled` tag controls only which resources' operational metadata the provider retrieves through `tag:GetResources`.
- **Tag filtering is applied by the provider, not by IAM.** The Resource Groups Tagging API doesn't support resource-level permissions or tag condition keys. The provider must include `TagFilters` in its requests. Enforce your tagging standards with periodic audits.
- **CloudWatch namespace scoping is applied by the provider, not by IAM.** `GetMetricData`, `GetMetricStatistics`, and `ListMetrics` don't support a namespace condition key, so the role can read metrics in any namespace. The provider's monitoring system should query only `AWS/DX`, plus `AWS/NetworkMonitor` if you use CloudWatch Network Synthetic Monitor.
- **Polling only.** The provider detects changes by polling. For event-driven notifications, see the Amazon EventBridge options in the guide.
- **Same partition only.** The provider's account must be in the same AWS partition as yours.

## Cost

IAM, AWS STS, and CloudFormation don't charge for the resources in this template. CloudWatch API requests that the provider makes with this role, such as `GetMetricData`, are billed to your account at standard CloudWatch rates. CloudTrail charges depend on your trail configuration.

## Clean up

```bash
aws cloudformation delete-stack --stack-name dx-info-sharing
```

Deleting the stack removes the role and the policy. Remove the `DXInfoSharing` and related tags from your Direct Connect resources separately.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
