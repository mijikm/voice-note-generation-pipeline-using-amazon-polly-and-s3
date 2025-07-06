# voice-note-generation-pipeline-using-amazon-polly-and-s3

### Project 
AI-Powered Voice Note Generator & Uploader with Amazon Q Support

### Project Overview
1. Generate a meeting script using Amazon Bedrock assisted by Amazon Q (Claude Sonnet 3)
2. Convert it to speech using Amazon Polly
3. Upload the audio to S3 using Boto3
(Optional: Transcribe it back using Amazon Transcribe)


### The Complete Flow:
1. Run `python test_lambda.py`
2. This triggers a Lambda function: `meeting-transcript-processor-v2`
3. This Main Lambda function processes the following:
   1) **Bedrock**: Bedrock generates a transcript
   2) **Polly**: Polly converts the transcript to MP3 audio file
   3) **S3**: The audio file (.mp3) is loaded to S3
   4) **DynamoDB**: The meeting data is saved to a DynamoDB table
      - Meta data: `meeting_id`, `topic`, `transcript`, `summary`, `audio_file__url`, `timestamp`
4. DynamoDB streams automatically trigger a Lambda function: `dynamodb-stream-sync`
5. This Stream Lambda function does the following:
   1) It reads DybamoDB change
   2) It converts DynamoDB format to a regular dictionary data
   3) It creates a text document and uploads it to S3 for Q Business
6. Amazon Q Business does the following:
   1) It scans S3, locates the text file, indexes the file (organising doc into searchable index (e.g. topics, keywords, content chunks))
   2) It makes content searchable
   3) It answers user questions
8. Users query data with Q Business
   - e.g. User: What was discussed in recent meetings?"
   
*Visual FLow: Manual Trigger → Main lambda (meeting-transcript-processor-v2) → (DynamoDB Stream (automatic)) → Stream Lambda (dynamodb-stream-sync) → (Automatic Indexing) → Amazon Q Business

What Gets Created Per Meeting:
1. DynamoDB Record: Structured data
2. S3 Audio File: `meeting_audio/{uuid}.mp3`
3. S3 Text Document: `meeting_documents/{uuid}.txt` (for Q Business)
4. Q Business Index: searchable meeting content

### Prerequisites
1. Install AWS Toolkit in VS Code
  - Go to Extensions → Search: AWS Toolkit → Install
  - Connect to your AWS account (ensure permissions for Polly + S3)
    - *Note: the default region may need to be explicitly set in the config file. Run `nano ~/.aws/config` -> add profile
    - ```
      [profile PollyS3Users]
      region = us-east-1
      output = json
      ```
2. Install the Amazon Q Extension in VS Code
  - In the AWS Toolkit panel, click on `Install the Amazon Q Extension`. ensure you're signed in and Amazon Q is active
  - Sign in with your Builder ID
  - You’ll see a sidebar panel where you can ask questions
3. Install Python and Boto3

### Prerequisites on AWS
1. Create a User Group with Permissions
  - Go to IAM Console: https://console.aws.amazon.com/iam
  - Create a new user group: `PollyS3Users`
  - Attach these managed policies to the User Group:
    - AmazonBedrockFullAccess - for Amazon Bedrock
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

3. Create Amazon S3 bucket
  - Bucket name: `s3-meeting-data-polly`

4. Enable Bedrock Model Access
  - Amazon Bedrock Console → Model access (left sidebar) → Click "Enable specific models" or "Manage model access" → Enable Anthropic `Claude 3 Sonnet` → Submit the request
### Set up
#### Visual Studio Code
1. Create a virtual environment
2. View > New Terminal > Switch to Command Prompt
3. Navigate to the folder where a virtual environment will be created
  - You may also create a folder: `mkdir aws-amazon-polly-s3` then `cd aws-amazon-polly-s3`
