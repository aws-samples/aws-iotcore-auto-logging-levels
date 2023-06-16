# Using automation to control AWS IoT Core v2 logging levels

## Introduction
In this repository, we'll describe and demonstrate a solution for automating the adjustment of the level of logging in AWS IoT Core. The example use-case for this solution is we need to configure our AWS IoT Core logging level to INFO for only 5 minutes, during peak usage hours, and then change that log level back to ERROR of logging for the rest of the day. This automation provides a signifcant cost-savings and allows you to record the level of logs you need at the time you need them.

## Walkthrough
For the demonstration in this repository, Amazon EventBridge will start the AWS Step Function at a scheduled time of the day. This AWS Step Function will then orchestrate the following tasks:
1. Configure the logging level of AWS IoT Core to <b>INFO</b>
2. Send a notification using Amazon SNS to let subscribers know this automation has started 
3. Wait for 5 minutes
4. Configure the logging level of AWS IoT Core to <b>Error</b>
5. Send a notification using Amazon SNS to let subscribers know this automation has ended 

The illustration below details what this solution will look like once fully implemented.

<img src="./assets/Solution%20Overview.png" />

<br /> 

### Prerequisites
To follow through this repository, you will need an <a href="https://console.aws.amazon.com/" >AWS account</a>, an <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/" >AWS IoT Core supported region</a>, permissions to <a href="https://docs.aws.amazon.com/iot/latest/developerguide/configure-logging.html" >configure AWS IoT logging </a>, create <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" > AWS Identity and Access Management (IAM) roles and policies</a>, create <a href="https://docs.aws.amazon.com/sns/latest/dg/sns-create-topic.html"> AWS Step Functions</a>, create <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rules.html">Amazon EventBridge rules</a>, and access to the <a href="https://aws.amazon.com/cli/">AWS CLI</a>. We also assume you have familiar with the basics of Linux bash commands.


### Step 1: Create the Amazon SNS topic and subscription (AWS CLI)
1. Create your Amazon SNS topic by issuing the <b>create-topic</b> command
    ```   
    aws sns create-topic --name "aws_iot_core_logging_levels_demo_topic"
    ```

    The output from the command should look similar to the following:
    ```
    {
        "TopicArn": "arn:aws:sns:AWSREGION:AWSACCOUNTID:aws_iot_core_logging_levels_demo_topic"
    }
    ```
2. Create your Amazon SNS subscription by issuing the <b>subscribe</b> command
    ```   
    aws sns subscribe --topic-arn "Replace_Me_With_SNS_Topic_ARN" --protocol email --notification-endpoint "Replace_Me_With_Email_Address"
    ```

    The output from the command should look similar to the following:
    ```
    {
        "SubscriptionArn": "pending confirmation"
    }
    ```
3. Confirm your subscription by using the <b>Subscription Confirmation</b> email that was sent to the email address you used in the previous command
### Step 2: Create the AWS Step Function (AWS CLI)
1. Create the IAM execution role for the Step function by issuing the <b>create-role</b> command
    ```
    aws iam create-role --role-name "aws_stepfunction_iotcore_logging_demo_role" --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }'
    ```
2. Use the below command to create the policy document for the role’s permissions
   ```
   echo '{                         
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "sns:*",
                "iot:GetV2LoggingOptions",
                "iot:SetV2LoggingOptions",
                "iam:PassRole",
                "iot:SetV2LoggingLevel",
                "iot:ListV2LoggingLevels",
                "iot:DeleteV2LoggingLevel"
            ],
            "Resource": "*"
        }
    ]}' >  iot_core_logging_policy.json
   ```
    <b>NOTE)</b> This IAM policy is to be use for development purposes only. We recommend following the best practice of least privilege with your IAM policy. You should consider scoping this down if deploying into a production account. 

3. Attach the IAM policy to the IAM role by issuing the <b>put-role-policy</b> command
    ```
    aws iam put-role-policy \
    --role-name "aws_stepfunction_iotcore_logging_demo_role" \
    --policy-name "aws_stepfunction_iotcore_logging_demo_policy" \
    --policy-document file://iot_core_logging_policy.json
    ```

