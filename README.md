
## 1. Objective
As enterprises start to embrace serverless paradigm (AWS Lambda), there is an increasing need for guidance on how to set up an automated AWS CodePipeline for the AWS Lambda functions. In enterprises, Microsoft .NET occupy a signifant footprint in technology stack. In the recent times, .NET core is also gaining momentum for various reasons.


When it comes to creating and deploying AWS Lambda functions in .NET core,
there are couple of options available. The option #1 is to leverage Visual Studio 2019. Then, the  option #2 is to leverage AWS Lambda Dotnet CLI. This post will cover how to set up an automated AWS CodePipeline for both the options. The detailed steps for creating AWS CodePipeline for option #1 is covered in the sections 2, 2a, to 2g. Also, the detailed steps for creating AWS CodePipeline for option #2 is covered in the sections 3, 3a to 3h.
                  

## 2. AWS CodePipeline for Dotnet Lambda functions created using Visual Studio
The following forms the pre-requisite in the windows environment.
- Windows 10 with latest updates.
- Visual Studio 2019 with latest updates.
- AWS toolkit for Visual Studio


## 2a. Create an AWS Lambda function in Visual Studio 2019

Open Visual Studio 2019, View --> Team Explorer --> Manage Connections.

You will see connections of various providers such as AWS Code Commit, Local Git repositories and other hosted providers etc.
If you have not configured connection for AWS Code Commit, you can set it up by providing AWS Code Commit Git Https Credentials.


Click 'Create' under AWS CodeCommit provider.

<p align="center">
<img src="/images/pic11.JPG">
</p>


Once the AWS CodeCommit repository is created successfully, you will get a message like "The repository was cloned successfully
. Create a new project or solution in the repository." in the Team Explorer.

<p align="center">
<img src="/images/pic2.JPG">
</p>


Go ahead and create a Visual Studio Solution and Project of type AWS Lambda.

<p align="center">
<img src="/images/pic3.JPG">
</p>

Select 'Empty Function' for 'Select Blueprint'.

<p align="center">
<img src="/images/pic4.JPG">
</p>



## 2b. Changes to aws-lambda-tools-defaults.json

Open the default aws-lambda-tools-defaults.json created with Visual Studio Project and it should look the following.

``` json
{
  "Information" : [
    "This file provides default values for the deployment wizard inside Visual Studio and the AWS Lambda commands added to the .NET Core CLI.",
    "To learn more about the Lambda commands with the .NET Core CLI execute the following command at the command line in the project root directory.",

    "dotnet lambda help",

    "All the command line options for the Lambda command can be specified in this file."
  ],


  "profile":"sundarprofile",
  "region" : "us-east-1",
  "configuration" : "Release",
  "framework" : "netcoreapp2.1",
  "function-runtime":"dotnetcore2.1",
  "function-memory-size" : 256,
  "function-timeout" : 30,
  "function-handler" : "Dotnetlambda4::Dotnetlambda4.Function::FunctionHandler"
}

``` 
Make two changes to aws-lambda-tools-defaults.json file. The first one is about the profile. In my local machine, it is pointing to the AWS profile 'sundarprofile' created in Visual Studio. So, change this to default profile. If your's has the 'default' as profile, leave it unchanged. For the second change, add an IAM role that needs to be assumed by Lambda function.

The updated aws-lambda-tools-defaults.json should look like below.


``` json
{
  "Information": [
    "This file provides default values for the deployment wizard inside Visual Studio and the AWS Lambda commands added to the .NET Core CLI.",
    "To learn more about the Lambda commands with the .NET Core CLI execute the following command at the command line in the project root directory.",

    "dotnet lambda help",

    "All the command line options for the Lambda command can be specified in this file."
  ],

  "profile": "default",
  "region": "us-east-1",
  "configuration": "Release",
  "framework": "netcoreapp2.1",
  "function-runtime": "dotnetcore2.1",
  "function-memory-size": 256,
  "function-timeout": 30,
  "function-handler": "Dotnetlambda4::Dotnetlambda4.Function::FunctionHandler",
  "function-role": "arn:aws:iam::yourawsaccountnumber:role/Sundarfulllambdarole"
}


``` 

## 2c. Add buildspec.yml

Add buildspec.yml at the root of the AWS CodeCommit repository.  