4. Run `conda create -p venv python==3.10`
- Explanation: -p venv: to create the environment in the venv directory; relative to my current working directory
5. To activate this environment: Run `conda activate venv/` OR `conda activate /Users/mijikim/aws-amazon-polly-s3/venv`
6. To deactivate an active environment, use `conda deactivate`
7. Create a Requirements text file with the required libraries
8. File > New Text File > enter boto3
  - This is AWS SDK for Python. This is used to interact with AWS services programmatically.
  - Note: `awscli` is not included as I won't use AWS CLI commands inside my Python project and I'm just using the AWS CLI to configure credentials (aws configure) once locally.
  - `awscli` is AWS Command Line Interface, which is used to manage AWS services from the command line.
9. Save as required_libraries.txt
10. Install required libraries: run `python -m pip install boto3`
  - What it does: Runs the pip module using the current Python interpreter (python). Installs the boto3 package into the site-packages directory of that specific Python environment.
  - Why use python -m pip instead of just pip? It ensures that you use the pip associated with the exact python interpreter you want.
  - This avoids problems where the pip command might point to a different Python environment (e.g., system Python vs virtualenv).
  - Especially important if you have multiple Python versions or virtual environments.
  - In summary: This command installs the AWS SDK for Python (boto3) into the Python environment that runs when you call python. Guarantees the package is installed where your Python script will run.

11. Install AWS CLI (on macOS)
  - (base) run `git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core fetch --unshallow`
    - This converts a shallow clone into a full clone by downloading the entire Git history
  - (base) run `brew install awscli`
    -  Installs the AWS Command Line Interface (CLI) using Homebrew, the package manager for macOS (and Linux).
    -  When running this prompt,
      -  Homebrew downloads the AWS CLI package and its dependencies from the internet.
      -  It builds or installs the binaries (usually precompiled).
      -  It places the aws command into a directory like /usr/local/bin or /opt/homebrew/bin, which should be in your PATH.
      -  You can then use aws from the terminal to interact with AWS services.
  - test installation by running `aws --version`
13. Configure Locally (VS Code / AWS CLI)
  - In terminal in VS Code, activate venv environment: `conda activate /Users/mijikim/aws-amazon-polly-s3/venv`
  - (venv) run `aws configure`
  - It will ask for:
  - AWS Access Key ID: enter the copied access key
  - AWS Secret Access Key: copy it from AWS and paste it here
  - Default region name: enter us-east-1
  - Default output format: enter json

14. Test connection
  - Run `python polly_s3_script.py`
    ```
    import boto3
    import botocore.exceptions
    
    # Creates a client for Amazon Polly
    polly = boto3.client("polly")
    # Test the connection by synthesizing speech
    ''' Calls synthesize_speech():
    Text: "Test" — the text to convert to speech.
    OutputFormat: "mp3" — the audio file format.
    VoiceId: "Joanna" — one of the standard English (US) voices.
    '''
    try:
        response = polly.synthesize_speech(Text="Test", OutputFormat="mp3", VoiceId="Joanna")
        print("Polly access successful.")
    except botocore.exceptions.ClientError as e:
        print("Polly access failed:", e.response["Error"]["Message"])
    
    # Creates a client for Amazon S3
    s3 = boto3.client("s3")
    # Uploads a text file (test.txt) with content "Hello" to the bucket your-bucket-name. Prints a message if the upload succeeds.
    try:
        s3.put_object(Bucket="s3-meeting-data-polly", Key="test.txt", Body="Hello")
        print("S3 access successful.")
    except botocore.exceptions.ClientError as e:
        print("S3 access failed:", e.response["Error"]["Message"])
    ```

### Implementation
1. Run code to generate meeting script with Amazon Bedrock, convert it to speech with Polly and upload the generated mp3 file to S3
  - `python bedrock_polly_meeting_transcript.py` 
3. Verify the outcome
  - `aws s3 ls --profile PollyS3Users`: To list all S3 buckets
  - `aws s3 ls s3://s3-meeting-data-polly/meeting_audio/ --profile PollyS3Users`: To list everything in the specified S3 bucket folder

