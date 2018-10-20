DevOps Coding Test
==================

# Goal

Script the creation of a service, and a healthcheck script to verify it is up and responding correctly.


# The Task

You are required to provision and deploy a new service in AWS. It must:

* Be publicly accessible, but *only* on port 80.
* Return the current time on `/now`.  http://ecsalb-544162836.eu-west-1.elb.amazonaws.com/now

# Provision the service.

I have created a simple web application using Express, a fast minimalist web framework for Node.js, Moment.js is a library that validate, manipulate, and display dates and times in JavaScript and then Redirect.js.  then I have build a Docker image for that application, I have run the image as a container locally to test the web page, then uploaded the docker image to ECS AWS Repository ,ECR. Starting from that we then deployed an ECS cluster usign Cloudformation , deploying two instances , in different availability zone, with Amazon ECS-Optimized Amazon Linux AMI in a scaling group with an ALB.



1)Create a docker container with Nodejs:

First, I have created a new directory ,now-project, where all the files stands and created a file package.json ,this file describes the app and its dependencies
With package.json file, I have run npm install ,this will generate a package-lock.json file which will be copied to the Docker image.

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

Then, I have created an index.js file that defines a web app using the Express.js framework:

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

I have created an empty file called Dockerfile then I have defined from what image I want to build from. Here we will use the latest LTS (long term support) version 8 of node available from the Docker Hub:

-Dockerfile

FROM node:8.11.1

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 80
CMD [ "npm", "start" ]
HEALTHCHECK CMD curl --fail http://localhost:80 || exit 1
VOLUME /etc/timezone:/etc/timezone
VOLUME /etc/localtime:/etc/localtime

Building your image
Go to the directory that has your Dockerfile and run the following command to build the Docker image. The -t flag lets you tag your image so it's easier to find later using the docker images command:

$ docker build -t <your username>/node-web-app .
Your image will now be listed by Docker:

$ docker images

# Example
REPOSITORY                      TAG        ID              CREATED
node                            8          1934b0b038d1    5 days ago
<your username>/node-web-app    latest     d64d3505b0d2    1 minute ago
Run the image
Running your image with -d runs the container in detached mode, leaving the container running in the background. The -p flag redirects a public port to a private port inside the container. Run the image you previously built:

$ docker run -p 49160:8080 -d <your username>/node-web-app
Print the output of your app:

# Get container ID
$ docker ps

# Print app output
$ docker logs <container id>

# Example
Running on http://localhost:8080
If you need to go inside the container you can use the exec command:

# Enter the container
$ docker exec -it <container id> /bin/bash
Test
To test your app, get the port of your app that Docker mapped:

$ docker ps

# Example
ID            IMAGE                                COMMAND    ...   PORTS
ecce33b30ebf  <your username>/node-web-app:latest  npm start  ...   49160->8080
In the example above, Docker mapped the 8080 port inside of the container to the port 49160 on your machine.

Now you can call your app using curl (install if needed via: sudo apt-get install curl):

$ curl -i localhost:49160

HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 12
ETag: W/"c-M6tWOb/Y57lesdjQuHeB1P/qTV0"
Date: Mon, 13 Nov 2017 20:53:59 GMT
Connection: keep-alive

Hello world


# Run the healthcheck script