``` .yml
version: 0.2
env:
  variables:
    DOTNET_ROOT: /root/.dotnet
  secrets-manager:
    AWS_ACCESS_KEY_ID_PARAM: CodeBuild:AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY_PARAM: CodeBuild:AWS_SECRET_ACCESS_KEY
phases:
  install:
    runtime-versions:
      dotnet: 3.1
  pre_build:
    commands:
      - echo Restore started on `date`
      - export PATH="$PATH:/root/.dotnet/tools"
      - pip install --upgrade awscli
      - aws configure set profile $Profile
      - aws configure set region $Region
      - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID_PARAM
      - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY_PARAM
      - cd Dotnetlambda4
      - cd src
      - cd Dotnetlambda4
      - dotnet clean 
      - dotnet restore
  build:
    commands:
      - echo Build started on `date`
      - dotnet new -i Amazon.Lambda.Templates::*
      - dotnet tool install -g Amazon.Lambda.Tools
      - dotnet tool update -g Amazon.Lambda.Tools
      - dotnet lambda deploy-function "Dotnetlambda4" --function-role "arn:aws:iam::yourawsaccountnumber:role/Sundarfulllambdarole" --region "us-east-1"
``` 

The above buildspec.yml installs .NET Core 3.1, sets the path of Dotnet core on the build enviornment and refers the access keys from secret store. It then configures aws cli tool.
Finally it installs Amazon Lambda Templates & Amazon Lambda Tools for Dotnet core and deploys the Lambda function dotnet lambda cli.


## 2d. Push to AWS CodeCommit repository

Navigate to the directory where the AWS CodeCommit repository is cloned locally. Execute the following commands to push changes to remote AWS CodeCommit repository.

```
git add --all
git commit --all
git push

```


## 2e. Define AWS CodePipline in AWS Console

Proceed to define AWS CodePipeline for deploying Lambda functions.

Name the Pipeline with any arbitrary name.

<p align="center">
<img src="/images/pic5.JPG">
</p>

Select 'AWS CodeCommit' as SourceProvider and also pick the right repository and branch.

<p align="center">
<img src="/images/pic6.JPG">
</p>


Select 'AWS CodeBuild' as Build provider.

<p align="center">
<img src="/images/pic7.JPG">
</p>


Proceed to create a new CodeBuild project. Name the project with any arbitrary name. For Environment select 'Managed Image' as Environment Image and 'Ubuntu' as operating system.

<p align="center">
<img src="/images/pic8.JPG">
</p>


Select 'aws/codebuild/standard:4.0' as Image and 'Always use the latest image for this runtime version' for Image version.

 <p align="center">
<img src="/images/pic9.JPG">
</p>

Now you can see the successful creation of AWS CodeBuild project.

<p align="center">
<img src="/images/pic10.JPG">
</p>



Proceed to define these four environment variables Profile, Region,  and SecretAccessKey in the CodeBuild environment settings.

<p align="center">
<img src="/images/pic12.JPG">
</p>



Setting these environment variables will assign the right aws profile for the CodeBuild environment. If you see above, the AccessKAccessKeyId and SecretAccessKey are configured as PlainText. I'm doing this for the simplicity of the demo. For production purposes, leverage parameter store and AWs Secrets.

The CodeDeploy is an optional stage in the AWS CodePipeline. Skip this to complete the creation of the AWS CodePipeline.

<p align="center">
<img src="/images/pic11.JPG">
</p>

## 2f. Configuration for successful CodePipeline

The following things need to be ensured for the successful CodeBuild exectuion.

- Name of Profile mentioned in the aws-lambda-tools-defaults.json should match with Profile set in the Environment section of Codebuild.
- The attribute "function-role" needs to be mandatorily set with an appropriate IAM role in the  aws-lambda-tools-defaults.json.
- The 'Dotnet lambda deploy-function' should be invoked from the directory where the .csproj of lambda project lives.


## 2g. Completion and Verification

Save the creation of AWS CodePipeline. Push the code changes of Lambda function from local repository to remote AWS CodeCommit repository.
After few seconds, you should see the trigger of AWS CodeCommit stage and transition to AWS CodeBuild stage. Then AWS Code Pipeline should complete successfully after few minutes.

<p align="center">
<img src="/images/pic15.JPG">
</p>


You can also see the successful creation of AWS Lambda function from AWS Code Pipeline.

<p align="center">
<img src="/images/pic16.JPG">
</p>

This completes the section 2.

## 3. AWS CodePipeline for Dotnet Lambda function created in Visual Studio
In this section, i'll cover how to setup an AWS CodePipeline for Lambda functions created using 'AWS Dotnet Lambda CLI'. This will be useful for MacOS and Linux environments.

Here is the typical development environment that you will need.
- Mac OS lates version or Linux (supported distros for .NET core 3.1) with latest updates. 
- .NET Core 3.1 or higher.
- AWS CLI
- AWS Dotnet Lambda CLI

