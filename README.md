# voice-note-generation-pipeline-using-amazon-polly-and-s3

### Project 
AI-Powered Voice Note Generator & Uploader with Amazon Q Support

### Project Overview
1. Generate a meeting script using Amazon Q
2. Convert it to speech using Amazon Polly
3. Upload the audio to S3 using Boto3
(Optional: Transcribe it back using Amazon Transcribe)

### Prerequisites
1. Create a User Group with Permissions
  - Go to IAM Console: https://console.aws.amazon.com/iam
  - Create a new user group: `PollyS3Users`
  - Attach these managed policies:
    - `AmazonPollyFullAccess` – for Polly
    - `AmazonS3FullAccess` – or create a scoped S3 policy for least privilege
      - ```{
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Action": [
                "polly:SynthesizeSpeech"
              ],
              "Resource": "*"
            },
            {
              "Effect": "Allow",
              "Action": [
                "s3:PutObject",
                "s3:GetObject"
              ],
              "Resource": "arn:aws:s3:::your-bucket-name/*"
            }
          ]
        }```


2. Create a New IAM User
  - Go to Users → Create user
  - Set a user name: `polly-s3-user`
  - Set permissions: select `Add user to group` and click on the `PollyS3Users` group
  - Click on the created user
  - Click `Create access key`
    - Select use case as `Command Line Interface (CLI)
  - Download the credentials (csv) file (Access key ID + Secret access key)

### Set up
#### Visual Studio Code
1. Create a virtual environment
2. View > New Terminal > Switch to Command Prompt
3. Navigate to the folder where a virtual environment will be created
  - You may also create a folder: `mkdir aws-amazon-polly-s3` then `cd aws-amazon-polly-s3`
4. Run `conda create -p venv python==3.10`
- Explanation: -p venv: to create the environment in the venv directory; relative to my current working directory
5. Activate this environment: Run `conda activate venv/` OR `conda activate /Users/mijikim/aws-amazon-polly-s3/venv`
6. To deactivate an active environment, use `conda deactivate`
7. Create a Requirements text file with the required libraries
8. File > New Text File > enter boto3
  - This is AWS SDK for Python. This is used to interact with AWS services programmatically.
  - Note: `awscli` is not included as I won't use AWS CLI commands inside my Python project and I'm just using the AWS CLI to configure credentials (aws configure) once locally.
  - `awscli` is AWS Command Line Interface, which is used to manage AWS services from the command line.
9. Save as required_libraries.txt
10. Install required libraries: run `pip install -r required_libraries.txt`
  - Explanation: -r is --requirement. It specifies that the argument following this is a requirements file containing a list of dependencies to install.
