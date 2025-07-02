# Wallarm Solutions Engineer Technical Evaluation Solution

## Overview Summary

## High Level Diagram

## Setting Up a Test Backend

1. First I spun up an AWS EC2 instance with Amazon Linux, and set it up with docker.
    ```bash
    sudo yum update -y
    sudo yum install -y docker
    sudo usermod -aG docker $USER
    newgrp docker
    ```

2. Installed Docker and ran a simple HTTP service using httpbin:
    ```bash
    docker run -p 80:80 kennethreitz/httpbin
    ```
3. I verified that httpbin was running by accessing it via the public IP of the EC2 instance.
    ![Screenshot of httpbin running on an EC2 instance](./images/1-httpbin.png "httpbin running on EC2 instance")

## Setting up a Wallarm Filtering Node

1. Follow the instructions for the [documentation](https://docs.wallarm.com/installation/cloud-platforms/aws/docker-container/)
2. Deploy an ECS Cluster on AWS called "wallarm-cluster"
3. Created the secret parameter in the AWS secret manager.
4. Created a task definition JSON following the instructions in the documentation.
5. AWS is not happy, have a conversation with "Amazon Q" about it.  
    1. Amazon Q is just making up menus that don't exist. The issue is a ecsTaskExecutionRole not existing.
    2. Created the new role by digging around the AWS menus
6. Task launched on the cluster.
7. Task failed, that ecs role didn't have permissions.
    1. Added the permissions to the ecsTaskExecutionRole.
    2. IAM > Roles > ecsTaskExecutionRole > Permissions > Add permissions > Create inline policy > JSON
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameters"
            ],
            "Resource": [
                "arn:aws:ssm:us-east-2:788180922105:parameter/wallarm-node-token"
            ]
        }
    ]
    }
    ```
8. Task launched successfully.
    ![Screenshot of AWS showing the task launched](/images/2-task-running.png "Task Launched")
9. Wallarm console updates and sees the node.
    ![Screenshot of wallarm console](/images/3-wallarm-sees-it.png "Wallarm Node Running")
10. Verify filtering by using `curl http://<ip>/etc/passwd` 
    1. This doesn't work. It sends me down a rabbit hole of trying to figure out why I can't connect.
    2. Public IP address doesn't work. EC2 doesn't work, and Fargate doesn't work.
    3. Can't ping the public IP address either.
    4. I created a google compute instance, adapted the script from the documentation to run SSH'd into the google compute instance.
        ```
        sudo docker run -d \
          -e WALLARM_API_TOKEN=[TOKEN REDACTED] \
          -e WALLARM_API_HOST=us1.api.wallarm.com \
          -e NGINX_BACKEND=ec2-18-116-67-13.us-east-2.compute.amazonaws.com \
          -p 80:80 \
          registry-1.docker.io/wallarm/node:6.2.0
        ```
    5. Problem found. Amazon removed the server that was running httpbin... Didn't notice that until I switched back to the terminal running the httpbin.
        ```
        Broadcast message from root@localhost (Wed 2025-07-02 21:11:27 UTC):
        The system will power off now!
        Connection to ec2-3-17-186-36.us-east-2.compute.amazonaws.com closed by remote host.
        Connection to ec2-3-17-186-36.us-east-2.compute.amazonaws.com closed.
        ```
    6. Spin up a EC2. It works now. I'm using AWS for the API, and Google Compute for the Wallarm node. AWS just doesn't want to play nice today.
        ![Screenshot of wallarm console](/images/4-google-sees-it.png "Wallarm Node Running")
        ![Screenshot of Firefox showing access through node](/images/5-accessed-through-node.png "Wallarm Node Accessed API")
11. Verify filtering by using `curl http://35.224.102.117/etc/passwd` 
    ```
    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
    <title>404 Not Found</title>
    <h1>Not Found</h1>
    <p>The requested URL was not found on the server.  If you entered the URL manually please check your spelling and try again.</p>
    ```
    ![Screenshot of wallarm showing attack](/images/6-etc-passwd-attack.png "Wallarm Shows curl etc/passwd Attack")

## Generate Traffic Using GoTestWAF

1. Pull the GoTestWAF repo:
    ```bash
    docker pull wallarm/gotestwaf
    ```
2. Run the GoTestWAF container:
    ```bash
    docker run --rm --network="host" -it -v ${PWD}/reports:/app/reports \
    wallarm/gotestwaf --url=http://35.224.102.117
    ```

3. Observe that wallarm is detecting the attacks in monitor mode.
    ![Screenshot of wallarm console showing GoTestWAF attacks](/images/7-gotestwaf-attacks.png "Wallarm Console with GoTestWAF Attacks")


## Checklist

- [x] Deploy a Wallarm filtering node using a supported method of your choice.  
    - [x] Docker and Google Compute
    - [x] Verify that the filtering node is properly deployed and running.  

- [x] Configure a backend origin to receive test traffic. (httpbin.org is also acceptable)  
    - [x] Ran httpbin on an AWS EC2 instance.
    - [x] Contactable from the filtering node.

- [x] Use the **GoTestWAF** attack simulation tool to generate traffic.  
- [x] Document the deployment and troubleshooting process.  
- [x] Demonstrate proficiency in using **Wallarm's official documentation**.
- [x] Fork Repository
- [ ] Overview Summary
- [ ] High Level Diagram
