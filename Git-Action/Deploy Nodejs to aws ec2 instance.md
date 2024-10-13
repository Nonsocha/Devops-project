# Deploying Nodejs To Ec2 Instance
To deploy a web application to an EC2 instance using GitHub Actions, you can automate the deployment process with a workflow that triggers upon each push or on a specific branch. Here’s a step-by-step guide to help you set up the workflow.
### Store Secrets in GitHub
Add the following secrets in your GitHub repository settings:

- EC2_HOST: Public IP or DNS name of the EC2 instance.
- EC2_USER: Username for SSH (e.g., ec2-user or ubuntu).
- EC2_KEY: Private SSH key (the content of your ~/.ssh/id_rsa).
- Go to GitHub repository -> Settings -> Secrets -> Actions -> New repository secret.
### Clone your github repository
- Create a github repository and give it a name .
- Go to the terminal and clone the github repository
- Change directory to working directoryin your github
### Create a Nodejs file
- Create an index.html file
- create a server.js file
- create  a node js file
  ```
  npm init
  ```
  ```
  npm install express
  ```
### Start Nodejs Locally
- To check if node js runs locally
  ```
  npm start
  ```
- If your app runs on a specific port (e.g., 3000), you can verify it by opening your EC2 instance's public IP in the browser:
  ```
  http://<EC2-Instance-Public-IP>:3000
   ```
## Create a Docker file (optional:when you want to ensure that the app runs consistently across different environments, including development, testing, and production.)

#### Use an official Node.js runtime as a parent image
```` 
FROM node:16

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json to install dependencies
COPY package*.json ./

# Install app dependencies
RUN npm install

# Copy the app source code to the container
COPY . .

# Expose the port that the app runs on
EXPOSE 3000

# Define the command to run your app
CMD ["npm", "start"]
```` 
## Build and Run the Docker Container on EC2
- Build the Docker image: Run the following command to build the Docker image on your EC2 instance:
  
   ``
    docker build -t my-app .
   ``

- Run the Docker container: Start the Docker container on port 3000 (or any other available port):
  
   ```
    docker run -d -p 3000:3000 my-app
    ```

### View the App in Your Browser
Now, open your browser and enter the public IP of your EC2 instance with port 3000:
```
http://<your-ec2-public-ip>:3000
```
If your security group is configured correctly and the container is running, you should see your Node.js app's content.

### Create a YML file for the workflow
- create a directory 

  ``
  mkdir -p .github/workflows
  ``
- cd into the directory
- Inside the directory,create a yml file
  
  ```
  touch deploy.yml
  ```

  ```
  vi deploy.yml
  ```
### Steps to Set Up and Deploy:
Configure GitHub Secrets:

Make sure you’ve added the following secrets to your repository:
- EC2_HOST – Public IP of your EC2 instance.
- EC2_USER – EC2 username (e.g., ubuntu).
- DOCKER_HUB_USERNAME-Dockerhub Username
- DOCKER_HUB_PASSWORD-Dockerhub Tokenname
  
2 Push Changes:

   When you push changes to the main branch, GitHub Actions will automatically build, transfer, and deploy the updated Docker container to your EC2 instance.
    View the App:

3 Open your browser and go to ``http://<your-ec2-public-ip>:3000`` to view your app

This workflow automates the process of deploying your app to an EC2 instance whenever code is pushed to the main branch, providing a simple CI/CD pipeline for your Dockerized app.