# Enhancement 1 - Trigger with Lambda
1. Generate scripts using Amazon Q to use Lambda
  - `lambda_function.py`: this is your main Lambda function that processes meeting transcripts. Here's what it does:
    - **Core Functions**:
      - generate_meeting_transcript() - Returns a hardcoded meeting transcript (we removed the Bedrock call to avoid permission issues)
      - text_to_speech() - Takes the transcript text and uses Amazon Polly to convert it into an MP3 audio file
      - upload_to_s3_direct() - Uploads the audio data directly to S3 without saving to local files (since Lambda has limited local storage)
    - **Main Handler**:
      - lambda_handler() - This is the entry point that AWS Lambda calls when triggered
      - It accepts parameters like bucket name, output file path, and voice ID
      - Runs the 3-step process: generate transcript → convert to speech → upload to S3
      - Returns JSON responses with success/error status codes
    - Key Lambda Adaptations:
      - No local file operations (everything stays in memory)
      - Proper error handling with HTTP status codes
      - Configurable via event parameters
  - `deploy_lambda_simple.py`: This is your deployment automation script. Here's what it does:
    - **Two Main Functions**:
      - create_lambda_package() - Zips up your lambda_function.py file into a deployment package that AWS can use
      - deploy_lambda_function() - Uploads the package to AWS and either creates a new Lambda function or updates an existing one
2. Add Lambda permissions to your user
  - Via AWS Console: Go to IAM → Users → polly-s3-user
  - Click "Add permissions" → "Attach policies directly"
  - Search for "AWSLambda_FullAccess"
  - Attach it
3. Create the IAM role manually via AWS Console
  - Go to IAM Console → Roles → Create role
    - *Note: This allow an AWS service like EC2 or **Lambda**, or others to perform actions in this account.
  - Select AWS service → Lambda
  - Attach these policies: `AWSLambdaBasicExecutionRole`, `AmazonPollyFullAccess`, `AmazonS3FullAccess`, `AmazonBedrockFullAccess`,
  - Name it `lambda-execution-role`
4. If you run `python deploy_lambda_simple.py`, it will create a Lambda function `meeting-transcript-processor-v2` under your user.
5. Add an API Gateway Trigger
  - Go to AWS Lambda Console → Select your Lambda function.
  - On the left panel, click “Configuration”, not Code.
  - Under “Triggers”, click “Add trigger”.
  - Select “API Gateway” from the list.
  - For API type, select HTTP API (Recommended) or REST API if you need advanced features.
  - Select “Create a new API”, or attach to an existing one.
  - Leave security as Open for testing (you can add authentication later).
  - Click “Add”. After a few seconds, you’ll see an API endpoint URL in the trigger list.
