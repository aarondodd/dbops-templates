# Description

This example shows how to update an RDS instance using CloudFormation and preview the steps first, allowing you to ensure resources aren't unexpectedly destroyed. This uses CloudFormation [Change Sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)

Change sets can be created anytime as they do not alter the environment. This also allows you to stage changes for execution at a later time. This can also aid a change-control process by submitting the running template and Change Set results for review.

This differs from the `aws cloudformation update` call, which does not show the changes ahead of time but executes as defined.

# Initial launch
From: https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/create-stack.html

1. RdsTest1.template is the example CFT, validate and make any changes.
2. create_parameters.json are the initial settings to apply on launch, validate and make any changes. For this example I've kept the Engine below the latest and the InstanceType small, as those will be altered via a ChangeSet next
3. Execute:

```
aws cloudformation create-stack --stack-name RdsTest2 --template-body file://RdsTest1.template --parameters file://create_parameters.json
```

4. Check status with:

```
aws cloudformation describe-stacks --stack-name RdsTest2 | grep -i stackstatus
```

When you see `CREATE_COMPLETE`, test updating.

# Test updating

Use-cases tested:
- Resizing the running instance (altering InstanceType)
- Upgrading the engine version (moving to newer point release)

Expectation:
- Infrastructure is modified as defined
- Database isn't destroyed in the process

## Just changing parameters

In this test, we're leaving the running stack itself alone and just changing the parameters previously used. The source template remains untouched.

1. Compare create_parameters.json and update_parameters.json. Paramters we're leaving alone have the ParameterValue removed and replaced with UsePreviousValue. Parameters we're changing have updated values. Parameters we're changing have updated values.
2. Create a [change set](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/create-change-set.html):

```
aws cloudformation create-change-set --stack-name RdsTest2 --use-previous-template --parameters file://update_parameters.json --change-set-name ParamsUpdateTest
```

3. [Describe the change set](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/describe-change-set.html) with:

```
aws cloudformation describe-change-set --change-set-name ParamsUpdateTest --stack-name RdsTest2
```

4. Verify that no recreations are needed (look for "Replacement" and "RequiresRecreation" entries other than False/Never)
5. [Execute the change set](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/execute-change-set.html):

```
aws cloudformation execute-change-set --change-set-name ParamsUpdateTest --stack-name RdsTest2
```

6. To see the operation in progress, use the describe-stacks command as per above. To see full output of the CloudFormation run, use the [describe-stack-events](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/describe-stack-events.html) call:

```
aws cloudformation describe-stack-events --stack-name RdsTest2
```

You could also use the [deploy](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/deploy.html) call instead, which creates and then executes a change set.

## Changing a template value

For this test, either change a value in the RdsTest1.template, or download the running template:

```
aws cloudformation get-template --stack-name RdsTest2 > running.template
```

1. For this test, add the following to the MyRDSParamGroup,'s Parameters section:

```
"sql_mode": "IGNORE_SPACE",
"max_allowed_packet": 1024
```

i.e.:

```
            "MyRDSParamGroup": {
                "Type": "AWS::RDS::DBParameterGroup",
                "Properties": {
                    "Family": "MySQL8.0",
                    "Description": "CloudFormation Sample Database Parameter Group",
                    "Parameters": {
                        "autocommit": "1",
                        "general_log": "1",
                        "sql_mode": "IGNORE_SPACE",
                        "max_allowed_packet": 1024
                    }
                }
            }
        },

```

The template "RdsTest1_updated.template" has the changes per above. If downloading the running template, you'll need to ensure only the contents of the "TemplateBody" block are in the file you're referencing in the below command.

2. Create the change set, this time specifying the template and parameters:

```
aws cloudformation create-change-set --stack-name RdsTest2 --template-body file://RdsTest1_updated.template --parameters file://update_template_parameters.json --change-set-name EditedTemplateUpdateTest
```

3. Describe the new change set:

```
aws cloudformation describe-change-set --change-set-name EditedTemplateUpdateTest --stack-name RdsTest2
```

4. Deploy the changes:

```
aws cloudformation execute-change-set --change-set-name EditedTemplateUpdateTest --stack-name RdsTest2
```