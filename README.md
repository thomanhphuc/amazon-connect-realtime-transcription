## Project Overview
The purpose of this project is to provide an example solution to get you started with capturing and transcribing Amazon Connect audio using Kinesis Video Streams and Amazon Transcribe. The example Lambda functions can be used to create varying solutions such as capturing audio in the IVR and transcribing customer audio. To enable these different use-cases there are multiple [environment variables](#lambda-environment-variables) environment variables and parameters in the [invocation event](#lambda-invocation-event-details) that control the behavior of the Lambda Function.

## Architecture Overview
![images/ivr-recording-architecture.png](https://raw.githubusercontent.com/thomanhphuc/amazon-connect-realtime-transcription/master/images/ivr-recording-architecture.png)

## Getting Started
Getting started with this project is easy. The most basic use case of capturing audio in the Amazon Connect IVR can be accomplished by downloading the pre-packaged Lambda Functions, deploying the CloudFormation template in your account, and importing the Contact Flows into your Amazon Connect Instance. 

### Easy Setup
- Clone the github repo into your account.
- Create an S3 bucket and create a new folder “deployment” and upload the deployment/ folder into it
    - Open the `cloudformation.template` file and copy the S3 url on it's detail page
- Go to CloudFormation and select 'Create Stack'.
    - Create the stack from an S3 url and paste the url from the cloudformation.template file
    - Fill in the parameters for the stack. The existingS3BucketName and existingS3Path should be the ones created above that contain all the deployment related code.
![images/cloud-formation-stack-parameters.png](https://github.com/thomanhphuc/amazon-connect-realtime-transcription/blob/master/images/cloud-formation-stack-parameters.png)  
- While the stack is building, go to the Amazon Connect AWS console and ensure that your Amazon Connect instance has the "live media streaming" feature enabled by following the [Amazon Connect documentation](https://docs.aws.amazon.com/connect/latest/userguide/customer-voice-streams.html) for "Enable Live Media Streaming"
- Once the stack is complete you will need to add the Lambda function to your Connect Instance. In the AWS Console open the Amazon Connect management console, select the Instance you would like to add the IVR recording capabilities to, go to the Contact Flows menu, and then the AWS Lambda section. Find the Lambda function with `kvsConsumerTrigger` in the name in the list and select Add Lambda Function.
![images/connect-lambda-configuration.png](https://raw.githubusercontent.com/thomanhphuc/amazon-connect-realtime-transcription/master/images/connect-lambda-configuration.png)
- Now go to the S3 management console, open the bucket that you created with the `deployment/` folder, and download the Contact Flow. The flow is called kvsStreamingSampleFlow.json.
- Log into your Amazon Connect instance and import the Contact Flow.
![images/connect-import-flow.png](https://raw.githubusercontent.com/thomanhphuc/amazon-connect-realtime-transcription/master/images/connect-import-flow.png)
- In the Contact Flow edit the Lambda function that is configured in the Invoke Lambda Function block and select the name of the kvsConsumerTrigger Lambda function that was deployed by the Cloudformation template.  
![images/connect-configure-lambda-block.png](https://raw.githubusercontent.com/thomanhphuc/amazon-connect-realtime-transcription/master/images/connect-configure-lambda-block.png)
- Click save and publish the Contact Flow
- In your Amazon Connect instance, claim a Phone Number and assign the Contact Flow you created to it and call the number. Depending on the settings in the KvsTranscriber Lambda Function, the audio will be saved in S3 and the transcriptions will be visible in DynamoDB.
![images/connect-configure-phone-number.png](https://raw.githubusercontent.com/thomanhphuc/amazon-connect-realtime-transcription/master/images/connect-configure-phone-number.png)

### Building the KVS Transcriber project
The lambda code is designed to be built with Gradle. All requisite dependencies are captured in the `build.gradle` file. Simply use `gradle build` to build the zip that can be deployed as an AWS Lambda application. After running `gradle build`, the updated zip file can be found in the `build/distributions` folder; copy it to the `deployment` folder then follow the Easy Setup steps above.  

Use github action to deploy .github/workflows/gradle-publish.yml

### Lambda Environment Variables
This Lambda Function has environment variables that control its behavior:
* `APP_REGION` - The region for AWS DynamoDB, S3 and Kinesis Video Streams resources (ie: us-east-1)
* `TRANSCRIBE_REGION` - The region to be used for AWS Transcribe Streaming (ie: us-east-1)
* `RECORDINGS_BUCKET_NAME` - The AWS S3 bucket name where the audio files will be saved (Lambda needs to have permissions to this bucket)
* `RECORDINGS_KEY_PREFIX` - The prefix to be used for the audio file names in AWS S3
* `RECORDINGS_PUBLIC_READ_ACL` - Set to TRUE to add public read ACL on audio file stored in S3. This will allow for anyone with S3 URL to download the audio file.
* `INPUT_KEY_PREFIX` - The prefix for the AWS S3 file name provided in the Lambda request. This file is expected to be present in `RECORDINGS_BUCKET_NAME`
* `CONSOLE_LOG_TRANSCRIPT_FLAG` - Needs to be set to TRUE if the Connect call transcriptions are to be logged.
* `TABLE_CALLER_TRANSCRIPT` - The DynamoDB table name where the transcripts of the audio from the customer need to be saved (Table Partition key must be: `ContactId`, and Sort Key must be: `StartTime`)
* `TABLE_CALLER_TRANSCRIPT_TO_CUSTOMER` - The DynamoDB table name where the transcripts of the audio to the customer need to be saved (Table Partition key must be: `ContactId`, and Sort Key must be: `StartTime`)
* `SAVE_PARTIAL_TRANSCRIPTS` - Set to TRUE if partial segments need to saved in the DynamoDB table. Else, only complete segments will be persisted.
* `START_SELECTOR_TYPE` - Set to NOW to get transcribe once the agent and user are connected. Set to FRAGMENT_NUMBER to start transcribing once the 'Start Media Streaming' block is executed in your contact flow

#### Sample Lambda Environment Variables
![images/env-variables-example.png](https://raw.githubusercontent.com/thomanhphuc/amazon-connect-realtime-transcription/master/images/env-variables-example.png)

### Lambda Invocation Event Details
This Lambda Function will need some details when invoked:
* `streamARN` - The ARN of the Kinesis Video stream that includes the customer audio, this is provided by Amazon Connect when streaming is started successfully
* `startFragmentNum` - Identifies the Kinesis Video Streams fragment in which the customer audio stream started, this is provided by Amazon Connect when streaming is started successfully
* `connectContactId` - The Amazon Connect Contact ID, this is always present in the Amazon Connect invocation event.
* `transcriptionEnabled` - An optional flag to instruct the Lambda function if transcription (using Amazon Transcribe) is to be enabled or not (options are "true" or "false")
* `saveCallRecording` - An optional flag to instruct the Lambda function to upload the saved audio to S3 (options are "true" or "false")
* `languageCode` - An optional flag to instruct the Lambda function on what language the source customer audio is in, as of this writing the options are: "en-US" or "es-US" (US-English, or US-Spanish)
* `streamAudioFromCustomer` - An optional flag to instruct the Lambda function on whether to stream audio from the customer. It is true by default (options are "true" or "false")
* `streamAudioToCustomer` - An optional flag to instruct the Lambda function on whether to stream audio to the customer. It is true by default (options are "true" or "false")
