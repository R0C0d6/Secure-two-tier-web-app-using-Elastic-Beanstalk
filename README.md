# Week 2 of my 12-Week AWS Hands-On Challenge

A secure, production-style Multi-Tier App using Elastic Beanstalk and learned a ton. This is the full story, technical walkthrough, and lessons.

---

## What I built

A secure two-tier web application where the frontend is internet-facing (public subnet) and the backend API is internal/private (private subnets), both deployed and managed with AWS Elastic Beanstalk. The frontend is a small Flask app with a neat UI; the backend is a Flask API. The frontend talks to the backend over internal networking only. The backend is unreachable from the internet.

---

## Why this project?

I wanted to go beyond “launch a single EC2 and call it a day.” In modern cloud apps you usually need:

 Resiliency (multiple AZs)
 Security (backend private, least privilege)
 Manageability ( BeanstalkUI, scaling)
 Repeatability (infrastructure as small, scriptable steps)

Elastic Beanstalk is great for quick prototyping, but you can, and should, configure it properly to follow secure best practices: Frontend in public subnets, private instances behind it, NAT for outbound package downloads, and tightly scoped security groups that only allow the traffic you actually need.

---

## Architecture Overview (simple diagram)

```
                     Internet
                        ↓
             Public ALB (Frontend) — public subnets (2 AZs)
                        ↓
            Frontend EC2 instances (private subnets, no public IP)
                        ↓  (HTTP)
              Internal ALB (Backend) — private subnets (2 AZs)
                        ↓
             Backend EC2 instances (private subnets, no public IP)
                        ↓
                      NAT Gateway
                        ↓
                   Internet (outbound only)
```

Key security points:

 Backend is internal (not reachable from the public internet).
 Backend instances have no public IPs.
 Frontend instances have no public IPs either; ALB is internet-facing and routes traffic to them.
 Security groups reference each other (allow exact SG → SG rules) so only intended components can talk.

---

## What you’ll find in this post

1. Exact project structure and code you can reuse
2. Step-by-step AWS console actions to build a secure VPC (public/private subnets, IGW, NAT, route tables)
3. IAM roles needed for Elastic Beanstalk (service role + EC2 instance profile)
4. Security group rules and wiring 
5. How to prepare zip packages for Elastic Beanstalk (folder structure pitfalls)
6. How to deploy backend (internal environment) and frontend (public environment)
7. How to verify, test.
8. What I learned and next steps

---

## Initial Configuration
Configure your AWS Environment and..... 

Make two folders: e.g. `week2-backend/` and `week2-frontend/`.

Create Application file(application.py), Requirements file(requirements.txt) and Procfile(procfile).
 application.py → your main web app file.
 Defines the Flask app logic.
 Must contain a variable called application (e.g., application = Flask(__name__)) because Elastic Beanstalk looks for it to start your app.

 requirements.txt → lists all the Python packages your app needs (like Flask, requests, etc.).
 Elastic Beanstalk installs everything in this file automatically when deploying.

 Procfile → tells Elastic Beanstalk how to run your app.
 Example: web: gunicorn application:application
 This runs your app in production mode using Gunicorn.