6. Test via Postman (https://web.postman.co/)
  - Open Postman.
  - Set method to POST.
  - Paste your API Gateway URL in the address bar.
7. (Optional) Amazon EventBridge Scheduler - to create schedule to invoke lambda function

# Enhancement 2 - Store Data in Dynamo DB
What is Dynamo DB? Amazon DynamoDB is a fully managed NoSQL database service from AWS.
  - Serverless: No need to manage servers, storage, or scaling.
  - Fast: Single-digit millisecond response times.
  - Scalable: Automatically handles workloads from small projects to enterprise-scale systems.
  - Key-value or document store: You store items like JSON objects.

Why not S3? S3 is good for static file storage. it is hard to query for individual meetings and there is no structured metadata.
Why not RDS (Relational DB)? it's usually good for complex queries, joins. It is an overkill for simple meeting data, needs server management.

Why use Dynamo DB? it's good for simple key-value storage. IT is perfect for JSON-like meeting data, serverless, and quick set up. T store your meeting scripts, summaries, and audio URLs permanently.	Q can search DynamoDB as a data source (or you can build APIs on top).DynamoDB is like a JSON storage where each row is a meeting, and you can easily look it up later — but it’s built for fast, scalable cloud applications.
 
Example:
| meeting\_id | topic     | transcript       | summary                              | audio\_file\_url               | timestamp         |
| ----------- | --------- | ---------------- | ------------------------------------ | ------------------------------ | ----------------- |
| abc123      | Team Sync | "Hello, team..." | "Discussed sales. Action: Bob to..." | s3://bucket/meeting\_audio.mp3 | 2025-07-05T10:00Z |

✅ Step 1: Go to DynamoDB Console
1. Log in to the AWS Console.
2. Search for and open “DynamoDB”.
3. Click “Create table”.

✅ Step 2: Configure the Table
| Field             | What to Enter                    |
| ----------------- | -------------------------------- |
| **Table name**    | `MeetingRecords`                 |
| **Partition key** | `meeting_id`                     |
| Type              | `String`                         |
| Sort key          | Leave empty (not needed for now) |

Leave everything else as default:
  - Table class: Standard
  - Capacity mode: On-demand
  - Encryption: Default AWS owned key

Click “Create table”.

✅ Step 3: Wait for the Table to Be Active
It will take a few seconds. You'll see the status as "Active" when ready.

✅ Step 4: (Optional for Now) Explore the Table
Go to “Explore table items”.

You'll see an empty table — you’ll insert data from Lambda next.

✅ Step 5: (Prepare for Lambda) — Table Summary
Table Name	Partition Key	Type
MeetingRecords	meeting_id	String

✅ Step 6: Modify lambda_function.py to save data into DynamoDB
With the help of Amazon Q, lambda_function.py was modified to create dynamodb service client,  assign a topic, generate unique meeting ID, timestamp, and summary

✅ Step 7: Add DynamoDB permissions to your Lambda role first, then deploy
Go to IAM Console → Roles → `lambda-execution-role`
Add Permission → Attach this policy: AmazonDynamoDBFullAccess

✅ Step 8: Run the following to deploy (uploads the code to AWS)
`python deploy_lambda_simple.py`

✅ Step 9: Run the following to trigger the Lambda
`python test_lambda.py`

✅ Step 10: Verify data is stored in DynamoDB table 
DynamoDB → Explore items → MeetingRecords

# Enhancement 3 - Build a conversational app with Amazon Q Apps
✅ STEP 1: Create an Organization to enable IAM Identity Center (prerequisite for creating Q App)
1. Go to the AWS Console → search for “Organizations”
2. If prompted, click “Create an organization”.
3. Choose “Create organization” → “Enable all features”.
4. Wait 1–2 minutes for setup to complete.
✔️ If your account is already part of an Organization, you’ll see it listed.

✅ STEP 2: Enable IAM Identity Center
1. Open the IAM Identity Center Console
- In AWS Console, search for “IAM Identity Center”.
- Click on it.
- If you see “Enable IAM Identity Center”, Click `Enable` button:
  - It will show current AWS region. e.g. `Current AWS Region: US East (N. Virginia)`
- Leave the defaults for the Identity source as “Identity Center directory” (no external provider needed now).
- Click “Enable”.
- Wait ~30 seconds for it to initialize.
- Once enabled, you’ll land on the IAM Identity Center dashboard, where you’ll see:
“Users”
“Groups”
“Applications”
2. Create User
- Username: meeting.analyst
- Password: Send an email to this user with password setup instructions.
- Email address
- Confirm email address
- First name: Meeting
- Last name: Analyst
- Display name: Meeting Analyst
- (Note*: Set up the authenticator app when prompted)
  
3. Create Group
- Name: MeetingAnalysisTeam
- Description: Users who can access meeting transcripts and analysis via Q Business.
- Add `meeting.analyst` to the group
- Create group
✔️ You don’t need to create users/groups now for testing Q Apps with “No access control”.
  
✅ STEP 3: Create an Amazon Q App
1. Go to Amazon Q Apps Console → Amazon Q Business → Applications → Create application
  - Application name: `QBusinessMeetingSummaryApp`
  - User access: `Authenticated Access`
  - Outcome: `Web Experience`
  - Access management method: `AWS IAM Identity Center`
    - You will see `Application connected to IAM Identity Center`
  - Click `Create`

🔍 What Did You Just Set Up?
| What                  | Why                                               |
| --------------------- | ------------------------------------------------- |
| AWS Organization      | Required for centralized identity management      |
| IAM Identity Center   | Required for all Amazon Q Business apps           |
| Basic identity source | Even for apps without fine-grained access control |

✅ STEP 4: Set up an Amazon Q App - create an index
1. Amazon Q Business → Applications → QBusinessMeetingSummaryApp → Data sources → Create an index
- Index name: `meeting-index`
- Index provisioning: `Starter` (*Note: The Starter Index is deployed in a single Availability Zone, making it ideal for PoCs and developer workloads.)
- Number of units: `1`
- Click `Add an index`
2. Once an index has been created, Add data source
- DynamoDB isn't directly available as a data source in Q Business, but S3 is.
- DynamoDB Streams will be implemented to
  - Keep DynamoDB for structured queries
  - Auto-sync to S3 for Q Business indexing
  - No manual redundancy management

✅ STEP 5: Set up an Amazon Q App - sync DynamoDB to S3
1. Enable DynamoDB Streams: run `python enable_dynamodb_streams.py` (this is a one-time setup)
  - *Note: Before running this, you need to add DynamoDB Admin Permissions (`AmazonDynamoDBFullAccess`) to user group `PollyS3Users`)
  - *Note: To disable via AWS Console
    - Disable Event Source Mapping:
    - 1. Lambda Console → Functions → dynamodb-stream-sync
    - 2. Configuration → Triggers
    - 3. Delete the DynamoDB trigger
    - Disable Streams:
    - 1. DynamoDB Console → Tables → MeetingRecords
    - 2. Exports and streams → DynamoDB stream details
    - 3. Disable stream