4. Create your AWS Step Function by issuing the <b>create-state-machine</b> command
    ```
    aws stepfunctions create-state-machine --name "aws_stepfunction_iotcore_logging_demo" --definition '{
    "StartAt": "SetV2LoggingOptions",
    "States": {
        "SetV2LoggingOptions": {
        "Type": "Task",
        "Parameters": {
            "DefaultLogLevel": "INFO",
            "RoleArn": "arn:aws:iam:::role/service-role/IoT_Role_For_Logs"
        },
        "Resource": "arn:aws:states:::aws-sdk:iot:setV2LoggingOptions",
        "Next": "SNS Publish"
        },
        "SNS Publish": {
        "Type": "Task",
        "Resource": "arn:aws:states:::sns:publish",
        "Parameters": {
            "TopicArn": "Replace_Me_With_SNS_Topic_ARN",
            "Message": {
            "DefaultLogLevel": "INFO"
            }
        },
        "Next": "Wait"
        },
        "Wait": {
        "Type": "Wait",
        "Seconds": 300,
        "Next": "SetV2LoggingOptions (1)"
        },
        "SetV2LoggingOptions (1)": {
        "Type": "Task",
        "Parameters": {
            "DefaultLogLevel": "ERROR",
            "DisableAllLogs": false,
            "RoleArn": "arn:aws:iam:::role/service-role/IoT_Role_For_Logs"
        },
        "Resource": "arn:aws:states:::aws-sdk:iot:setV2LoggingOptions",
        "Next": "SNS Publish (1)"
        },
        "SNS Publish (1)": {
        "Type": "Task",
        "Resource": "arn:aws:states:::sns:publish",
        "Parameters": {
            "Message": {
            "DefaultLogLevel": "ERROR"
            },
            "TopicArn": "Replace_Me_With_SNS_Topic_ARN"
        },
        "Next": "Success"
        },
        "Success": {
        "Type": "Succeed"
        }
    },
    "Comment": "Turn Logging Level to Info for set period of time then change it back to Error",
    "TimeoutSeconds": 1200
    }' --role-arn "Replace_Me_With_Role_ARN_From_Step3"
    ```
### Step 3: Create the Amazon EventBridge rule (AWS CLI)
1. Create the IAM execution role for the EventBridge rule by issuing the <b>create-role</b> command
    ```
    aws iam create-role --role-name "aws_stepfunction_iotcore_logging_demo_eventbridge_role" --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }'
    ```
2. Use the below command to create the policy document for the role’s permissions
   ```
   echo '{                         
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "sns:*",
                "iot:GetV2LoggingOptions",
                "iot:SetV2LoggingOptions",
                "iam:PassRole",
                "iot:SetV2LoggingLevel",
                "iot:ListV2LoggingLevels",
                "iot:DeleteV2LoggingLevel"
            ],
            "Resource": "*"
        }
    ]}' >  iot_core_logging_eventbridge_policy.json
   ```
    <b>NOTE)</b> This IAM policy is to be use for development purposes only. We recommend following the best practice of least privilege with your IAM policy. You should consider scoping this down if deploying into a production account. 

3. Attach the IAM policy to the IAM role by issuing the <b>put-role-policy</b> command
    ```
    aws iam put-role-policy \
    --role-name "aws_stepfunction_iotcore_logging_demo_eventbridge_role" \
    --policy-name "aws_stepfunction_iotcore_logging_demo_eventbridge_policy" \
    --policy-document file://iot_core_logging_eventbridge_policy.json
    ```
4. Create your Amazon EventBridge rule by issuing the <b>put-rule</b> command
    ```
    aws events put-rule --name "aws_iot_core_logging_levels_demo_event_rule" --schedule-expression "cron(5,35 14 * * ? *)" --role-arn "Replace_Me_With_Role_ARN_From_Step3"
    ```
    * This example creates a rule that runs every day, at 2:05pm and 2:35pm UTC+0.
  
    </br>
5. Add your AWS Step Function as a target to your Amazon EventBridge rule by issuing the <b>put-targets</b> command

    ```
    aws events put-targets --rule "Replace_Me_With_Rule_ARN_From_Step1" --targets "Replace_Me_With_ARN_Of_AWS_Step_Function_State_Machine"
    ```

## Cleaning up
Be sure to remove the resources created in this blog to avoid charges. Run the following commands to delete these resources:

1. aws sns delete-topic --topic-arn "Replace_Me_With_SNS_Topic_ARN"
2. aws sns unsubscribe --subscription-arn "Replace_Me_With_SNS_Subscription_ARN"
3. aws iam delete-role-policy --role-name "aws_stepfunction_iotcore_logging_demo_role"  --policy-name "aws_stepfunction_iotcore_logging_demo_policy"
4. rm iot_core_logging_policy.json
5. aws iam delete-role --role-name "aws_stepfunction_iotcore_logging_demo_role"
6. aws stepfunctions delete-state-machine --state-machine-arn "Replace_Me_With_ARN_Of_AWS_Step_Function_State_Machine"
7. aws iam delete-role-policy --role-name "aws_stepfunction_iotcore_logging_demo_eventbridge_role"  --policy-name "aws_stepfunction_iotcore_logging_demo_eventbridge_policy"
8. rm iot_core_logging_eventbridge_policy.json
9. aws iam delete-role --role-name "aws_stepfunction_iotcore_logging_demo_eventbridge_role"
10. aws stepfunctions delete-state-machine --state-machine-arn "Replace_Me_With_ARN_Of_AWS_Step_Function_State_Machine"
11. aws events delete-rule --name "aws_iot_core_logging_levels_demo_event_rule" 

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

