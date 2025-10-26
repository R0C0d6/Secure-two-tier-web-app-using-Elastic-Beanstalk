# Week 2 of my 12-Week AWS Hands-On Challenge (shoutout to Paula Wakabi)

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

Elastic Beanstalk is great for beginners and for quick prototyping, but you can, and should, configure it properly to follow secure best practices: Frontend in public subnets, private instances behind it, NAT for outbound package downloads, and tightly scoped security groups that only allow the traffic you actually need.

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

## Code 

Make two folders: e.g. `week2-backend/` and `week2-frontend/`.

### Backend: `week2-backend/application.py`

```python
from flask import Flask, jsonify

# Elastic Beanstalk expects the WSGI callable to be named `application`
application = Flask(__name__)

@application.route('/')
def home():
    return jsonify({"message": "Hello from Backend (private)!"})

@application.route('/data')
def data():
    return jsonify({
        "users": ["Roland", "Habiba"],
        "message": "Backend (private) responds"
    })
```

`week2-backend/requirements.txt`

```
Flask==2.2.5
```

### Frontend: `week2-frontend/application.py`

```python
from flask import Flask, render_template
import os, requests

application = Flask(__name__)

# This env var will be set in the frontend EB environment
BACKEND_URL = os.environ.get("BACKEND_URL", "http://127.0.0.1:5000/data")

@application.route('/')
def home():
    try:
        r = requests.get(BACKEND_URL, timeout=4)
        data = r.json()
        users = data.get('users', [])
        message = data.get('message', '')
    except Exception as e:
        users = []
        message = f"Error contacting backend: {e}"

    return render_template('index.html', users=users, message=message)
```

`week2-frontend/requirements.txt`

```
Flask==2.2.5
requests==2.31.0
```

`week2-frontend/templates/index.html`




> Important: Name the file `application.py` and ensure the WSGI callable is named `application`. Elastic Beanstalk looks for that by default. Also put `requirements.txt` at the top level of the zip.

---

## VPC & Networking

Create a VPC with 2 public and 2 private subnets in two AZs, attach an IGW, create a NAT Gateway, and route tables.


1. Create VPC: `10.0.0.0/16`, name `week2-secure-vpc`.
2. Create Public Subnet 1: `10.0.1.0/24` (AZ A), Public Subnet 2 — `10.0.2.0/24` (AZ B).
3. Create Private Subnet 1: `10.0.11.0/24` (AZ A), Private Subnet 2 — `10.0.12.0/24` (AZ B).
4. Create Internet Gateway (IGW) and attach to VPC.
5. Create a Public Route Table, add route `0.0.0.0/0` -> IGW, associate with public subnets.
6. Allocate an Elastic IP and create a NAT Gateway in a public subnet (for private instances to download packages).
7. Create a Private Route Table, add route `0.0.0.0/0` -> NAT Gateway, associate with private subnets.

Confirm: public subnets send internet to IGW; private subnets send internet to NAT.

---

## IAM: Roles for Elastic Beanstalk

Elastic Beanstalk needs:

Service role (Elastic Beanstalk to perform management tasks). Attach `AWSElasticBeanstalkServiceRolePolicy`.
EC2 instance profile (role for the EC2 instances EB launches). Attach `AWSElasticBeanstalkWebTier` (and relevant policies like S3 access if needed), and ensure the instance profile ARN exists.

Create them in IAM → Roles → Create role → choose AWS service (Elastic Beanstalk / EC2) and attach the policies. Name them:

 `aws-elasticbeanstalk-service-role`
 `aws-elasticbeanstalk-ec2-role`

---

## Security Groups

Create four SGs in the VPC:

sg-frontend-alb (internet ALB)

Inbound: HTTP 80 from `0.0.0.0/0`
Outbound: All

sg-frontend-ec2 (frontend instances)

   Inbound: HTTP 80 from `sg-frontend-alb`
   Inbound (optional): SSH 22 from your IP
   Outbound: Allow HTTP to `sg-backend-alb` (or all outbound)

sg-backend-alb (internal ALB)

   Inbound: HTTP 80 from `sg-frontend-ec2`
   Outbound: All

sg-backend-ec2 (backend instances)

   Inbound: HTTP 80 from `sg-backend-alb`
   Inbound (optional): SSH 22 from your IP (for debugging)
   Outbound: All

Note: Use security group references (SG → SG) — this ensures only the components you expect can talk to one another.

---


