<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/ebs-lifecycle-policy/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# EBS Lifecycle Policy CloudFormation Blueprint

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

This blueprint provisions [AWS::DLM::LifecyclePolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dlm-lifecyclepolicy.html#cfn-dlm-lifecyclepolicy-description) backup policy.  It will:

* Backup EBS Volumes every 12 hours by default. It can be configured via `@create_rule_interval`
* Retain 28 copies of your EBS Snapshot backups. So you have 2 weeks worth of backups. This can also be configured via `@retain_rule_count`.
* The default resource type backed up is INSTANCE. Only resources tagged with `Backup=true` are backed up.
* This can be deployed across AWS account, so you can have the same backup policies.

## Usage

1. Add blueprint to Gemfile
2. Configure: configs/ebs-lifecycle-policy values
3. Deploy

## Add

Add the blueprint to your lono project's `Gemfile`.

```ruby
gem "ebs-lifecycle-policy", git: "git@github.com:boltopspro/ebs-lifecycle-policy.git"
```

## Configure

First, you want to configure the [configs](https://lono.cloud/docs/core/configs/) files. Use [lono seed](https://lono.cloud/reference/lono-seed/) to configure starter values quickly.

    LONO_ENV=development lono seed ebs-lifecycle-policy

For additional environments:

    LONO_ENV=production  lono seed ebs-lifecycle-policy

The generated files in `config/ebs-lifecycle-policy` folder look something like this:

    configs/ebs-lifecycle-policy/
    └── variables
        └── development.rb

## Deploy

Use the [lono cfn deploy](http://lono.cloud/reference/lono-cfn-deploy/) command to deploy. Example:

    lono cfn deploy ebs-lifecycle-policy --sure

## Configure Details

### Configure Schedule Entirely

You have full control over how you want the schedule to be configured with variables. Here's an example:

```ruby
# AWS::DLM::LifecyclePolicy
@lifecycle_policy_description = "Daily Backups"
@resource_types = ["VOLUME"] # lifecycle policies only support one element, so ["INSTANCE"] or ["VOLUME"]
@target_tags = [{Key: "Backup", Value: true}]
@create_rule_interval = 12 # every 12 hours
@retain_rule_count = 28 # keep 28 copies

# Uncomment for complete control of the schedule
# @schedules = [{
#   Name: "Daily Snapshots",
#   TagsToAdd: [
#     {
#       Key: "Type",
#       Value: "DailySnapshot"
#     }
#   ],
#   CreateRule: {
#     Interval: create_rule_interval,
#     IntervalUnit: "HOURS",
#     Times: ["18:00"]
#   },
#   RetainRule: {
#     Count: retain_rule_count
#   },
#   CopyTags: true
# }]
```

### Backup both Types: INSTANCE and VOLUME

EBS Lifecycle Policies only support one resource type. So to back up both resource types, simply create 2 lifecycle policies. Configure the configs:

configs/ebs-lifecycle-policy/variables/backup-volumes.rb:

```ruby
@lifecycle_policy_description = "Daily Backups Volumes"
@resource_types = ["VOLUME"] # lifecycle policies only support one element, so ["INSTANCE"] or ["VOLUME"]
```

configs/ebs-lifecycle-policy/variables/backup-instances.rb:

```ruby
@lifecycle_policy_description = "Daily Backups Instances"
@resource_types = ["INSTANCE"] # lifecycle policies only support one element, so ["INSTANCE"] or ["VOLUME"]
```

And deploy:

    lono cfn deploy backup-volumes   --blueprint ebs-lifecycle-policy --sure
    lono cfn deploy backup-instances --blueprint ebs-lifecycle-policy --sure

### Default Resource Type

Updating LifeCycle `PolicyDetails.ResourceTypes` with the `@resource_types` after you have deployed the stack once results in a rollback:

    10:58:19PM UPDATE_FAILED AWS::DLM::LifecyclePolicy LifecyclePolicy The following parameter(s) cannot be updated: ResourceTypes (Service: AmazonDLM; Status Code: 400; Error Code: InvalidRequestException; Request ID: bae272fd-f4fa-4f28-abf4-732114686cb6)

EBS Lifecycle Policies do not support changing this property. So you will have to delete the stack and redeploy a new stack, to change that property. Example:

    lono cfn delete ebs-lifecycle-policy
    lono cfn deploy ebs-lifecycle-policy --sure
