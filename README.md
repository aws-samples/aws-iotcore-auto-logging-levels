# Using automation to control AWS IoT Core v2 logging levels

## Introduction
In this repository, we'll describe and demonstration a solution for automating the adjustment of the level of logging in AWS IoT Core. The example use-case for this solution is we need to configure out AWS IoT Core logging to a log level of INFO for only 5 minutes, during peak usage hours, and then change that log level back to ERROR of logging for the rest of the day. This automation provides a signifcant cost-savings and allows you to record the level of logs you need at the time you need them.

## Walkthrough
For the demonstration in this repository, we'll use AWS Step Functions to orchestrate the following tasks:
1. Configure the logging level of AWS IoT Core to <b>INFO</b>
2. Send a notification using Amazon SNS to let subscribers know this automation has started 
3. Delay the Step Function by 5 minutes
4. Configure the logging level of AWS IoT Core to <b>Error</b>
5. Send a notification using Amazon SNS to let subscribers know this automation has ended 

The illustration below details what this solution will look like once fully implemented.

<img src="./assets/Solution%20Overview.png" />

<br /> 

### Prerequisites
To follow through this repository, you will need an <a href="https://console.aws.amazon.com/" >AWS account</a>, an <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/" >AWS IoT Core supported region</a>, permissions to <a href="https://docs.aws.amazon.com/iot/latest/developerguide/configure-logging.html" >configure AWS IoT logging </a>, create <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" > AWS Identity and Access Management (IAM) roles and policies</a>, create <a href="https://docs.aws.amazon.com/sns/latest/dg/sns-create-topic.html"> AWS Step Functions</a>, and access to the <a href="https://aws.amazon.com/cli/">AWS CLI</a>. We also assume you have familiar with the basics of Linux bash commands.

## Cleaning up
Be sure to remove the resources created in this blog to avoid charges. Run the following commands to delete these resources:

1. aws 

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

