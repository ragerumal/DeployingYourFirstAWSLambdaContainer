# Deploying Your First Lambda Container

This guide walks you through the steps to create, test, build, and deploy a Lambda function using a container image. You'll work with AWS SAM, Docker, and other tools to deploy a Lambda function that generates a PDF file.

## Prerequisites

Before you begin, ensure that the following tools are installed and available in your Cloud9 environment:

- AWS CLI
- AWS SAM CLI
- Docker
- Node.js
- NPM

You can verify the installation of these tools by running the following command in the Cloud9 terminal:

```bash
aws --version && sam --version && docker --version && node --version && npm --version
```
![image](https://github.com/user-attachments/assets/d7245cfa-2e2b-4c51-bf7a-47f5acd32300)

Steps to Create and Deploy the Lambda Function
1. Initialize a New SAM Application
Run the following command to initialize a new SAM application:
```bash
sam init
```
When prompted, choose the following options:

Template source: AWS Quick Start Templates
Package type: Hello World Example
Runtime: nodejs18.x
Package type: Image
X-Ray Tracing: No
CloudWatch Application Insights: No
Structured Logging: No
Project name: getletter
![image](https://github.com/user-attachments/assets/d36dae5e-1888-446d-a22f-a930153023ec)

2. Update the SAM Template
Replace the contents of template.yaml with the following:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  getletter

  SAM Template for our letter writer
Globals:
  Function:
    Timeout: 5

Resources:
  GetLetter:
    Type: AWS::Serverless::Function
    Properties:
      PackageType: Image
      Role: !Sub 'arn:aws:iam::${AWS::AccountId}:role/lambda-execution-role'
    Metadata:
      DockerTag: nodejs18.x-v1
      DockerContext: ./hello-world
      Dockerfile: Dockerfile

Outputs:
  GetLetter:
    Description: "Letter Writer Lambda Function ARN"
    Value: !GetAtt GetLetter.Arn
```

3. Update the Lambda Function Code
Replace the contents of hello-world/app.mjs with the following code:
```js
import PDFDocument from 'pdfkit';
import faker from 'faker';

export const lambdaHandler = async (event, context) => {
    const pdfBuffer = await new Promise(resolve => {
        const doc = new PDFDocument();
        const randomName = faker.name.findName();

        doc.text(randomName, { align: 'right' });
        doc.text(faker.address.streetAddress(), { align: 'right' });
        doc.text(faker.address.secondaryAddress(), { align: 'right' });
        doc.text(faker.address.zipCode() + ' ' + faker.address.city(), { align: 'right' });
        doc.moveDown();
        doc.text('Dear ' + randomName + ',', { align: 'left' });
        doc.moveDown();
        for(let i = 0; i < 3; i++) {
            doc.text(faker.lorem.paragraph());
            doc.moveDown();
        }
        doc.text(faker.name.findName(), { align: 'right' });
        doc.end();

        const buffers = [];
        doc.on("data", buffers.push.bind(buffers));
        doc.on("end", () => resolve(Buffer.concat(buffers)));
    });

    return {
        statusCode: 200,
        headers: {
            'Content-Length': Buffer.byteLength(pdfBuffer),
            'Content-Type': 'application/pdf',
            'Content-disposition': 'attachment;filename=test.pdf'
        },
        isBase64Encoded: true,
        body: pdfBuffer.toString('base64')
    };
};
```
4. Install Required Packages
Navigate to the hello-world folder and install the required packages:
```bash
cd getletter/hello-world/
npm init -y && npm install pdfkit faker
npm audit fix --force

```

5. Build the Docker Image
Review the Dockerfile and build the Docker image:
```bash
docker build -t getletter .
docker images
```

6. Test the Lambda Function Locally
Run the Docker container locally to test the function:
```bash
docker run -p 9000:8080 getletter:latest

```
![image](https://github.com/user-attachments/assets/a94c7fe2-6b2f-4705-a2ca-ffbaa3e950a7)

In another terminal, test the function:
```bash
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
```
![image](https://github.com/user-attachments/assets/2b429803-b5e8-4962-b040-742275df68ee)

7. Build and Deploy the Function with SAM
Build and deploy the function using SAM:
```bash
aws ecr create-repository --repository-name getletter --image-scanning-configuration scanOnPush=true
docker tag getletter:latest <repositoryUri>
aws ecr get-login-password | docker login --username AWS --password-stdin <repositoryUri>
docker push <repositoryUri>:latest
```
Use the following command to navigate to getletter directory and build your container image again using AWS SAM:
```bash
cd /home/ec2-user/environment/getletter
sam build
```
![image](https://github.com/user-attachments/assets/e016bcdb-ad00-4835-bf9a-e97b55f86850)

You can also test your function using AWS SAM CLI with the following command:
```bash
sam local invoke GetLetter
```
![image](https://github.com/user-attachments/assets/b0066d1a-cb2c-406b-b425-c6d171d1d734)

Upload your Lambda container image to the Amazon Elastic Container Registry (Amazon ECR).
Create an Amazon Elastic Container Registry (ECR) repository by running the following code in your terminal window. If a process is still running in your terminal window, you can press CTRL + C to interrupt it.
```bash
aws ecr create-repository \
    --repository-name getletter \
    --image-scanning-configuration scanOnPush=true
```
![image](https://github.com/user-attachments/assets/e090c00d-4e49-4f1c-b73e-309d6721b667)

Run the following command to tag the local Docker container image to indicate which container you would like to push. Replace 
<repositoryUri>
with the value from the previous step. There will be no output.
```bash
docker tag getletter:latest <repositoryUri>
```
![image](https://github.com/user-attachments/assets/be1d6a66-0fd9-4a2b-ad0c-48277df1bf04)

Run the following command, replacing 
<repositoryUri>
 with the repositoryUri you copied earlier.
```bash
aws ecr get-login-password \
    | docker login \
    --username AWS \
    --password-stdin <repositoryUri>
```
![image](https://github.com/user-attachments/assets/52708c99-4741-40eb-9522-40fe0236be5a)

Run the following command, replacing your 
<repositoryUri>
 copied earlier. Be sure to replace the angle brackets as well.
```bash
docker push <repositoryUri>:latest
```
![image](https://github.com/user-attachments/assets/78efdbd3-888d-4845-a9c9-5edb9510c9be)

Update the samconfig.toml file with the following:
```toml
confirm_changeset = false
resolve_s3 = false
resolve_image_repos = false
```

Deploy the SAM application:
```bash
sam deploy --stack-name getletter --s3-bucket <S3Bucket> --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND --image-repository <repositoryUri>/getletter
```
![image](https://github.com/user-attachments/assets/f02c143c-2148-45a3-b1af-899b98b62f7f)

9. Add API Gateway Trigger
In the AWS Lambda console, add an API Gateway trigger with the following settings:
![image](https://github.com/user-attachments/assets/88a0898f-7ccc-4439-8082-00c2c895f73a)

API: Create a new API
API type: HTTP API
Security: Open
10. Test the Deployed Lambda Function
Access the API Gateway endpoint to download or view the generated PDF.
Under the Configuration tab, click Triggers.
Here you will see the API Gateway endpoint provided by configuring the trigger, which should look similar to the following:
```url
https://abc0123def.execute-api.us-east-1.amazonaws.com/default/getletter-GetLetter-abc012DEF345
```
![image](https://github.com/user-attachments/assets/02925fa9-6174-4b1a-bae5-9bba6a384b1a)
Copy the entire API Endpoint and paste it into the URL bar of a new browser tab.
In about 10-20 seconds, your browser should begin downloading the PDF generated by your function or display the PDF in a new window.

10a.	Open the PDF to see the output file of your function. The output PDF file will include generated placeholder text and resemble a letter.
    ![image](https://github.com/user-attachments/assets/2e6ffc90-ffe9-4c72-930f-95b9a72011b0)
##Modifying Your App
11. Modify the Lambda Function (Optional)
To modify the Lambda function, update the code in app.mjs, rebuild the Docker image, and deploy the updated function using the following commands:
```js
import PDFDocument from 'pdfkit';
import faker from 'faker';

export const lambdaHandler = async (event, context) => {
    
    const pdfBuffer = await new Promise(resolve => {
    const doc = new PDFDocument();
    
    const randomName = faker.name.findName();
    const letterdate = new Date(faker.date.past()).toLocaleDateString('en-us');

    doc.text(randomName, { align: 'right' });
    doc.text(faker.address.streetAddress(), { align: 'right' });
    doc.text(faker.address.secondaryAddress(), { align: 'right' });
    doc.text(faker.address.zipCode() + ' ' + faker.address.city(), { align: 'right' });
    doc.moveDown();
    doc.text(letterdate, { align: 'left' });
    doc.moveDown();
    doc.text('Dear ' + randomName + ',');
    doc.moveDown();
    for(let i = 0; i < 3; i++) {
        doc.text(faker.lorem.paragraph());
        doc.moveDown();
    }
    doc.text(faker.name.findName(), { align: 'right' });
    doc.end();

    const buffers = []
    doc.on("data", buffers.push.bind(buffers))
    doc.on("end", () => {
      const pdfData = Buffer.concat(buffers)
      resolve(pdfData)
    })
  })

    const pdfBase64 = pdfBuffer.toString('base64');

    const response = {
        statusCode: 200,
        headers: {
            'Content-Length': Buffer.byteLength(pdfBase64),
            'Content-Type': 'application/pdf',
            'Content-disposition': 'attachment;filename=test.pdf'
        },
        isBase64Encoded: true,
        body: pdfBase64
    };
    return response;
};
```
12. Once your function has been changed, you will need to re-build, re-tag, and re-deploy your updated container image. Remember that we can use AWS SAM to take care of this for us.
Run the following command to build your modified application and container image:

```bash
sam build
```
13. Run the below command (the same as used earlier) to deploy your modified application using SAM again. Be sure to replace 
<S3Bucket>
 with the S3Bucket name shown to the left of the instructions, as well as 
<repositoryUri>
 with the repositoryUri copied earlier.
```bash
sam deploy \
	--stack-name getletter \
	--s3-bucket <S3Bucket> \
	--capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
	--image-repository <repositoryUri>
```
SAM will detect the changes made, removing, adding, or updating resources as necessary. Within a few minutes, your function will be updated. Once you see the message Successfully created/updated stack - getletter in < region >, your SAM application has been re-deployed, along with the corresponding Lambda function and container image from ECR.

14. Test the API Gateway endpoint URL again by copying it from the Lambda Triggers page and pasting it into a new browser tab URL. Your PDF should now display a date, reflecting changes have been successfully made to the container.
    ![image](https://github.com/user-attachments/assets/5c47bfe6-5979-4727-8162-81ce03957e15)

Conclusion
You've successfully deployed a Lambda function using a container image and triggered it via API Gateway. The function generates a PDF file, demonstrating the use of container images with AWS Lambda.

```js
This `README.md` file provides a comprehensive guide for setting up, deploying, and testing the Lambda container function, along with instructions for modifying the application.
```
