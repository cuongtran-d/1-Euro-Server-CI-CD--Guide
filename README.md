## 1€ Server Guide
Guide to create a server, most fitted for learning reasons.
This Guide will show you how to setup a CI/CD pipeline with github and a virtual server. 
We'll focus on deploying a Dockerized application, with a specific emphasis on a Node.js app. While the concepts we cover are applicable across various technologies, we've chosen Node.js for its popularity and suitability for CI/CD pipelines.

**Requirements**
 - Github repository with an app
	 - App should Dockerized / Dockerfile should be available
 - Ionos vServer (https://www.ionos.de/server/vps)
	 - VPS Linux XS (1€)
 - Ionos domain
	 - any domain provider will do
 - Docker Hub Account


## 1. Sign up at Ionos & Register Domain
 **Register Server and Domain:** Sign in to your Ionos account and register your server and domain. Follow the on-screen instructions provided by Ionos to complete the registration process. Once completed, you'll need to wait for the server to boot up, which may take a few minutes.
    
**Configure A-Record:** Once your server is up and running, you'll receive an IP address. This IP address is essential for directing traffic to your server. To associate your domain with your server, you'll need to create an A-Record (Address Record) in the DNS settings of your domain registrar (in this case, Ionos). This record maps your domain name to the IP address of your server, ensuring that visitors can reach your website by typing in your domain name.

## 2. Setup Server

**SSH to your server. (On Windows use powershell)**

    ssh root@YOUR_SERVER_IP

**After login update system**

    apt update && apt upgrade -y

**Setting up a User / Some security stuff** 
*(I don't know if we actually have to do this. IMO if you keep your root password safe it will save you a lot of permission bullshittery later on 🤷‍♀️)*

**Set a good password. Use Default for the rest.**

    adduser YOUR_USERNAME
    adduser YOUR_USERNAME sudo
    su YOUR_USERNAME

**Disable ssh access for root account** 

    sudo nano /etc/ssh/sshd_config

**search for  this and switch it to** *no*

    PermitRootLogin yes
**exit nano with crtl + x and Enter**

**restart sshd**

    sudo systemctl restart sshd
    sudo reboot

**Setup firewall**

    sudo ufw enable
    sudo ufw allow ssh
    sudo ufw allow http
    sudo ufw allow https
 **Setup Docker** 
 Follow this
 https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

## 3. Setup Github Runner
This tool gives us the oppetunity to execute commands on our server with github actions. Thats more or less dangerous, so handle with care. This should only be used on private repositories.

**Check system architecture with** 
```
dpkg --print-architecture
```
**Navigate to your github repository.** 

Settings -> Runners -> Add New Runner

Select image + Architecture and follow thru the instructions

Follow this instructions to make the runner start on boot
https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/configuring-the-self-hosted-runner-application-as-a-service

If everything went well, you should see your runner idling in the runners tab.

## 4. Github Action
*Initially, I attempted to execute the build process directly on the same server where the application would ultimately run. However, I encountered an issue during the `npm install` step, which resulted in the process freezing. I think the server _lacked the necessary capacity to handle the build process. So I went to a two-Step approach.*

**Step 1 - Build & Push** 
Build and Push the docker image to Docker HUB using classic github actions.
**Step 2 - Pull & Run**
Pull the Image from Step 1 and run it on our own server.

Add the `deploy.yml` to `.github/workflows/`

**Update the env Variables**

    DOCKER_HUB_REPO: Your Docker Hub Username / Your Docker Hub Repo Name  
    DOCKER_CONTAINER_NAME: Name for Docker Container # any name will do

****Update the portmapping if your app is not running on 3000.***

    run: docker run -d --name ${{ env.DOCKER_CONTAINER_NAME }} -p SYSTEM_PORT:YOUR_APP_PORT ${{ env.DOCKER_HUB_REPO }}:${{ env.IMAGE_TAG }}

**Run the Workflow** 
github -> actions 
If everything went well, your app should be running. Your can verify it with. 


    docker ps

Our Application is now running on port 3000. 
*If you dont need HTTPS you can change the portmapping in deploy.yml to 80:3000.*
## 5. Nginx & HTTPS
 To enable tls we use let's encrypt / certbot and nginx as reverse proxy.


**Install Nginx**
```
sudo apt install nginx
sudo nano /etc/nginx/sites-available/default
```
Add/Update this to the file:
```
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000; #whatever port your app runs on
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
```
save & exit with ctrg + x and enter

```
# Check NGINX config
sudo nginx -t

# Restart NGINX
sudo service nginx restart
```

Your App should be now available with no port (port 80)

**HTTPS/TLS**
Follow thru each command and do what the terminal says.
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Only valid for 90 days, test the renewal process with
certbot renew --dry-run  
```
Note: certbot renew --dry-run throws an error, because of permisson issues. For now I didn't fixed it.

## 6. Update the workflow file

 Update the `deploy.yml` file  to match your needs. 
 Example: 

     on:  
	    push:  
		    branches: [ develop, master ]  
	    workflow_dispatch:
This will run the pipeline on every push to develop/or master.

## Resources

- Ubuntu Server Setup https://www.youtube.com/watch?v=VXSgEvZKp-8
- NGINX, SSL With Lets Encrypt https://www.youtube.com/watch?v=oykl1Ih9pMg
- Github Runners https://docs.github.com/en/actions/hosting-your-own-runners
