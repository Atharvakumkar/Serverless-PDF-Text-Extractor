# Event-Driven Serverless PDF Text Extraction Pipeline

An event-driven, serverless PDF processing pipeline built on AWS that automatically extracts text from uploaded PDF documents using Python and PyMuPDF.

The system uses Amazon S3 event notifications to trigger AWS Lambda, processes PDF documents page by page, and stores the extracted text in a separate S3 bucket. The architecture eliminates the need for persistent compute infrastructure and demonstrates event-driven cloud automation.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Execution Flow](#execution-flow)
- [Features](#features)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Configuration](#configuration)
- [Lambda Function](#lambda-function)
- [Lambda Layer](#lambda-layer)
- [Security Considerations](#security-considerations)
- [Observability](#observability)
- [Implementation Notes](#implementation-notes)
- [Future Improvements](#future-improvements)
- [Author](#author)
- [License](#license)

---

## Overview

This project demonstrates how to build an automated document-processing workflow using AWS serverless services.

Whenever a PDF is uploaded to the input S3 bucket, an event notification invokes a Lambda function. The function downloads the document into memory, extracts text from each page using PyMuPDF, and writes the resulting text file to a separate S3 bucket.

The architecture combines serverless compute, event-driven execution, object storage, IAM permissions, dependency packaging, and centralized logging.

### Key Objectives

- Automate PDF text extraction using event-driven execution.
- Eliminate the need for continuously running servers.
- Integrate AWS Lambda with Amazon S3.
- Package native Python dependencies using Lambda Layers.
- Apply least-privilege IAM permissions.
- Monitor execution and failures using Amazon CloudWatch.

## Architecture

The pipeline uses the following AWS services and components.

| Component | Responsibility |
|---|---|
| Amazon S3 — Input Bucket | Receives PDF documents under the `uploads/` prefix |
| Amazon S3 Event Notifications | Triggers processing when matching objects are created |
| AWS Lambda | Executes the PDF extraction logic on demand |
| PyMuPDF | Extracts text from PDF documents |
| AWS Lambda Layers | Packages PyMuPDF and its required dependencies |
| AWS IAM | Controls access to input and output resources |
| Amazon S3 — Output Bucket | Stores extracted text files under the `extracted/` prefix |
| Amazon CloudWatch | Collects execution logs, metrics, and diagnostic information |

The input and output buckets are separate to prevent generated text files from triggering the PDF-processing workflow again.

### Architecture Diagram

```text
                 PDF Upload
                     |
                     v
          +----------------------+
          |   Amazon S3          |
          |   Input Bucket       |
          |   uploads/*.pdf      |
          +----------------------+
                     |
                     | ObjectCreated Event
                     v
          +----------------------+
          |   AWS Lambda         |
          |   Python Runtime     |
          +----------------------+
                     |
                     v
          +----------------------+
          |   PyMuPDF            |
          |   Text Extraction    |
          +----------------------+
                     |
                     v
          +----------------------+
          |   Amazon S3          |
          |   Output Bucket      |
          |   extracted/*.txt    |
          +----------------------+

                     |
                     v
          +----------------------+
          |   Amazon CloudWatch  |
          |   Logs and Metrics   |
          +----------------------+
```

**Supporting components:** AWS IAM provides permissions for the Lambda execution role, while the Lambda Layer supplies the PyMuPDF dependency.

## Execution Flow

The processing workflow follows these steps:

1. A user uploads a PDF to the input S3 bucket under `uploads/`.
2. An S3 event notification detects the matching object creation.
3. AWS Lambda is invoked with the S3 event payload.
4. The function extracts the bucket name and object key.
5. The PDF is retrieved from S3 and loaded into memory.
6. PyMuPDF extracts text from each page.
7. The function generates an output key under `extracted/`.
8. The extracted text is uploaded to the output S3 bucket.
9. Execution details and errors are recorded in CloudWatch Logs.

### Example

**Input object:**

```text
s3://your-input-bucket/uploads/sample.pdf
```

**Output object:**

```text
s3://your-output-bucket/extracted/sample.txt
```

The output contains the text extracted from the PDF document.

## Features

- **Event-Driven Processing:** Automatically initiates processing when a matching PDF is uploaded.
- **Serverless Execution:** Uses AWS Lambda without requiring persistent application servers.
- **Automated Text Extraction:** Extracts text page by page using PyMuPDF.
- **S3 Integration:** Reads input documents and writes extracted text to Amazon S3.
- **Lambda Layer Packaging:** Separates third-party dependencies from application code.
- **Configurable Deployment:** Uses environment variables for deployment-specific settings.
- **IAM-Based Access Control:** Restricts access through an execution role.
- **CloudWatch Observability:** Supports centralized logging and execution monitoring.
- **Filename Handling:** URL-decodes S3 object keys to support spaces and special characters.
- **Separate Storage Boundaries:** Keeps input documents and generated outputs in different buckets.

## Technology Stack

### Cloud Infrastructure

- AWS Lambda
- Amazon S3
- AWS IAM
- Amazon CloudWatch
- AWS Lambda Layers

### Programming and Libraries

- Python 3.11
- PyMuPDF
- boto3
- `urllib.parse`

### Deployment and Packaging

- AWS CLI
- Docker for dependency packaging
- ZIP-based Lambda deployment packages

## Project Structure

```text
pdf-text-extractor/
|
|-- lambda_function.py
|   Core Lambda handler for retrieving PDFs,
|   extracting text, and writing output to S3.
|
|-- function.zip
|   Deployment package containing the Lambda function.
|
|-- pymupdf-layer.zip
|   Prebuilt Lambda Layer containing PyMuPDF dependencies.
|
|-- layer-build/
|   Scripts and instructions for building the Lambda Layer.
|
|-- layer-final/
|   Final packaged Python dependencies.
|
|-- layer-packages/
|   Intermediate dependency packaging artifacts.
|
|-- README.md
|   Project documentation.
```

The exact contents of the directories depend on the repository's current implementation.

## Prerequisites

Before deploying the pipeline, ensure that you have:

- An AWS account.
- AWS CLI installed and configured.
- Python 3.11 available for local development.
- Permissions to create S3 buckets, Lambda functions, IAM roles, and Lambda Layers.
- Permissions to configure S3 event notifications and Lambda invocation permissions.
- Docker, if rebuilding the native Python dependency package.

Configure your AWS CLI credentials:

```bash
aws configure
```

Verify the active AWS identity:

```bash
aws sts get-caller-identity
```

Use an AWS account and region where you have the required permissions.

## Deployment

The following commands illustrate the deployment workflow. Replace the example resource names, account ID, and ARNs with your actual values.

### 1. Create the S3 Buckets

Create separate buckets for incoming PDFs and extracted text.

```bash
aws s3 mb s3://your-input-bucket-name
aws s3 mb s3://your-output-bucket-name
```

S3 bucket names must be globally unique.

The Lambda function will process PDFs uploaded under the `uploads/` prefix and write results under the `extracted/` prefix in the output bucket.

### 2. Create the Lambda Execution Role

Create an IAM role that AWS Lambda can assume.

The role should trust the Lambda service:

```json
{
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
}
```

Attach the AWS-managed `AWSLambdaBasicExecutionRole` policy for standard CloudWatch logging permissions.

Add a custom permissions policy that grants access only to the required S3 resources.

Example policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadInputPDFs",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::your-input-bucket-name/uploads/*"
    },
    {
      "Sid": "WriteExtractedText",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::your-output-bucket-name/extracted/*"
    }
  ]
}
```

This policy restricts object access to the specified prefixes.

### 3. Publish the Lambda Layer

The project includes a prebuilt Lambda Layer containing PyMuPDF and its dependencies.

Publish the layer using:

```bash
aws lambda publish-layer-version \
  --layer-name pymupdf-layer \
  --zip-file fileb://pymupdf-layer.zip \
  --compatible-runtimes python3.11
```

Record the `LayerVersionArn` returned by AWS.

**Important:** Native dependencies must be compatible with the Lambda runtime, operating system, and architecture. Verify that the supplied layer works with the target execution environment before deploying it.

### 4. Deploy the Lambda Function

Upload the function package and attach the Lambda Layer.

```bash
aws lambda create-function \
  --function-name pdf-text-extractor \
  --runtime python3.11 \
  --role arn:aws:iam::<account-id>:role/<your-role-name> \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --layers <LayerVersionArn> \
  --environment 'Variables={OUTPUT_BUCKET=your-output-bucket-name}' \
  --timeout 60 \
  --memory-size 256
```

Replace the placeholders with your actual account ID, role name, layer ARN, and output bucket.

The memory allocation and timeout are starting values. Adjust them based on the size of the PDFs you expect to process.

### 5. Grant S3 Permission to Invoke Lambda

Allow the input bucket to invoke the function.

```bash
aws lambda add-permission \
  --function-name pdf-text-extractor \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::your-input-bucket-name
```

If you deploy resources across different AWS accounts, review the additional permissions and configuration required for that arrangement.

### 6. Configure the S3 Event Notification

Configure the input bucket to invoke the Lambda function when matching PDFs are created.

Recommended notification settings:

| Setting | Value |
|---|---|
| Event type | `s3:ObjectCreated:Put` |
| Prefix filter | `uploads/` |
| Suffix filter | `.pdf` |
| Destination | `pdf-text-extractor` Lambda function |

You can configure this through the Amazon S3 console or through an S3 bucket notification configuration.

Ensure the notification points to the correct Lambda function and that the required invocation permission has been added.

### 7. Test the Pipeline

Upload a sample PDF:

```bash
aws s3 cp sample.pdf s3://your-input-bucket-name/uploads/sample.pdf
```

Check the output bucket:

```bash
aws s3 ls s3://your-output-bucket-name/extracted/
```

If processing succeeds, the corresponding text file should appear in the output location.

Download and inspect the extracted text:

```bash
aws s3 cp \
  s3://your-output-bucket-name/extracted/sample.txt \
  ./sample.txt
```

Open `sample.txt` to verify the extracted content.

## Configuration

The Lambda function uses environment variables for deployment-specific configuration.

| Variable | Description | Required |
|---|---|---|
| `OUTPUT_BUCKET` | Destination bucket for extracted text files | Yes |

Configure the variable during deployment or update it through the AWS console.

Keep environment-specific values outside the application code. Use IAM roles for AWS access rather than embedding access keys in the function.

## Lambda Function

The main application logic resides in `lambda_function.py`.

For each invocation, the handler performs the following operations:

1. Parses the S3 event payload.
2. Retrieves the source bucket and object key.
3. URL-decodes the object key to handle encoded characters.
4. Validates that the object has a `.pdf` extension.
5. Retrieves the PDF using `s3.get_object()`.
6. Loads the document from its in-memory bytes using `fitz.open()`.
7. Iterates over the pages and extracts text using `page.get_text()`.
8. Constructs the output key under `extracted/`.
9. Writes the extracted content to S3 using `s3.put_object()`.
10. Returns a response containing the output object key.

The function uses `boto3` for S3 interactions and PyMuPDF for document processing.

### Example Transformation

```text
Input:
uploads/invoice.pdf

Processing:
Download PDF -> Extract Page Text -> Combine Text

Output:
extracted/invoice.txt
```

The output path and transformation behavior depend on the implementation in `lambda_function.py`.

## Lambda Layer

PyMuPDF includes native extensions, so its dependencies must be packaged for a compatible Lambda execution environment.

The project includes `pymupdf-layer.zip`, which contains the dependency package used by the Lambda function.

### Why Use a Lambda Layer?

- Separates third-party dependencies from application code.
- Allows the same dependency package to be reused across functions.
- Keeps the function deployment package separate from the library package.
- Simplifies dependency updates and packaging.

### Rebuilding the Layer

The `layer-build/` directory contains the scripts and instructions used to build the layer.

The general process is:

1. Select a build environment compatible with the target Lambda runtime.
2. Install the required PyMuPDF version and dependencies.
3. Place the packages in the correct Lambda Layer directory structure.
4. Package the resulting `python/` directory into a ZIP file.
5. Publish the layer and attach it to the function.
6. Test the function in the target runtime.

Verify runtime, architecture, and native-library compatibility when rebuilding the layer.

## Security Considerations

The deployment incorporates several security practices.

- **Least-Privilege IAM:** Restricts S3 access to the required input and output object prefixes.
- **Separate Buckets:** Isolates incoming documents from generated output files.
- **Environment-Based Configuration:** Avoids hardcoding deployment-specific bucket names.
- **Managed AWS Credentials:** Uses the Lambda execution role for AWS API access.
- **CloudWatch Logging:** Provides visibility into execution failures and processing activity.

### Additional Production Considerations

For production deployments, consider:

- Enabling encryption at rest for both S3 buckets.
- Using TLS for data transfer.
- Enforcing S3 Block Public Access.
- Validating file size and document structure before processing.
- Limiting execution time and memory usage.
- Handling malformed or encrypted PDFs gracefully.
- Configuring alerts for repeated Lambda failures.
- Reviewing log content to avoid exposing sensitive document data.
- Using a dead-letter queue or a Lambda destination where appropriate for failure handling.

These are recommended enhancements and should not be interpreted as implemented features unless they have been configured and tested.

## Observability

Amazon CloudWatch provides logs and execution metrics for the Lambda function.

The default Lambda log group is:

```text
/aws/lambda/pdf-text-extractor
```

Depending on the function's logging implementation, logs may include:

- Source bucket and object key.
- Number of pages processed.
- Number of characters extracted.
- Destination object key.
- Processing errors and skipped files.

AWS Lambda also publishes metrics such as:

- Invocations.
- Duration.
- Errors.
- Throttles.

Review these metrics to understand execution behavior and identify failures or performance issues.

CloudWatch metrics are available automatically for Lambda, while application-specific details depend on the logging statements implemented in the function.

## Implementation Notes

### Separate Input and Output Buckets

Using separate buckets prevents generated `.txt` files from matching the input bucket's PDF notification rules.

It also creates a clear separation between source documents and processed results.

### Event Filtering

The `uploads/` prefix and `.pdf` suffix filters restrict notifications to matching objects.

The exact notification event types configured determine which S3 operations trigger processing.

### Environment Variables

The output bucket name is configured through `OUTPUT_BUCKET`, making the function easier to reuse across environments.

### In-Memory Processing

The PDF is read into memory rather than written to Lambda's temporary filesystem.

This simplifies the processing workflow but means memory consumption increases with document size. Large documents may require more memory or a different streaming or temporary-storage strategy.

### Error Handling

S3 events and Lambda invocations may be delivered or executed more than once.

For production use, consider idempotent processing, explicit exception handling, retry behavior, and protection against overwriting valid outputs unexpectedly.

### PDF Limitations

Text extraction depends on the contents of the source document.

Scanned PDFs containing only images generally require OCR before meaningful text can be extracted. Password-protected, corrupted, or unusually large documents may also require additional handling.

## Future Improvements

Potential enhancements include:

- Adding OCR support for scanned PDFs.
- Introducing structured JSON output alongside plain text.
- Supporting configurable page ranges.
- Adding file-size validation and document-type checks.
- Implementing more comprehensive retry and failure handling.
- Adding automated tests for the extraction logic.
- Managing the infrastructure through Terraform or AWS CloudFormation.
- Creating CloudWatch alarms and dashboards.
- Adding idempotency controls for repeated event deliveries.
- Supporting larger documents through an alternative processing architecture.

These are possible future enhancements rather than claims about the current implementation.

## Author

**Atharva Kumkar**

GitHub: [Atharvakumkar](https://github.com/Atharvakumkar)

## License

This project is licensed under the MIT License.

See the [`LICENSE`](LICENSE) file for the complete license text.