![Initial configuration and Setup](https://i.postimg.cc/GtTt0zCb/Screenshot-2025-10-23-152753.png)


### Backend: `week2-backend/application.py`

```python
from flask import Flask, jsonify

application = Flask(__name__)

@application.route('/')
def home():
    return jsonify({"message": "Hello from the Backend API!"})

@application.route('/data')
def data():
    return jsonify({
        "users": ["12WKChallenge", "Roland", "Paula Wakabi"],
        "message": "Backend is running as it should, Paula"
    })

if __name__ == '__main__':
    application.run(host='0.0.0.0', port=8080)  # IMPORTANT: EB listens on port 8080
```

### Backend: `week2-backend/requirements.txt` (Requirements.txt)
```
Flask==3.0.0
 flask-cors==4.0.0
 gunicorn==21.2.0
```

### Backend: `week2-backend/procfile` (Procfile)
web: gunicorn application:application


...


### Frontend: `week2-frontend/application.py`

```python
from flask import Flask, render_template
import requests

application = Flask(__name__)

# Backend API URL (your Beanstalk backend)
BACKEND_URL = "http://roawschallenge.us-east-1.elasticbeanstalk.com/"

@application.route('/')
def home():
    try:
        response = requests.get(BACKEND_URL)
        data = response.json()
    except Exception as e:
        data = {"error": str(e), "message": "Eka aa aba no. We failed to fetch backend data o"}

    return render_template('index.html', data=data)

if __name__ == '__main__':
    application.run(host='0.0.0.0', port=8080)
```

### Frontend: `week2-frontend/requirements.txt` (Requirements.txt)

```
Flask==3.0.2
gunicorn==21.2.0
requests==2.32.3
```

### Frontend: `week2-frontend/templates/index.html` (index.html)
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>AWS Challenge Frontend</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      margin-top: 50px;
    }
    .card {
      display: inline-block;
      padding: 20px;
      border: 1px solid #ccc;
      border-radius: 10px;
      box-shadow: 2px 2px 8px rgba(0,0,0,0.1);
    }
    h1 { color: #0073bb; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Frontend Connected Successfully ooooo, wonim effort</h1>
    <p><b>Message from Backend:</b> {{ data.message }}</p>
    <p><b>Users:</b> {{ data.users }}</p>
  </div>
</body>
</html>
```

### Frontend: `week2-frontend/procfile` (procfile)
web: gunicorn application:application

### Test the code locally (I tested on port 5000. Port 80 was in use on my local machine)
![Test Code locally](https://i.postimg.cc/gJwHzNx7/Screenshot-2025-10-24-184258.png)
![Test Code locally](https://i.postimg.cc/R0hcdXLC/Screenshot-2025-10-24-184321.png)


> Important: Name the file `application.py` and ensure the WSGI callable is named `application`. Elastic Beanstalk looks for that by default. 

---

## VPC & Networking

Create a VPC with 2 public and 2 private subnets in two AZs, attach an IGW, create a NAT Gateway, and route tables.


1. Create VPC: `10.0.0.0/16`, name it e.g.`week2-secure-vpc`.
   ![VPC created](https://i.postimg.cc/15FNnRqt/Screenshot-2025-10-23-160733.png)
   
3. Create Public Subnet 1: `10.0.1.0/24`.
   ![Public Subnet](https://i.postimg.cc/KjfB9xnL/Screenshot-2025-10-23-161425.png)
   ![Public Subnet](https://i.postimg.cc/PJVNpxW4/Screenshot-2025-10-23-161559.png)
   ![Public Subnet](https://i.postimg.cc/Wzz1vvBJ/Screenshot-2025-10-23-161622.png)
   
4. Create Private Subnet 1: `10.0.11.0/24`.
   ![Private Subnet Created](https://i.postimg.cc/FzXg1Txz/Screenshot-2025-10-23-161911.png)
5. Create Internet Gateway (IGW) and attach to VPC.
   ![IGW Created](https://i.postimg.cc/2yxcYpfx/Screenshot-2025-10-23-162649.png)
   ![IGW Attached](https://i.postimg.cc/x1PzP03b/Screenshot-2025-10-23-162732.png)
   
6. Create a Public Route Table, add route `0.0.0.0/0` -> IGW, associate with public subnets.
   ![PubRT](https://i.postimg.cc/hvgJm6Qh/Screenshot-2025-10-23-163111.png)
   ![PubRT](https://i.postimg.cc/x1kdXnFS/Screenshot-2025-10-23-163418.png)
   ![PubRT](https://i.postimg.cc/CxfFFJzZ/Screenshot-2025-10-23-163534.png)
   ![PubRT](https://i.postimg.cc/658K0zVd/Screenshot-2025-10-23-163548.png)
8. Allocate an Elastic IP and create a NAT Gateway in a public subnet (for private instances to download packages).
   ![NGW](https://i.postimg.cc/sgwDH1QQ/Screenshot-2025-10-23-164328.png)
10. Create a Private Route Table, add route `0.0.0.0/0` -> NAT Gateway, associate with private subnets.
![PrivRT](https://i.postimg.cc/kGrHvF7Z/Screenshot-2025-10-23-163701.png)
![PrivRT](https://i.postimg.cc/q7zZfZRh/Screenshot-2025-10-23-163722.png)
![PrivRT](https://i.postimg.cc/MGMt3MPm/Screenshot-2025-10-23-164514.png)
Confirm: public subnets send internet to IGW; private subnets send internet to NAT.

---

## IAM: Roles for Elastic Beanstalk

Elastic Beanstalk needs:

Service role (Elastic Beanstalk to perform management tasks). 
![SVCRole](https://i.postimg.cc/cH6hjDx0/Screenshot-2025-10-23-154332.png)
![SVCRole](https://i.postimg.cc/HnXwDRfk/Screenshot-2025-10-23-154359.png)

EC2 instance profile (role for the EC2 instances EB launches). 
![SVCRole](https://i.postimg.cc/KzvZPbvw/Screenshot-2025-10-23-155554.png)

Create them in IAM → Roles → Create role → choose AWS service (Elastic Beanstalk / EC2) and attach the policies.

---

## Security Groups

Create two SGs in the VPC:

Name them as you prefer: e.g. sg-frontend-ec2 (frontend instances)

   Inbound: HTTP 80 
   Inbound (optional): SSH 22 from your IP
   Outbound: Allow HTTP to backend (or all outbound)
   ![PublicSG](https://i.postimg.cc/85YMxNcg/Screenshot-2025-10-23-165359.png)

Name them as you prefer: e.g. sg-backend-ec2 (backend instances)

   Inbound: HTTP 80 from frontend
   Inbound (optional): SSH 22 from your IP (for debugging)
   Outbound: All
![PrivateSG](https://i.postimg.cc/Twx3wssh/Screenshot-2025-10-23-165830.png)

---


## Packaging for Elastic Beanstalk (zip the backend)

Important: When you create the zip file to upload to Elastic Beanstalk, zip the contents of the app folder, not the folder itself. This is one of the issues i faced initially😅

Correct:

```
cd week2-backend
zip -r ../backend.zip .
```

Wrong:

```
zip -r backend.zip week2-backend
# This creates week2-backend/application.py inside zip and EB won't find application.py at the top level
```

Make sure `application.py` and `requirements.txt` are at the top level inside the zip.

---

## Deploy the Backend (internal Elastic Beanstalk environment)

In Elastic Beanstalk console:

1. Create Application → Upload your `backend.zip` file.
2. Configure (more options) → Network:
    VPC: Select your VPC
    Instance subnet: priv-subnet-1.
3. Security groups: assign your private SG e.g.`sg-backend` for instances
4. Use Basic monitoring.
5. Managed Updates were turned off. Rolling updates were disabled. Not required due to the type of application
6. Deployment Policy: All at once. Batch Size type: Percentage
7. IAM roles: `aws-elasticbeanstalk-service-role` + `aws-elasticbeanstalk-ec2-role`
8. Create environment → Wait for Green.

   ### STEP 1
   Select Web Server Environment. Give it a Name.
![Backend Beanstalk Setup](https://i.postimg.cc/N07GScsZ/Screenshot-2025-10-23-153311.png)

Platform: Select Python. 
PLatform Branch: Python 3.9 or equivalent
![Backend Beanstalk Setup](https://i.postimg.cc/Hs2fqkX4/Screenshot-2025-10-23-153322.png)

Application Code: Select Upload your code. Upload your zipped file. 
Version Label. Provide a version identifier.
Single Instance: Free tier Deployment to avoid unintended cost , or interruptions in the case of spot.
![Backend Beanstalk Setup](https://i.postimg.cc/kGypPT6K/Screenshot-2025-10-23-153331.png)

### STEP 2
Next, Configure Service Access by Selecting the Service Role and the EC instance Profile created earlier
Provide a Key Pair. Create one if you don't have any (Choose RSA and .pem when creating it)
![Backend Beanstalk Setup](https://i.postimg.cc/brL3BdWN/Screenshot-2025-10-23-155653.png)


### STEP 3
Select the VPC you Created and the Private Subnet.
Leave the assign public IP unchecked because we are deploying our backend in Private subnets
![Backend Beanstalk Setup](https://i.postimg.cc/Kjc5Bxjb/Screenshot-2025-10-23-171302.png)


### STEP 4 
Choose the Instance Type: General Purpose
Monitoring Interval: Leave as default
![Backend Beanstalk Setup](https://i.postimg.cc/28M4844J/Screenshot-2025-10-23-172252.png)

CHoose your backend security group
Select a single instance and use an on demand instance to prevent interruptions with spot instance deployments
![Backend Beanstalk Setup](https://i.postimg.cc/wxSX9Xcx/Screenshot-2025-10-23-172315.png)
Leave all other subsequent settings as default

Use Basic monitoring.
![Backend Beanstalk Setup](https://i.postimg.cc/wvMY6y80/Screenshot-2025-10-24-221441.png)

Managed Updates were turned off. Rolling updates were disabled. Not required due to the type of application
![Backend Beanstalk Setup](https://i.postimg.cc/GpH1RT1N/Screenshot-2025-10-24-221517.png)
![Backend Beanstalk Setup](https://i.postimg.cc/vBF2K48V/Screenshot-2025-10-24-221534.png)

Deployment Policy: All at once. Batch Size type: Percentage
![Backend Beanstalk Setup](https://i.postimg.cc/vBF2K48V/Screenshot-2025-10-24-221534.png)

Enable Health Checks. Health threshold(OK). Set timeout (e.g. 600 seconds) Use Nginx proxy.
![Backend Beanstalk Setup](https://i.postimg.cc/4N2gSQDv/Screenshot-2025-10-24-221549.png)

Ensure X-Ray Daemon, Log rotation and log streaming are all turned off to avoid extra charges
![Backend Beanstalk Setup](https://i.postimg.cc/x1gtR6Bc/Screenshot-2025-10-24-221604.png)

Wait for it to complete deployment
![Backend Beanstalk Setup](https://i.postimg.cc/kXvBLKTw/Screenshot-2025-10-23-172644.png)

You will get an internal Domain (e.g., `internal-backend-xxxx.amazonaws.com`). Note it down. this is the backend URL used by the frontend.

---



## Deploy the Frontend (public EB environment)

1. Environment properties in the frontend code (frontend application.py), set the BACKEND_URL value = `http://<the backend Url you copied>/data`
2. Zip the Frontend files
3. Select Create Application → Upload `frontend.zip`.
4. Configure (more options) → Network:
    Select your VPC
    Instance subnet: priv-subnet-1. instances have public IP
5. Security groups: assign your public SG e.g.`sg-frontend` for instances. Security groups: assign  `sg-frontend-ec2` (instances)
6. IAM roles: same as above
7. Create environment → Wait for Green success
Leave all other settings same as backend

   ### STEP 1
   Select Web Server Environment. Give it a Name.
![Frontend Beanstalk Setup](https://i.postimg.cc/0yyM1nXB/Screenshot-2025-10-24-220525.png)

Platform: Select Python. 
PLatform Branch: Python 3.9 or equivalent
![Frontend Beanstalk Setup](https://i.postimg.cc/Hs2fqkX4/Screenshot-2025-10-23-153322.png)

Application Code: Select Upload your code. Upload your zipped file. 
Version Label. Provide a version identifier.
Single Instance: Free tier Deployment to avoid unintended cost , or interruptions in the case of spot.
![Frontend Beanstalk Setup](https://i.postimg.cc/VsGfxwMj/Screenshot-2025-10-24-230856.png)

### STEP 2
Next, Configure Service Access by Selecting the Service Role and the EC instance Profile created earlier
Provide a Key Pair.
![Frontend Beanstalk Setup](https://i.postimg.cc/brL3BdWN/Screenshot-2025-10-23-155653.png)


### STEP 3
Select the VPC you Created and the Public Subnet.
Assign public IP :check it because we are deploying our frontend in Public subnets and would like to access it via the internet
![Frontend Beanstalk Setup](https://i.postimg.cc/pXxkzNyj/Screenshot-2025-10-24-220921.png)


### STEP 4 
Choose the Instance Type: General Purpose
Monitoring Interval: Leave as default
![Frontend Beanstalk Setup](https://i.postimg.cc/28M4844J/Screenshot-2025-10-23-172252.png)

CHoose your Frontend security group
Select a single instance and use an on demand instance to prevent interruptions with spot instance deployments
![Frontend Beanstalk Setup](https://i.postimg.cc/T1Z6NRSf/Screenshot-2025-10-24-221232.png)

Use Basic monitoring.
![Frontend Beanstalk Setup](https://i.postimg.cc/wvMY6y80/Screenshot-2025-10-24-221441.png)

Managed Updates were turned off. Rolling updates were disabled. Not required due to the type of application
![Frontend Beanstalk Setup](https://i.postimg.cc/GpH1RT1N/Screenshot-2025-10-24-221517.png)
![Frontend Beanstalk Setup](https://i.postimg.cc/vBF2K48V/Screenshot-2025-10-24-221534.png)

Deployment Policy: All at once. Batch Size type: Percentage
![Frontend Beanstalk Setup](https://i.postimg.cc/vBF2K48V/Screenshot-2025-10-24-221534.png)

Enable Health Checks. Health threshold(OK). Set timeout (e.g. 600 seconds) Use Nginx proxy.
![Frontend Beanstalk Setup](https://i.postimg.cc/4N2gSQDv/Screenshot-2025-10-24-221549.png)

Ensure X-Ray Daemon, Log rotation and log streaming are all turned off to avoid extra charges
![Frontend Beanstalk Setup](https://i.postimg.cc/x1gtR6Bc/Screenshot-2025-10-24-221604.png)

Leave any other settings as default

Wait for it to complete deployment
![Frontend Beanstalk Setup](https://i.postimg.cc/13WbtRmj/Screenshot-2025-10-24-231331.png)

Copy the frontend URL (public). This, when accessed, should show the styled UI and backend data.

---

## Testing & verification

Event checks: All deployment events completed successfully.
![Testing & verification](https://i.postimg.cc/V6YwTfFF/Screenshot-2025-10-24-231945.png)

Frontend works: Open the public URL and confirm the UI displays backend data.
![Testing & verification](https://i.postimg.cc/XYCQM9HZ/Screenshot-2025-10-24-231347.png)

---

## Common errors i came across and how i fixed them

1. Deployment failed: ELastic beanstalk log shows ModuleNotFoundError:
   Ensure `requirements.txt` lists all dependencies and is at top level.

2. `application.py` not found / EB cannot start app:
   Make sure you zipped app contents, not the folder itself, and the file is named `application.py` with WSGI callable `application = Flask(__name__)`.

3. Health checks failing:
   Make sure app listens on the correct port (EB binds application to port EB expects), and health check path `/` returns 200 quickly.

4. Frontend cannot talk to backend:
   Check SG rules and that the `BACKEND_URL` points to the correct internal ALB DNS.

---

## Cost & performance notes

NAT Gateway and Elastic IP are charged hourly, so remember to release them during testing if not needed.
t2/t3.micro instances are cost-effective for testing; for production choose t3.small+ and enable auto-scaling.
Elastic Beanstalk simplifies operations but hides finer controls; for full control consider ECS/EKS or EC2 Autoscaling groups later.

---


## Lessons learned (personal notes of mine that might help others as well)

Elastic Beanstalk is fantastic for quick, managed deployments, but it requires mindful networking/configuration.
Always test DNS and reachability locally and from inside the VPC.
Security groups referencing other SGs are a simple and powerful way to enforce least privilege networking.
Zip structure and naming are tiny things that often break deployments, so fix them early.

---
