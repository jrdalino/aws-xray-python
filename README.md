# myproject-aws-xray-eks-python

## Step 1: Instrument and run your back end application (Python Flask)
- Install the X-Ray SDK
```
$ pip install aws-xray-sdk flask zappa requests
```
- Add middleware to instrument incoming HTTP requests if using Flask
- import x-ray modules, patch the requests module, configure x-ray recorder, and instrument the flask app
```
# Import the X-Ray modules
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
from aws_xray_sdk.core import patcher, xray_recorder
from flask import Flask
import requests

# Patch the requests module to enable automatic instrumentation
patcher.patch(('requests',))

app = Flask(__name__)

# Configure the X-Ray recorder to generate segments with our service name
xray_recorder.configure(service='My First Serverless App')

# Instrument the Flask application
XRayMiddleware(app, xray_recorder)
 
@app.route('/')
def hello_world():
    resp = requests.get("https://aws.amazon.com")
    return 'Hello, World: %s' % resp.url
```
- Reference: https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python-serverless.html

## Step 2: Instrument front end application
- Reference: https://docs.aws.amazon.com/xray/latest/devguide/scorekeep-client.html

## Step 3: Run the AWS X-Ray Daemon
The AWS X-Ray SDK does not send trace data directly to AWS X-Ray. To avoid calling the service every time your application serves a request, the SDK sends the trace data to a daemon, which collects segments for multiple requests and uploads them in batches. Use a script to run the daemon alongside your application.

### EC2 Linux
- Run the X-Ray daemon with a user data script.
```
#!/bin/bash
curl https://s3.dualstack.ap-southeast-2.amazonaws.com/aws-xray-assets.ap-southeast-2/xray-daemon/aws-xray-daemon-2.x.rpm -o /home/ec2-user/xray.rpm
yum install -y /home/ec2-user/xray.rpm
```

### Elastic Beanstalk
- Elastic Beanstalk platforms released since December 2016 include the X-Ray daemon. Enable the daemon with a configuration file or by setting the corresponding option in Elastic Beanstalk console.
```
# .ebextensions/xray-daemon.config
option_settings:
  aws:elasticbeanstalk:xray:
    XRayEnabled: true
```

### Lambda
AWS Lambda automatically runs the X-Ray daemon when you enable tracing on your function.
References:
- https://docs.aws.amazon.com/lambda/latest/dg/lambda-x-ray.html
- https://docs.aws.amazon.com/lambda/latest/dg/using-x-ray.html
- https://docs.aws.amazon.com/lambda/latest/dg/downstream-tracing.html
- https://docs.aws.amazon.com/lambda/latest/dg/lambda-x-ray-daemon.html
- https://docs.aws.amazon.com/xray/latest/devguide/xray-services-lambda.html

### Amazon ECS/EKS
- Add Policy to the Worker Node IAM Role. Using AWS CLI, add the following policy to the worker node's IAM role
```
$ aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```
-  Create a folder and download the daemon
```
$ mkdir xray-daemon && cd xray-daemon
$ curl https://s3.dualstack.ap-southeast-2.amazonaws.com/aws-xray-assets.ap-southeast-2/xray-daemon/aws-xray-daemon-linux-2.x.zip -o ./aws-xray-daemon-linux-2.x.zip
$ unzip -o aws-xray-daemon-linux-2.x.zip -d .
```
- Create a Dockerfile with the following content.
```
FROM ubuntu:12.04
COPY xray /usr/bin/xray-daemon
CMD xray-daemon -f /var/log/xray-daemon.log &
```
-  Build the image and Push to new ECR with name: xray
```
$ $(aws ecr get-login --no-include-email --region ap-southeast-2)
$ docker build -t xray .
$ docker tag xray:latest 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/xray:latest
$ docker push 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/xray:latest
```
- Deploy X-Ray as a DaemonSet, Validate and View logs
```
$ kubectl create -f https://github.com/jrdalino/myproject-aws-xray-eks-python/blob/master/xray-k8s-daemonset.yaml
```
- View status and logs for all the X-Ray demon pods
```
$ kubectl describe daemonset xray-daemon
$ kubectl logs -l app=xray-daemon
```

## Step 4: Deploy and Test Application, Examine and Traces X-Ray Console
- View Service Map
- View Traces > Trace Overview
- View Traces > Trace Details

## Step 5: Enable Upstream Rest API Gateway Tracining
References:
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-xray.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-enabling-xray.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-using-xray-maps.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-configuring-xray-sampling-rules.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-understanding-xray-traces.html
- https://docs.aws.amazon.com/xray/latest/devguide/xray-services-apigateway.html

## Step 6 : Enable Downstream Tracing for DynamoDB

## Step 7: App Mesh Integration

## Step 8: CloudTrail Integration

## Step 9: Cloudwatch Integration

## Step 10: AWS Config Integration

## Step 11: Elastic Load Balancer Integration

## References
- https://aws.amazon.com/blogs/compute/application-tracing-on-kubernetes-with-aws-x-ray/ 
- https://eksworkshop.com/intermediate/245_x-ray/x-ray-daemon/
- https://github.com/aws-samples/reinvent2018-dev303-code
- https://github.com/GoogleCloudPlatform/microservices-demo
