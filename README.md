# Building operational tasks with lambda functions
In this blog post I would like to share an approach to easily develop, test and deploy an operational task in AWS. For example, to suspend AWS resources outside business hours. To schedule back-ups of AWS services which do not meet your requirements. Or to discover abuse of the powers users usually get, and take immediate action.

Lambci did a great job in creating a solution to [test a lambda function](https://github.com/lambci/docker-lambda) locally with Docker. Operational lambda functions use other AWS services as well. After writing your lambda function, you have to create a script to deploy the function, and this includes writing a policy and role. It also takes some time to update and invoke te function, just to test. Wouldn't it be great if the lambda function in Docker runs with the same role? Even deploying roles and policies becomes an easy job, and you're near to deploy your first operational task. Most DevOps engineers understand Python, or will at least be able to copy+paste+rewrite python scripts, so that's why I've chosen Python instead of any other supported language.

Since botocore is outdated in AWS production environment (!!!), and lambci docker-lambda logically too, another bonus is added to give you an easy way to install botocore as part of your function regardless the OS you're using. Why did I care? Well, I wanted to stop my development RDS instances, and the `rds.stop_instances()` is part of a more recent botocore library.

I could have easily distribute a cloudformation or terraform template, but this blog post is not only there to deploy your function to production. Rather it has the intend to learn more about AWS than just deploying a lambda function.

## Prerequisites
Before we start, let's set some variables. Change them here if you don't want to replace them later in this guide. Or put it in a file __settings.cfg__ and initialize them with the command ```source settings.cfg```.

```
ACCOUNT_ID=1234567890
AWS_PROFILE=yourprofile
AWS_REGION=eu-west-1
AWS_USERNAME=myusername
FUNCTIONNAME=MyOpsFunction
ROLENAME=LambdaExecutionRole
STACKNAME=RolePolicyForMyDocker
```
Other prerequisites; verify if the following commands work on your command line: `jq`, `docker`, `zip`, `aws`. Otherwise use Google to learn how to install and use these tools. You should also have a basic understanding of Docker, AWS in general, and of the AWS CLI with multiple profiles.

Also set your default aws profile with the following command, otherwise you must add --profile in every aws command. You can skip this step if you only have the default profile.

```bash
export AWS_DEFAULT_PROFILE=<yourprofilename>
```

## Deploy the role to AWS
For the assume_role.json policy document, you copy and paste the following code into __assume\_role.json__, and replace <account_id> for your account_id (similar to: 111122223333), and replace <username> for your username corresponding to the profile you're using in the CLI. It basically  allows services like Lambda, and your username to _assume_ this role. _Assume_ means: generate temporary credentials and use them in this case to access AWS resources. It also allows Lambda to assume the same role.

```javascript
{
  "Version" : "2012-10-17",
  "Statement": [ {
    "Effect": "Allow",
    "Principal": {
      "Service": [ "lambda.amazonaws.com" ]
    },
    "Action": [ "sts:AssumeRole" ]
  },
  {
    "Effect": "Allow",
    "Principal": {
      "AWS": [ "arn:aws:iam::<account_id>:user/<username>" ]
    },
    "Action": [ "sts:AssumeRole" ]
  } ]
}
```
The policy itself is copied to __lambda\_policy.json__, which currently gives Lambda access to write logs and list all buckets in your account. You should always develop with the principle of "Least Privilege" in mind. So if you're just stopping ec2 instances, don't specify `ec2:*` or even `*.*` as the action, but just the specific permission: `ec2:StopInstances`. Consider using the specific resource ARNs you own within your account, by specifying the Resource condition too.

```javascript
{
  "Version": "2012-10-17",
  "Statement": [ {
      "Sid": "BaseLambdaPolicyForLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Sid": "YourLambdaPolicy",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    } ]
}
```
When you created the files __lambda\_policy.json__ and __assume\_role.json__ including the appropriate permissions, you can run the following commands to deploy the role and policy to AWS. Make sure the variables are set correctly. 

```bash
aws iam create-role \
--role-name ${ROLENAME} \
--assume-role-policy-document file://assume_role.json
```

```bash
aws iam put-role-policy \
--role-name ${ROLENAME} \
--policy-name ${ROLENAME}-policy \
--policy-document file://lambda_policy.json
```
## Develop a basic lambda function
Create a new folder and file: __lambda_function/lambda\_function.py__, and copy the following content. It doesn't contain any real functionality, it's just printing the current boto3 and botocore version for now. Also logging is included, because this is a very useful library for structured logging.

```python
import boto3
import json
import botocore
import logging

log = logging.getLogger(__name__)
log.setLevel(logging.INFO)
logging.basicConfig(
	level=logging.INFO,
	format="%(asctime)s - %(name)s (%(lineno)s) - %(levelname)s: %(message)s",
	datefmt='%Y.%m.%d %H:%M:%S')

def lambda_handler(event, context):
	log.info("Boto3 Version: " + boto3.__version__)
	log.info("Botocore Version: " + botocore.__version__)
```

## Test the lambda function
First let's test if the lambda function works, just like lambci intended. Run the command from the project directory. If you're not executing this command from the project directory, you'll get the error: `Unable to import module 'lambda_function': No module named lambda_function`.

```
docker run -v "$PWD"/lambda_function:/var/task lambci/lambda:python2.7
```
Output:

```bash
START RequestId: ... Version: $LATEST
[INFO]	2017-08-04T18:55:22.891Z	2e29785e-c4ef-41cb-ab8c-2fbe56aec341	Boto3 Version: 1.4.4
[INFO]	2017-08-04T18:55:22.891Z	2e29785e-c4ef-41cb-ab8c-2fbe56aec341	Botocore Version: 1.5.19
END RequestId: ...
REPORT RequestId: ... Duration: 217 ms Billed Duration: 300 ms 
  Memory Size: 1536 MB Max Memory Used: 20 MB
null
```

Btw, the null in the end, is because the lambda_function does not return anything.

## Install botocore and run again
At the command line, go into the lambda_function folder and then execute the following command. With `-v "$PWD":/localdir` Docker mounts the current dir to /localdir in the container. `python:2.7-alpine` is the Docker image on Dockerhub which we are going to use. `pip install botocore -t /localdir` is the command being executed in the container. It will download botocore and with -t it's installed to the specified target folder.

```bash
docker run -v "$PWD"/lambda_function:/localdir python:2.7-alpine \
    pip install botocore -t /localdir
```

After running the previous Docker command, now again invoke the Lambda function, and notice the change of the versions. Of course they will be different for you.

```bash
docker run -v "$PWD"/lambda_function:/var/task lambci/lambda:python2.7
```

Output:

```bash
START RequestId: ... Version: $LATEST
[INFO]	2017-08-04T18:57:22.891Z	2e29785e-c4ef-41cb-ab8c-2fbe56aec341	Boto3 Version: 1.4.4
[INFO]	2017-08-04T18:57:22.891Z	2e29785e-c4ef-41cb-ab8c-2fbe56aec341	Botocore Version: 1.5.92
END RequestId: ...
REPORT RequestId: ... Duration: 217 ms Billed Duration: 300 ms 
  Memory Size: 1536 MB Max Memory Used: 20 MB
```

## Run lambda with role
Make sure you have deployed the role to AWS, and replaced or set the variables `${ACCOUNT_ID}`, `${ROLENAME}`, `${AWS_REGION}`. The main part of the script is the aws sts assume-role, which generates new API access keys, which will be sent to Docker when it executes the container with your lambda function.

```bash
temp_role=$( aws sts assume-role \
               --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/${ROLENAME}" \
               --role-session-name "justsomerandomcharacters" )
export A_ACCESS_KEY_ID=$(echo $temp_role | jq .Credentials.AccessKeyId | xargs)
export A_SECRET_ACCESS_KEY=$(echo $temp_role | jq .Credentials.SecretAccessKey | xargs)
export A_SESSION_TOKEN=$(echo $temp_role | jq .Credentials.SessionToken | xargs)
env | grep -i A_
docker run -e AWS_ACCESS_KEY_ID=${A_ACCESS_KEY_ID} \
           -e AWS_SECRET_ACCESS_KEY=${A_SECRET_ACCESS_KEY} \
           -e AWS_REGION=${AWS_REGION} \
           -e AWS_SESSION_TOKEN=${A_SESSION_TOKEN} \
           -v "$PWD"/lambda_function:/var/task lambci/lambda:python2.7
```
Now you could add some more lines to your lambda function and check if your solution works. The following command reads all S3 buckets and list them in the output. If you don't have S3 buckets, you should first create one. Bonus points for adding the permission to your policy and using boto3 to create this bucket.

```python
...
def lambda_handler(event, context):
	print "Boto3 Version: " + boto3.__version__
	print "Botocore Version: " + botocore.__version__
	s3r = boto3.resource('s3')
	r = []
	for bucket in s3r.buckets.all():
		r.append(bucket.name)
	return r
```

## Deploy to AWS
You can now deploy your function using the Management Console, Cloudformation or other tooling, or the Command Line Interface (CLI), which is used below. Because the role we created earlier also allows Lambda to assume the role, we just create the function and invoke it manually. 

First, we need to zip the function, because of the libraries we need to include. After creating the zip file, we can create the function.

```bash
cd lambda_function && zip -r9 ../lambda_function.zip * && cd ..
```
```bash
aws lambda create-function \
--function-name ${FUNCTIONNAME} \
--runtime python2.7 \
--role arn:aws:iam::${ACCOUNT_ID}:role/${ROLENAME} \
--handler lambda_function.lambda_handler \
--zip-file fileb://lambda_function.zip
```

To manually invoke the lambda function and check the results:

```bash
aws lambda invoke \
--function-name ${FUNCTIONNAME} \
output.txt
cat output.txt
```
Output:

```bash
["some", "of", "your", "buckets"]
```

To update the lambda funtion in AWS, if you made changes and tested locally:

```bash
cd lambda_function && zip -r9 ../lambda_function.zip * && cd ..
aws lambda update-function-code \
--function-name ${FUNCTIONNAME} \
--zip-file fileb://lambda_function.zip
```
In the Management Console you're able to find your Lambda Function and check all the details. It's also possible to Invoke the function there, and to find the logs.

## Schedule the lambda function
Now the function is in place and tested correctly, we can schedule it. There are 2 functions available: cron() and rate(). The cron() is mainly useful in case something must happen at a specific time every hour, day, week, while the rate() is there to trigger every x minutes or x hours. Some examples and more information can be found at the [AWS Documentation about scheduled events](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html). Since we're not doing anything special, and I would like to have some proof everything runs, I recommend using rate(5 minutes).

```bash
aws events put-rule \
--name ${FUNCTIONNAME}Trigger \
--schedule-expression "rate(5 minute)"
```

```bash
aws lambda add-permission \
--function-name ${FUNCTIONNAME} \
--statement-id ${FUNCTIONNAME}Trigger \
--action 'lambda:InvokeFunction' \
--principal events.amazonaws.com \
--source-arn arn:aws:events:${AWS_REGION}:${ACCOUNT_ID}:rule/${FUNCTIONNAME}Trigger
```

```bash
aws events put-targets \
--rule ${FUNCTIONNAME}Trigger \
--targets "Id"="1","Arn"="arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:${FUNCTIONNAME}"
```
Go to the Management Console, go to __CloudWatch__, go to __Events__ and then __Rules__. Click on the event you just created and check the metrics and other details.

## Clean up
Don't forget to clean up once you're finished with your function, or update the policy of the role by removing your principle to assume access. These commands will also be useful if you made any errors and need to revert things. 

```bash
aws events remove-targets \
--rule ${FUNCTIONNAME}Trigger \
--ids 1

aws events delete-rule \
--name ${FUNCTIONNAME}Trigger

aws lambda remove-permission \
--function-name ${FUNCTIONNAME} \
--statement-id ${FUNCTIONNAME}Trigger

aws lambda delete-function \
--function ${FUNCTIONNAME}

aws iam delete-role-policy \
--role-name ${ROLENAME} \
--policy-name ${ROLENAME}-policy

aws iam delete-role \
--role-name ${ROLENAME}
```

## Wrap up
Does it feel useful? I guess it will. Does it feel complex for just a simple function? Yes, it does for sure. Although with the knowledge you now have, you could easily write a Terraform or Cloudformation script, and make an automated deployment with any prefered deployment tool. If you structure this well, and you have a lot of operational tasks to upload and maintain, it becomes an easy job.