## Duplicate Rule Detection Tool

[AWS Config](https://aws.amazon.com/config/) continuously audits and assesses the configurations of your AWS resources by generating configuration items for supported AWS resources that exist in your account and when the configuration of a resource changes, and it maintains historical records of the configuration items of your resources from the time you start the configuration recorder.

AWS Config rules, also referred to as detective controls, continuously evaluate your AWS resource configurations for desired settings. Depending on the rule, AWS Config will evaluate your resources either in response to configuration changes or periodically. AWS Config provides AWS managed rules, which are predefined, customizable rules that AWS Config uses to evaluate whether your AWS resources comply with common best practices.

AWS Config rules can be enabled individually or through AWS Config Conformance Packs, which groups rules to help customers meet their compliance requirements. Additionally, AWS services [AWS Security Hub](https://aws.amazon.com/security-hub/) and [AWS Control Tower](https://aws.amazon.com/controltower/) Controls Library integrate with Config to deploy detective controls grouped together by specific standards or categories.

AWS customers often deploy a combination of these mechanisms to establish a more comprehensive security posture but the underlying detective controls are all deployed as individual AWS Config rules. At times, there may exist an overlap in the scope of the grouped detective controls and duplicate rules may be deployed in a single AWS account and region. Duplicate rules can contribute to a loss of efficiency and additional manual intervention when evaluating rules at scale.

## Overview of Solution

In order to solve this problem, we have built a solution to assess the current active Config rules and identify duplicates in an AWS account so customers can make informed decisions on how to streamline rules and reduce complexity.

The diagram below illustrates the solution we have created. It begins with an [Amazon EventBridge](https://aws.amazon.com/pm/eventbridge/) event triggered on the basis of a cron expression that is configurable based on the customer’s preferred schedule. This event then triggers an [AWS Lambda](https://aws.amazon.com/lambda/) function, which makes the describe-config-rules API call to AWS Config API. The Lambda function aggregates all of the config rules deployed in the account within the same region, from Security Hub Standards, Config Conformance Packs, standalone Config Rules, and Control Tower Library. Then, the Lambda function iterates through all of the deployed Config rules to determine whether there are any duplicate rules. In order to be considered duplicates, Config rules need to have identical sources, scopes, input parameters and states. If any duplicates are found, they are grouped together in JSON format. The Lambda function takes the JSON of the duplicate rules, timestamps it, and saves it to an Amazon S3 bucket for further analysis. With this solution, customers have a quick way to identify any duplicate rules within their environment and take action accordingly.

![Figure 1. Architectural diagram of the Duplicate Rule Detection Tool.](/Rule-Dup-Architecture.png)

## Walkthrough

For this demonstration, we are using an AWS account that has two Config Conformance Packs deployed ([Operational Best Practices for HIPAA Security](https://docs.aws.amazon.com/config/latest/developerguide/operational-best-practices-for-hipaa_security.html) and [Operational Best Practices for NIST CSF](https://docs.aws.amazon.com/config/latest/developerguide/operational-best-practices-for-nist-csf.html)) along with the [AWS Foundational Security Best Practices (FSBP)](https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html) standard in Security Hub.

### AWS CloudFormation template review

The CloudFormation template included in this blog will deploy several necessary components:

- DuplicateRuleDetectionLambda - An AWS Lambda function that:
  - Calls the AWS Config describe_config_rules API to return all enabled Config rules
  - Examines the returned Config rules to identify duplicate rules with identical parameters
  - Writes the date-stamped output JSON file to the DetectionLambdaResultsBucket bucket
- DetectionLambdaPolicy - An AWS IAM policy attached to the DetectionLambdaRole role that allows access to:
  - Basic Lambda Execution Permissions
  - config:DescribeConfigRules
  - s3:PutObject with a constraint to only allow on the DetectionLambdaResultsBucket bucket
- DetectionLambdaRole - An AWS IAM role with a trust policy to allow only the AWS Lambda service to assume
- DetectionLambdaResultsBucket - An Amazon S3 bucket for storing the output JSON files written by the DuplicateRuleDetectionLambda function
- DetectionLambdaEventBridgeRule - This Amazon EventBridge rule is used to trigger the DuplicateRuleDetectionLambda that detects duplicate Config rules deployed in the AWS account
  - ScheduledExpression property can be used to configure reoccurence

### Prerequisites

An AWS account with detective controls enabled, which may include AWS Config conformance packs, AWS Security Hub standards, and/or AWS Control Tower controls.

### Deploy the solution

> **Note:** You must have IAM permissions to launch CloudFormation templates that create IAM roles, and to create all the AWS resources in the solution. Also, you are responsible for the cost of the AWS services used while running this solution. For more information about costs, see the pricing pages for each AWS service.

1. Download the duplicate-rule-detection.yml CloudFormation template from this GiHub repository.
   1. Make any necessary changes to template, by default, the ScheduledExpression for the EventBridge rule is set for monthly reocurrence.
2. Log in to the **AWS CloudFormation** console.
3. From the sidebar on this page, choose **Stacks**.
4. At the top of the Stacks page, choose **Create Stack** and then **With new resources** from the dropdown menu.
5. On the **Create stack** page
   1. For Prerequisite - Prepare template, leave the default **Template is ready**
   2. Under Specify template, choose **Upload a template** file, then select the downloaded duplicate-rule-detection.yml template and click **Open**.
6. At the bottom of the page, choose **Next**.
7. On the Specify stack details page
   1. For **Stack name**, provide a name for the Stack. You can use: **DuplicateRulesStack**.
8. At the bottom of the page, choose **Next**.
9. On the **Configure stack options** page
   1. For Tags, add any desired tags. Tags are optional.
   2. For Permissions, don't choose a role, CloudFormation uses permissions based on your user credentials.
   3. For Stack failure options, leave the default option to **Roll back all stack resources**
10. At the bottom of the page, choose **Next**.
11. On the **Review and create** page, review the details of your stack. Because this solution creates an IAM policy and role, you will need to check the box next to **I acknowledge that AWS CloudFormation might create IAM resources**.
12. After you review the stack creation settings, choose **Submit** to launch your stack
13. From the CloudFormation Stack page, monitor the status of the DuplicateRulesStack as it updated from **“CREATE_IN_PROGRESS”** to **“CREATE_COMPLETE”**. You may need to refresh the page to view updates.
    14.From the **Resources** tab, you will see all of the resources that were created from the template.

### Testing/Validation

The output of the Lambda function is a JSON file written to an S3 bucket. Each duplicate rule is presented as an object and are grouped together in an array.
![Figure 2. Screen shot of solution output.](/Rule-Dup-Output.png)

From the output, we can see that in this account we have three instances of the same Config managed rule:

- The “SourceIdentifier” key value identifies the managed rule as ACCESS_KEYS_ROTATED (noted in red).
- The “CreatedBy” key value identifies the service that enabled the rule (noted in green); both

Each rule has the same InputParameters, which is a qualifier for how we are defining a duplicate rule. There could be a use case that a customer may have a need to deploy multiple variations of the same rule with distinct input parameters.

Now that the duplicate rules have been identified, further investigation may be required to identify the specific conformance pack and Security Hub standards that the rule is included in.
![Figure 3. Screen shot of AWS Config conformance pack dashboard.](/Conf-Pack.png)

Each output object contains a ConfigRuleName key value pair that includes the prefixes and suffixes that may be applied to uniquely identify each rule and that ties back to the specific conformance pack. From the Config Conformance Pack dashboard, you can search rules by name to map to the corresponding conformance pack.
![Figure 4. Screen shot of AWS Security Hub dashboard.](/Sec-Hub.png)

For Security Hub security standards dashboard, you can search for the specific control and identify the corresponding config rule.

## Cleaning up

Take the following steps to remove the resources you created in this walkthrough:

### Delete Stack:

Note: Any objects added to the solution bucket after initial stack deployment will will need to be deleted before attempting to delete Stack.)

1. Sign in to the console of the AWS account and navigate to the **CloudFormation** console.
2. From the sidebar on this page, choose **Stacks**.
3. Select the radio button next to the stack name used in the deployment step and select **Delete**.
4. Confirm that you would like to Delete stack by choosing the **confirmation** button.
5. From the CloudFormation Stack page, monitor the status of the stack as it updated from **“DELETE_IN_PROGRESS”** to **“DELETE_COMPLETE”**.

## Conclusion

For customers in regulated industries, it’s critical to understand the compliance of resources as it relates to specific rules, such as default encryption settings or ensuring network connections are encrypted. Detective controls enable customers to evaluate the evolving state of their resources on AWS. Detective controls in AWS are enabled as AWS Config rules and can be deployed individually or grouped together in Config conformance packs or by proxy via Security Hub standards and Control Tower Controls Library. Many customer will use more than one of these mechanisms and this can result in duplicate rules being deployed in an AWS account. This solution provides a tool to assess the currently deployed Config rules in a single AWS account and identify when duplicate rules exist so customers can make informed decisions about any changes that may be necessary to reduce the complexity of their compliance posture and reduce the manual intervention required for ongoing management.

## Additional Resources/Call to Action

Once the assessment is complete and duplicate rules are identified, further investigation can be completed to identify the sources of the deployed Config rules. If the AWS account being evaluated is part of an AWS Organizations, then dependencies may exist.

Rules deployed via Conformance Packs will include a suffix to the displayed Config rule name that includes “conformance-pack-” followed by a string of alpha-numeric characters, which logically represents a specific Conformance Pack.

Rules deployed via Security Hub standards will include both a prefix and suffix to the displayed Config rule name that begins “securityhub-” and also followed by a random string of alpha-numeric characters

- If conformance packs were deployed from AWS Systems Manager Quick Start, then only sample templates are available, which can’t be modified, and custom templates can’t be used.
- If conformance packs were deployed from the Config dashboard, then custom templates can be used.
- If standards are enabled in Security Hub, individual controls can be disabled to prevent redundancy.
- If controls are deployed from the Control Tower Controls Library, then steps can be taken to consolidate controls.

After the sources of the duplicate rules have been identified, customers can make decisions if changes are required and prioritize the most appropriate method for consolidating rule deployment based on service functionality and how compliance would be most effectively aggregated.

## Below are some additional configurations that should be implemented when deploying this solution to align with security best practices.

When you set permissions with IAM policies, grant only the permissions required to perform a task. You do this by defining the actions that can be taken on specific resources under specific conditions, also known as least-privilege permissions. Some IAM actions (ex. config:DescribeConfigRules in this solution) only support the all resources wildcard("\*"). In such use cases, further consideration can be taken with the use of AWS global condition context keys (https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) and will be based on individual requirements.

AWS Lambda function should be deployed inside a VPC for additional networking configurations, such as access to define the VPC security groups and subnets that are attached to a Lambda function. When you connect a function to a VPC, Lambda creates an elastic network interface for each combination of security group and subnet in the function's VPC configuration.
Lambda Developer Guide - https://docs.aws.amazon.com/lambda/latest/dg/foundation-networking.html
CloudFormation resource reference - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-vpcconfig.html

Monitoring Amazon Simple Storage Service (S3) buckets should include logging server access. Server access logging provides detailed records for the requests that are made to a bucket.
S3 User Guide - https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html
CloudFormation resource reference - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket-loggingconfiguration.html

Amazon S3 applies server-side encryption with Amazon S3 managed keys (SSE-S3), by default, as the base level of encryption for every bucket in Amazon S3. Other encryption options are also available and should be implemented, including specifying server-side encryption with AWS KMS-managed keys (SSE-KMS)
S3 User Guide - https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html
CloudFormation resource reference - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket-bucketencryption.html


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

