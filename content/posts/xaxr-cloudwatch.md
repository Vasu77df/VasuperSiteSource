---
title: "Setting up Cloudwatch Cross Account Observability with CDK"
date: 2025-02-22
tags: ["aws", "cloudwatch", "observability"]
author: "Vasuper"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
---


Cross Account Cross Region(XAXR) Cloudwatch Observability has been a feature that has been around for a while, it was introduced by AWS in [2022](https://aws.amazon.com/blogs/aws/new-amazon-cloudwatch-cross-account-observability/).

Let's consider this scenario, you work in a company, that uses AWS. You are in a team that vends constructs/templates to other internal teams in your company. These internal teams are your customers. You vend your solution as an abstraction, that solves an undifferentiated problem for your customer. This let's your customer focus on what makes their [beer taste better](https://podup.substack.com/p/jeff-bezoss-beer-tasting-analogy).

Let's also consider a scenario where your team manages a service, and interacts with an other service, and you you want to monitor the health of your dependency.

XAXR Cloudwatch is a nice way to keep tabs on your solution, for example to gather key business metrics, or help troubleshoot an issue, with your customer. With the second secanrio, XAXR also lets you create event traces between services across accounts. This is how it works.

Source accounts share their observability data with the monitoring account. The shared observability data can include the following types of telemetry:

- Metrics in Amazon CloudWatch. You can choose to share the metrics from all namespaces with the monitoring account, or filter to a subset of namespaces.
- Log groups in Amazon CloudWatch Logs. You can choose to share all log groups with the monitoring account, or filter to a subset of log groups.
- Traces in AWS X-Ray
- Applications in Amazon CloudWatch Application Insights
- Monitors in CloudWatch Internet Monitor

Here are some more details on this mechanism:
* Source Account sets up a **Link** to a single or upto 5 Monitoring Accounts.
* Monitoring Account will setup as a **Sink** with two types of permissions options.
    * Account IDs
    * AWS Org IDs
* The monitoring Account can link upto 100,000 source accounts.
* There are no additional cost implication to either the source account or monitoring account


Now let's see some CDK on how to set this up.

### 1. Setting up the OAM Link and the XAXR role

This would be used to setup permisions in your **Source Account**:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as oam from 'aws-cdk-lib/aws-oam';
import { Construct } from 'constructs';

export type CloudWatchPolicy =
  | 'CloudWatch-and-AutomaticDashboards'
  | 'CloudWatch-and-ServiceLens'
  | 'CloudWatch-AutomaticDashboards-and-ServiceLens'
  | 'CloudWatch-core-permissions'
  | 'View-Access-for-all-services';

export interface CloudWatchCrossAccountProps {
  monitoringAccountIds: string[];
  policy: CloudWatchPolicy;
}

export interface CreateOamLinkProps {
  monitoringAccountId?: string; // Optional - defaults to 637423601711
  sinkIdentifier?: string; // Optional - defaults to the one in template
  labelTemplate?: string; // Optional - defaults to $AccountName
}

const POLICY_MAP: Record<CloudWatchPolicy, string[]> = {
  'View-Access-for-all-services': [
    'arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess',
    'arn:aws:iam::aws:policy/CloudWatchAutomaticDashboardsAccess',
    'arn:aws:iam::aws:policy/job-function/ViewOnlyAccess',
    'arn:aws:iam::aws:policy/AWSXrayReadOnlyAccess',
  ],
  'CloudWatch-and-AutomaticDashboards': [
    'arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess',
    'arn:aws:iam::aws:policy/CloudWatchAutomaticDashboardsAccess',
  ],
  'CloudWatch-and-ServiceLens': [
    'arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess',
    'arn:aws:iam::aws:policy/AWSXrayReadOnlyAccess',
  ],
  'CloudWatch-AutomaticDashboards-and-ServiceLens': [
    'arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess',
    'arn:aws:iam::aws:policy/CloudWatchAutomaticDashboardsAccess',
    'arn:aws:iam::aws:policy/AWSXrayReadOnlyAccess',
  ],
  'CloudWatch-core-permissions': ['arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess'],
};

export function createCloudWatchCrossAccountRole(
  scope: Construct,
  id: string,
  props: CloudWatchCrossAccountProps,
): iam.Role {
  const principals = props.monitoringAccountIds.map((accountId) => new iam.AccountPrincipal(accountId));

  const managedPolicies = POLICY_MAP[props.policy].map((policyArn) =>
    iam.ManagedPolicy.fromAwsManagedPolicyName(policyArn.replace('arn:aws:iam::aws:policy/', '')),
  );

  return new iam.Role(scope, id, {
    roleName: 'CloudWatch-CrossAccountSharingRole',
    assumedBy: new iam.CompositePrincipal(...principals),
    managedPolicies,
    path: '/',
  });
}

export function createOamLink(scope: Construct, id: string, props: CreateOamLinkProps = {}): oam.CfnLink {
  const {
    monitoringAccountId = '63742',
    sinkIdentifier = 'arn:aws:oam:us-east-1:6371711:sink/e2aa2952-d9c6-4e40-94e0-31',
    labelTemplate = '$AccountEmail',
  } = props;

  // Create condition to skip in monitoring account
  const skipMonitoringAccount = new cdk.CfnCondition(scope, `${id}SkipMonitoringAccount`, {
    expression: cdk.Fn.conditionNot(cdk.Fn.conditionEquals(cdk.Stack.of(scope).account, monitoringAccountId)),
  });

  // Create the OAM Link
  const link = new oam.CfnLink(scope, id, {
    labelTemplate,
    resourceTypes: [
      'AWS::CloudWatch::Metric',
      'AWS::Logs::LogGroup',
      'AWS::XRay::Trace',
      'AWS::ApplicationInsights::Application',
      'AWS::InternetMonitor::Monitor',
    ],
    sinkIdentifier,
  });

  // Apply the condition
  link.cfnOptions.condition = skipMonitoringAccount;

  return link;
}
```

Link to gist: https://gist.github.com/Vasu77df/22e69620cbcefe824f6d8fdb8809e8f8

You can use this in your CDK Construct or Stack

```typescript
const xaxrUwcCentralObeservability = createCloudWatchCrossAccountRole(
  scope,
  'BakeryUwcGodViewRole',
  crossAccountProps,
);
const oamLink = createOamLink(this, 'OamLink', {
  monitoringAccountId: '6374231',
  sinkIdentifier: 'arn:aws:oam:us-east-1:63742:sink/e2aa2952-d9c6-4e40-94e0-31c',
  labelTemplate: '$AccountEmail',
});
```

### 2. Setting up the Sink

Setting up the monitoring account permissions through CDK is a bit tedious and unecessarily complex, as we don't have the [Level 2 CDK Constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html#constructs_lib_levels) yet.

If you company or Org has a good standard on AWS Organizational Ids, the most simple process is to just enable it through the AWS Console, and enter you Org ID.

Here are the instruction from the AWS Documentation: 
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account-Setup.html#Unified-Cross-Account-Setup-ConfigureMonitoringAccount

![xaxr](/xaxr.png)


### 3. Conclusion

Now with this setup you can search for metrics across accounts, setup dashboards, with metrics from mulitple accounts, and even see log groups across accounts. A very nice feature is seeing X-Ray traces across resources hosted in multiple accounts.

### 4. References
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account-Setup.html
- https://aws.amazon.com/blogs/aws/new-amazon-cloudwatch-cross-account-observability/




