# aws-xray-python

## Step 1: Instrument and run your back end application (Python Flask)
- Install the X-Ray SDK
```
$ pip install aws-xray-sdk flask requests
```
- Add middleware to instrument incoming HTTP requests if using Flask
- import x-ray modules, patch the requests module, configure x-ray recorder, and instrument the flask app
```
import os
import config
from flask import Flask, request
from aws_xray_sdk.core import patch_all, xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

app = Flask(__name__)

xray_recorder.configure(
    context_missing='LOG_ERROR',
    service=config.XRAY_APP_NAME,
)

patch_all()

XRayMiddleware(app, xray_recorder)

@app.route('/helloworld')
def ping():
    return 'Hello World'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=config.PORT, debug=config.DEBUG_MODE)
```

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
- without flask
```
import json
import os
import decimal
import boto3

# Import the X-Ray modules
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch all requests
patch_all()

# This is a workaround for: http://bugs.python.org/issue16535
class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, decimal.Decimal):
            return int(obj)
        return super(DecimalEncoder, self).default(obj)

def lambda_handler(event, context):

    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('todos')

    # fetch all todos from the database
    result = table.scan()

    # create a response
    response = {
        "statusCode": 200,
        "body": json.dumps(result['Items'], cls=DecimalEncoder)
    }

    return response
```


- with flask
```
from flask import Flask
import requests

# Import the X-Ray modules
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
from aws_xray_sdk.core import patcher, xray_recorder

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

### Amazon ECS/EKS
- Add Policy to the Worker Node IAM Role. Using AWS CLI, add the following policy to the worker node's IAM role
```
$ aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```
-  Create a folder and download the daemon and create a Dockerfile as per https://github.com/jrdalino/aws-xray-python/blob/master/Dockerfile
```
$ mkdir xray-daemon && cd xray-daemon
```
-  Create ECR Repo, build the image and Push to ECR
```
$ aws ecr create-repository --repository-name xray-daemon
$ $(aws ecr get-login --no-include-email --region ap-southeast-2)
$ docker build -t xray-daemon .
$ docker tag xray-daemon:latest 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/xray-daemon:latest
$ docker push 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/xray-daemon:latest
```
- Deploy X-Ray as a DaemonSet, Validate and View logs
```
$ cd ~/environment/aws-xray-python/kubernetes/xray-k8s-daemonset.yaml
$ kubectl create -f xray-k8s-daemonset.yaml
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

## Step 5: Enable Upstream Tracing for Rest API Gateway
References:
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-xray.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-enabling-xray.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-using-xray-maps.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-configuring-xray-sampling-rules.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-understanding-xray-traces.html
- https://docs.aws.amazon.com/xray/latest/devguide/xray-services-apigateway.html

## Step 6 : Enable Downstream Tracing for DynamoDB

## Step 7: App Mesh Integration
- To configure the Envoy proxy to send data to X-Ray, set the ENABLE_ENVOY_XRAY_TRACING environment variable in its container definition.
```
{
      "name": "envoy",
      "image": "840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy:v1.12.1.1-prod",
      "essential": true,
      "environment": [
        {
          "name": "APPMESH_VIRTUAL_NODE_NAME",
          "value": "mesh/myMesh/virtualNode/myNode"
        },
        {
          "name": "ENABLE_ENVOY_XRAY_TRACING",
          "value": "1"
        }
      ],
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -s http://localhost:9901/server_info | cut -d' ' -f3 | grep -q live"
        ],
        "startPeriod": 10,
        "interval": 5,
        "timeout": 2,
        "retries": 3
      }
```

## Step 8: CloudWatch Synthetics
- Create heartbeat monitoring on a single URL
- Broken Link Checker
- GUI Workflow
- Monitor your API

## Step 9: CloudWatch ServiceLens
Unified access to metrics, logs, traces and canaries
- Service Map: https://ap-southeast-2.console.aws.amazon.com/cloudwatch/home?region=ap-southeast-2#servicelens:map
- Traces: https://ap-southeast-2.console.aws.amazon.com/cloudwatch/home?region=ap-southeast-2#servicelens:traces

## References
- https://aws.amazon.com/blogs/compute/application-tracing-on-kubernetes-with-aws-x-ray/ 
- https://eksworkshop.com/intermediate/245_x-ray/x-ray-daemon/
- https://github.com/aws-samples/reinvent2018-dev303-code
- https://github.com/GoogleCloudPlatform/microservices-demo
- https://github.com/pksinghus/xray-python-k8s
