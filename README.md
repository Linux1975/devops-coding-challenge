DevOps Coding Test
==================

# Goal

Script the creation of a service, and a healthcheck script to verify it is up and responding correctly.


# The Task

You are required to provision and deploy a new service in AWS. It must:

* Be publicly accessible, but *only* on port 80.
* Return the current time on `/now`.  http://ecsalb-544162836.eu-west-1.elb.amazonaws.com/now

# Provision the service.

I have created a simple web application using Express, a fast minimalist web framework for Node.js,and Moment.js that is a library that validate, manipulate, and display dates and times in JavaScript. I have also used this Library : Redirect.js.  
I have build a Docker image for that application and then run the image as a container locally to test the web page, uploaded the docker image to the ECS AWS Repository ,ECR. 
Starting from that I then deployed an ECS cluster usign Cloudformation , with two instances , in different availability zone, using Amazon ECS-Optimized Amazon Linux AMI in a scaling group with an ALB.


1)Create a docker container with Nodejs:

First, I have created a new directory ,now-project, where all the files stand and created a file package.json ,this file describes the app and its dependencies
With package.json file, I have run npm install ,this will generate a package-lock.json , file which will be copied to the Docker image.

-Package.json

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

-Index.js :

##################################
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
        <p>App started ${STARTUP_TIME.fromNow()}.</p> //It will show current time
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


#####################################

I have created an empty file called Dockerfile then I have defined from what image I want to build from. 

-Dockerfile

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

I have then built the image , run it and test the app 

$ docker build -t now-time .

$ docker images

# Example
REPOSITORY                      TAG        ID              CREATED
node                            8          1934b0b038d1    5 days ago
ow-time    latest     d64d3505b0d2    1 minute ago

$ docker run  -d <your username>/node-web-app
Print the output of your app:

# Get container ID
$ docker ps

# Print app output
$ docker logs <container id>

# Example
Running on http://localhost:9999

# Example
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                  PORTS                NAMES           
ee17dea56f1c        37b5603a3041        "npm start"         44 hours ago        Up 44 hours (healthy)   0.0.0.0:9999->80/tcp   now-container


# Run the healthcheck script