2. Deploy Stream Handler: run `python deploy_stream_lambda.py` (this will create a new Lambda function - `dynamodb-stream-sync`
  - Triggered by DynamoDB streams, Syncs data from DynamoDB to S3, creates documents for Q Business
  - *Note: You need to enter the DynamoDB Stream ARN which can be found: DynamoDB Tables → MeetingRecords → Exports and streams tab → Under DynamoDB stream details, you'll see `Latest Stream ARN`
    
3. Deploy Updated Main Lambda: run `python deploy_lambda_simple.py`
  - as the code inside `lambda_function.py` changed (it used to save document to S3, but it is now removed)
   
4. Trigger the lambda: run `python test_lambda.py`
  - You can now see a new text file created in `meeting_documents/` S3 folder in `s3-meeting-data-polly` bucket.

✅ STEP 6: Set up an Amazon Q App - add S3 as data source
1. Amazon Q Business → Applications → QBusinessMeetingSummaryApp → Data sources → Add data source → S3
2. Data source name: `meeting-documents`
3. Enter the data source location: `s3://s3-meeting-data-polly`
4. Filter patterns - optional: Include prefix pattern: `meeting_documents/`
5. File types: Select Text files (.txt)
6. IAM role: `Create a new service role (Recommended), Role name: `QBusiness-DataSource-ekucs`
  - *Note: Q Business will need permissions to read your S3 bucket. Q Business will automatically:
  - Create the IAM role with correct permissions
    - Set up trust relationship with qbusiness.amazonaws.com
    - Grant S3 read access to your bucket
    - Handle all the IAM complexity for you
7. Sync mode: select `Full sync` initially
8. Sync run schedule - Frequency: `Run on demand`
9. Click Add data source
✔️ Once added, click on  `Sync now`

✅ STEP 7: Assign groups for Q Business App
- Amazon Q Business → Applications → QBusinessMeetingSummaryApp → Manage user access → Add groups and user → Add `MeetingAnalaysisTeam` group as Q Business Lite.

✅ STEP 8: Test Q Business App
- Amazon Q Business → Applications → QBusinessMeetingSummaryApp → Web experience settings → Click on `Deployed URL` -> Sign in with `meeting.analyst`
