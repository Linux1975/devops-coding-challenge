DevOps Coding Test
==================

# Goal

Script the creation of a service, and a healthcheck script to verify it is up and responding correctly.


# The Task

You are required to provision and deploy a new service in AWS. It must:

* Be publicly accessible, but *only* on port 80.
* Return the current time on `/now`.  http://ecsalb-544162836.eu-west-1.elb.amazonaws.com/now

# Provision the service.

I have created a simple web application using Express, a fast minimalist web framework for Node.js,and Moment.js a library that validate, manipulate, and display dates and times in JavaScript. I have also used this Library : Redirect.js , to redirect the URL /now to the root.
I have built a Docker image for that application and run the image as a container locally to test the web page, uploaded the docker image to the ECS AWS Repository ,ECR. 
Starting from that I then deployed an ECS cluster usign Cloudformation , with two instances , in different availability zone, using Amazon ECS-Optimized Amazon Linux AMI in a scaling group with an ALB.


1)Create a docker container with Nodejs:

First, I have created a new directory ,now-project, where all the files stand and created a file package.json ,this file describes the app and its dependencies
With package.json file, I have run npm install ,this will generate a package-lock.json , file which will be copied to the Docker image.

-***Package.json***
```
{
  "name": "Now",
  "version": "1.0.0",
  "description": "Web service to print current time",
  "main": "now",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "express": "^4.16.3",
    "express-simple-redirect": "^1.0.1",
    "moment": "^2.22.0"
  }
}

Then, I have created an index.js file that defines a web app using the Express.js framework.
```
-**Index.js**

##################################
```

const express = require('express');
const moment = require('moment');
const redirect = require('express-simple-redirect');

const STARTUP_TIME = moment();
const PORT = 80;
const { version } = require('./package.json');


const app = express();
app.get('/', (req, res) => {
    res.send(`<!DOCTYPE html>\n
        <html>\n
        <head><title>Project Now</title></head>\n
        <body>\n
        <p>Remember the time is always now :)</p>
        <p>App version ${version}</p>
        <p>App started ${STARTUP_TIME.fromNow()}.</p>              
	</head>
	<p id="now"></p>
        <script>
	var localTime = new Date();
	document.getElementById("now").innerHTML = localTime;
	</script>
	</body>
	</html>

   `);
});
app.get('/health', (req, res) => {
    res.json({ healthy: true });
});

app.use(redirect({
  '/now': '/'                 // Internal redirect to /now page
}));

app.listen(PORT);
console.log(`Running on port: ${PORT}`);

```
#####################################

I have created an empty file called Dockerfile then I have defined from what image I want to build from. 

-**Dockerfile**
```
FROM node:8.11.1

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 80
CMD [ "npm", "start" ]
HEALTHCHECK CMD curl --fail http://localhost:80 || exit 1  #With this command we can check if our container is healthy or not 
VOLUME /etc/timezone:/etc/timezone                         #With this command we can syncronize time between the host and the container
VOLUME /etc/localtime:/etc/localtime

```
I have then built the image , run it and test the app 

$ docker build -t now-time:latest .

$ docker images

# Example0
```
docker images                            
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
now-time            1.0.0               be0392df8aa3        31 hours ago        683MB
now-time            latest              be0392df8aa3        31 hours ago        683MB
<none>              <none>              37b5603a3041        44 hours ago        683MB
node                8.11.1              78f8aef50581        5 months ago        673MB
```

$ docker run  -d <your node-web-app
Print the output of your app:

# Get container ID

$ docker ps

# Print app output

$ docker logs <container id>

# Example

Running on http://localhost:9999

# Example

'''
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                  PORTS                NAMES           
ee17dea56f1c        37b5603a3041        "npm start"         44 hours ago        Up 44 hours (healthy)   0.0.0.0:9999->80/tcp   now-container
...

I have pushed the image just created and uploaded to the ECR repository.

2)Create an ECS Cluster with Cloudformation (please refer to the templatein my aws account )

I have used a CloudFormation stack to launch all the requisite AWS resources: two instances , in different availability zones, using Amazon ECS-Optimized Amazon Linux AMI using a scaling group with an ALB.
In the Userdata part the healtcheck.sh script will be downloaded from my GitHub account changing the permisssion (chmod u+x script)
I have vreated the stack in a terminal and run the AWS CLI command below to create :
```
aws cloudformation deploy \
    --stack-name Now\
    --template-file ./cloudformation/<template>.yml \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides KeyName='<keypair_id>' \
    VpcId='<vpc_id>' \
    SubnetId='<subnet_id_1>,<subnet_id_2>' \
    ContainerPort=80 \
    DesiredCapacity=2 \
    EcsImageUri='<ecr_image_uri>' \
    EcsImageVersion='<app_version>' \
    InstanceType=t2.micro \
    MaxSize=2
```
# Run the healthcheck script

You can run the healtcheck.sh externally using using this script:
```
#!/bin/bash


aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].PublicIpAddress' --output text > ip-instance-1.txt
aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].PublicIpAddress' --output text > ip-instance-2.txt
echo "####-Instance number 1-####"
ssh -i "Key.pem" ec2-user@$(cat ip-instance-1.txt ) sudo su -c /root/healthcheck.sh
echo "####-Instance number 2-####"
ssh -i "Key.pem" ec2-user@$(cat ip-instance-2.txt ) sudo su -c /root/healthcheck.sh
```
