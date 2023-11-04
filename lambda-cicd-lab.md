
***This whole lab can be completed using AWS CloudShell***

# Lab 1 - Invoking functions using versions and aliases

1. Create a Lambda function using Python

2. Invoke function synchronously

aws lambda invoke --function-name lambda-cicd-lab response.json

3. Create versions and an alias and invoke alias ARN

aws lambda invoke --function-name arn:aws:lambda:us-east-1:927633303628:function:lambda-cicd:1 response.json

# Lab 2 - Create an CodeCommit Repository and Configure Source Code

1. Login to AWS CloudShell and configure the CodeCommit credential helper

```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

2. Create a CodeCommit Repository using the AWS CLI

```bash
aws codecommit create-repository --repository-name LambdaRepo --repository-description "Repo for Lambda Deployment Lab"
```

3. Clone the repository

```bash
git clone <clone-https-url-from-the-output-above>
```

4. Change into the repository directory

```
cd LambdaRepo
```

5. Create the Lambda Function file named `lambda_function.py`

```python
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Third Version'
    }
```

6. Create the Buildspec file named `buildspec.yml`:

```yml

version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.11
  build:
    commands:
      - echo 'Building...'
      - zip -r lambda_function.zip lambda_function.py
  post_build:
    commands:
      - echo 'Build completed.'
      - aws lambda update-function-code --function-name lambda-cicd-lab --zip-file fileb://lambda_function.zip
      - echo 'Waiting for 30 seconds before publishing version...'
      - sleep 30
      - aws lambda publish-version --function-name lambda-cicd-lab

artifacts:
  files: lambda_function.zip

```

7. Commit & Push the Code to CodeCommit

```bash
git add .
git commit -m "Initial lambda function and buildspec"
git push origin master
```

***If you're asked for configuration information to complete the commit, use the following format***

```bash
git config --global user.name "Jane Doe"
git config --global user.email janedoe@example.com
```

# Lab 3 - Create CodeBuild Project

1. Navigate to CodeBuild in the AWS Management Console
   
2. Create a Service Role for CodeBuild with a trust policy for AWS CodeBuild

3. Attach necessary permissions to this role. For the scope of this lab, we'll provide permissions to CodeCommit, S3 (for build artifacts), and Lambda (for deployments):

- AWSCodeCommitFullAccess
- AmazonS3FullAccess
- AWSLambda_FullAccess

4. Create the CodeBuild Project:

- Name: LambdaLab
- Source Provider: CodeCommit with master branch
- Environment: Managed image, Amazon Linux 2 standard 5.0
- Service role: the one we just created
- Buildspec: No need to name it but leave enabled
- Optionally enable logging to CloudWatch (recommended)

# Testing

1. Start a build in CodeBuild via the console - the "Third Version" should be deployed

2. Update Lambda Function Code

```python
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Fourth Version'
    }
```

2. Commit & Push the Updated Code

```bash
git commit -am "Updated lambda function."
git push origin master
```

3. Start another build in CodeBuild via the console

## Lab 4 - Add Automation with CodePipeline

1. Create an AWS CodePipeline to automate the build process

2. Push a code update and the whole deployment should be automated

3. Change the version the alias points to

```bash
aws lambda update-alias --function-name myfirstfunction --name myalias --function-version 5
aws lambda update-alias --function-name lambda-cicd --name myAlias --routing-config AdditionalVersionWeights={}
```

## Lab 5 - Publish Function through an API

1. Create the API in API Gateway
- Create a REST API
- Create a GET method
- Integrate the Lambda function as a Lambda proxy integration
- Deploy to the "prod" stage (create it)

2. Test the API:
```bash
curl -X GET https://bagpw9svjc.execute-api.us-east-1.amazonaws.com/prod/
```

## Lab 6 - Add Web Front End (if we have time)

1. Create an S3 static website
- Enable public access
- Enable static website hosting 
- Set the default document to index.html

2. Edit the API endpoint in the following code and then save as "index.html":

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>API Data Display</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin-top: 50px;
        }

        #result {
            border: 1px solid #ddd;
            padding: 20px;
            display: inline-block;
        }
    </style>
</head>
<body>
    <h2>API Response</h2>
    <div id="result">
        Loading...
    </div>

    <script>
        // Fetch data from the API Gateway URL
        fetch('https://bagpw9svjc.execute-api.us-east-1.amazonaws.com/prod/')
            .then(response => {
                if(!response.ok) {
                    throw new Error('Network response was not ok');
                }
                return response.text();  // Changed from response.json() to response.text()
            })
            .then(data => {
                // Display data on the webpage
                document.getElementById('result').textContent = data;  // Displaying the plain text data
            })
            .catch(error => {
                console.log('There was a problem with the fetch operation:', error.message);
                documxent.getElementById('result').textContent = 'Error fetching data';
            });
    </script>
</body>
</html>
```
3. Update the code to include the CORS controls

```python
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'headers': {
            "Access-Control-Allow-Origin": "*",  # Allow any origin (not recommended for production)
            "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
            "Access-Control-Allow-Methods": "OPTIONS,GET,POST,PUT,DELETE"  # You can adjust this based on your needs
        },
        'body': 'TEXT TO RETURN'
    }
```
4. Commit and push the code updates
5. Ensure CORS is enabled on the API and update the stage
6. Open the static website endpoint to test the solution




