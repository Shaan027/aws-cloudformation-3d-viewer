# AWS CloudFormation 3D Infrastructure Viewer

Automated AWS Cloud Infrastructure & Web Presentation Deployment using AWS CloudFormation and Three.js.

## 🚀 Overview
This project uses Infrastructure as Code (IaC) to automatically deploy a complete network and web server environment on AWS, and serves an interactive 3D architecture viewer built with Three.js.

## 🛠️ What is Built
* **Custom VPC** (`10.0.0.0/16`) with DNS support enabled.
* **Internet Gateway** connected to the VPC[cite: 2].
* **Public Subnet** (`10.0.1.0/24`) with automatic public IP assignment[cite: 2].
* **Route Table** routing internet traffic (`0.0.0.0/0`) to the Internet Gateway[cite: 2].
* **Security Group** allowing inbound traffic on Port 80 (HTTP) and Port 22 (SSH)[cite: 2].
* **EC2 Instance** (`t3.micro` running Ubuntu & Apache) that automatically decodes and serves the 3D viewer on boot[cite: 2].

## 📂 Repository Structure
```text
.
├── cloudformation/
│   └── template.yaml        # Full CloudFormation stack template
├── docs/
│   ├── index.html           # Interactive 3D viewer (Three.js)
│   └── screenshots/         # Deployment proof screenshots
└── README.md
⚙️ How to Deploy
Open the AWS CloudFormation Console[cite: 2].

Create a new stack using the template file (cloudformation/template.yaml)[cite: 2].

Once the stack reaches CREATE_COMPLETE, go to the Outputs tab and open the WebsiteURL[cite: 2].