## Packaging for Elastic Beanstalk (gotchas)

Important: When you create the zip file to upload to Elastic Beanstalk, zip the *contents* of the app folder, not the folder itself.

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

## Deploy the Backend (internal EB environment)

In Elastic Beanstalk console:

1. Create Application → Upload `backend.zip`.
2. Configure (more options) → Network:

    VPC: `week2-secure-vpc`
    Load balancer subnets: priv-subnet-1, priv-subnet-2
    Instance subnets: priv-subnet-1, priv-subnet-2
    Load balancer = Application Load Balancer, Scheme = internal
3. Security groups: assign `sg-backend-alb` for ALB and `sg-backend-ec2` for instances
4. IAM roles: `aws-elasticbeanstalk-service-role` + `aws-elasticbeanstalk-ec2-role`
5. Create environment → Wait for Green.

You will get an internal ALB DNS (e.g., `internal-backend-xxxx.elb.amazonaws.com`). Note it down — this is the backend URL used by the frontend inside the VPC.

---

## Deploy the Frontend (public EB environment)

1. Create Application → Upload `frontend.zip`.
2. Configure (more options) → Network:

    VPC: `week2-secure-vpc`
    Load balancer subnets: **pub-subnet-1**, **pub-subnet-2** (ALB must be public)
    Instance subnets: **priv-subnet-1**, **priv-subnet-2** (instances have no public IP)
    Load balancer = Application Load Balancer, **Scheme = internet-facing**
3. Security groups: assign `sg-frontend-alb` (ALB) and `sg-frontend-ec2` (instances)
4. Environment properties (env vars): set `BACKEND_URL = http://<internal-backend-alb-dns>/data`
5. IAM roles: same as above
6. Create environment → Wait for **Green**

Open the frontend Environment URL (public) — you should see the styled UI and backend data.

---

## Testing & verification (do these checks)

Frontend works: Open the public URL and confirm the UI displays backend data.
Backend unreachable publicly: From your local machine, try to access backend internal ALB URL — it should not work (expected).
Security groups: Confirm `sg-backend-ec2` inbound only allows `sg-backend-alb`; `sg-backend-alb` inbound only allows `sg-frontend-ec2`.
No public IPs on instances: EC2 Console → Instances → check `IPv4 Public IP` is blank for both frontend and backend instances.
Health checks: ALB targets should be healthy; EB Health should be Green.

---

## Common errors & how to fix them

Deployment failed — `eb-engine.log` shows ModuleNotFoundError: Ensure `requirements.txt` lists all dependencies and is at top level.
`application.py` not found / EB cannot start app: Make sure you zipped app contents, not the folder itself, and the file is named `application.py` with WSGI callable `application = Flask(__name__)`.
Health checks failing: Make sure app listens on the correct port (EB binds application to port EB expects), and health check path `/` returns 200 quickly.
Frontend cannot talk to backend: Check SG rules and that `BACKEND_URL` points to the correct internal ALB DNS.

---

## Cost & performance notes (quick)

NAT Gateway and Elastic IP are charged hourly — remember to release them during testing if not needed.
t2/t3.micro instances are cost-effective for testing; for production choose t3.small+ and enable auto-scaling.
Elastic Beanstalk simplifies operations but hides finer controls; for full control consider ECS/EKS or EC2 Autoscaling groups later.

---


## Lessons learned (personal notes)

Elastic Beanstalk is fantastic for quick, managed deployments, but it requires mindful networking/configuration to be secure.
Always test DNS and reachability from inside the VPC to ensure internal ALBs are resolvable and usable.
Security groups referencing other SGs are a simple and powerful way to enforce least privilege networking.
Zip structure and naming are tiny things that often break deployments, so fix them early.

---

## Next steps & improvements

Add HTTPS to the frontend ALB (use AWS Certificate Manager).
Add CloudWatch alarms and autoscaling policies.
Use RDS (in private subnets) for persistent data and Secrets Manager for credentials.
Automate the whole stack with CloudFormation / Terraform for repeatability.
Add a Bastion host or use Systems Manager (SSM) Session Manager for secure troubleshooting.

---

## Final thoughts 

Building this secure, two-tier app felt like preparing jollof rice perfectly, a bit of setup, the right ingredients, and patience on low heat until everything comes together. Big thanks to everyone encouraging the weekly challenge. If you’re thinking of starting a hands-on cloud challenge, do it. it changes how you think about architecture, cost, and security.
