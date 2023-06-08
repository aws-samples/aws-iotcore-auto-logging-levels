# Using automation to control AWS IoT Core v2 logging levels

## Introduction
This describes one method to adjust the level of IoT logging for a 
predetermined length of time and on a schedule you set. Example use-case 
for this would be to log full INFO level of logging for only 5 minutes during 
peak usage hours then change back to ERROR level of logging for the rest of 
the day. While not covered in this article you could use the same AWS Step 
Function to turn on INFO or DEBUG level if a certain event happens and use 
IoT Rule Actions to start the AWS Step Function.

## Walkthrough
The Step Function we will be building performs the
following tasks – Change IoTv2loggingOptions setting to
INFO, send a notification via SNS to topic, wait 5
minutes, change IoTv2LoggingOptions to ERROR, and
send a notification to the same topic informing the
subscribers the logging has changed to ERROR.

We will assume you already have a SNS topic already configured. If not 
please follow these instructions here. 
https://docs.aws.amazon.com/sns/latest/dg/sns-getting-started.html

<img src="./assets/Solution%20Overview.png" />

### Prerequisites

## Cleaning up
## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