## 3a. Create an AWS CodeCommit repository
Create an AWS CodeCommit repositry in the console.

<p align="center">
<img src="/images/pic14.JPG">
</p>

Clone the repository locally using Git credentials.

## 3b. Create an AWS Lambda function using AWS Dotnet Lambda CLI

Install the nuget package 'Amazon.Lambda.Templates' to have all the AWS Lambda templates in the environment.


```
dotnet new -i Amazon.Lambda.Templates

```

Verify the installation by issuing this command.


```
dotnet new -all

```

You should see the following output and many more .NET core template types listed there.


<p align="center">
<img src="/images/pic13.JPG">
</p>

Navigate to the cloned repository (created in section #3a) and Create an AWS Lambda function using the below command.

```
dotnet new lambda.EmptyFunction --name Dotnetlambda4 --profile default --region us-east-1

```


## 3c. Changes to aws-lambda-tools-defaults.json

Make two changes to aws-lambda-tools-defaults.json file. The first one is about the profile. In my local machine, it is pointing to the AWS profile 'sundarprofile' created in Visual Studio. So, change this to default profile. If your's has the 'default' as profile, leave it unchanged. For the second change, add an IAM role that needs to be assumed by Lambda function.

The updated aws-lambda-tools-defaults.json should look like below.

``` json
{
  "Information" : [
    "This file provides default values for the deployment wizard inside Visual Studio and the AWS Lambda commands added to the .NET Core CLI.",
    "To learn more about the Lambda commands with the .NET Core CLI execute the following command at the command line in the project root directory.",

    "dotnet lambda help",

    "All the command line options for the Lambda command can be specified in this file."
  ],

  "profile":"default",
  "region" : "us-east-1",
  "configuration" : "Release",
  "framework" : "netcoreapp2.1",
  "function-runtime":"dotnetcore2.1",
  "function-memory-size" : 256,
  "function-timeout" : 30,
  "function-handler" : "Dotnetlambda4::Dotnetlambda4.Function::FunctionHandler",
  "function-role": "arn:aws:iam::awsaccountno:role/IAMrole"
}

```


## 3d. Add buildspec.yml

Add buildspec.yml at the root of the AWS CodeCommit repository. I mean, add it to the root of the local CodeCommit repository. It should look like the following.

``` json
version: 0.2
env:
  variables:
    DOTNET_ROOT: /root/.dotnet
  secrets-manager:
    AWS_ACCESS_KEY_ID_PARAM: CodeBuild:AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY_PARAM: CodeBuild:AWS_SECRET_ACCESS_KEY
phases:
  install:
    runtime-versions:
      dotnet: 3.1
  pre_build:
    commands:
      - echo Restore started on `date`
      - export PATH="$PATH:/root/.dotnet/tools"
      - pip install --upgrade awscli
      - aws configure set profile $Profile
      - aws configure set region $Region
      - aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID_PARAM
      - aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY_PARAM
      - cd Dotnetlambda4
      - cd src
      - cd Dotnetlambda4
      - dotnet clean 
      - dotnet restore
  build:
    commands:
      - echo Build started on `date`
      - dotnet new -i Amazon.Lambda.Templates::*
      - dotnet tool install -g Amazon.Lambda.Tools
      - dotnet tool update -g Amazon.Lambda.Tools
      - dotnet lambda deploy-function "Dotnetlambda4" --function-role "arn:aws:iam::yourawsaccountnumber:role/Sundarfulllambdarole" --region "us-east-1"
```

## 3e. Push to AWS CodeCommit repository
Navigate to the directory where the AWS CodeCommit repository is cloned locally. Execute the following commands to push changes to remote AWS CodeCommit repository.

```
git add --all
git commit --all
git push

```

## 3f. Define AWS CodePipline in AWS Console
Follow the steps mentioned in section 2e for defining an AWS CodePipeline.

## 3g. Configuration for successful CodePipeline
Follow the checklist mentioned in section 2f for successful execution of Pipeline.


## 3h. Completion and Verfification

Save the creation of AWS CodePipeline. Push the code changes of Lambda function from local repository to remote AWS CodeCommit repository.
After few seconds, you should see the trigger of AWS CodeCommit stage and transition to AWS CodeBuild stage. Then AWS Code Pipeline should complete successfully after few minutes.

<p align="center">
<img src="/images/pic15.JPG">
</p>


You can also see the successful creation of AWS Lambda function from AWS Code Pipeline.

<p align="center">
<img src="/images/pic17.JPG">
</p>

This completes the section .


## 4. Conclusion

This completes the post of creation an automated AWS Code Pipeline for AWS Lambda functions created using the two approaches (mentioned above).