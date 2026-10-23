# AWS CloudFormation 3D Infrastructure Viewer

![AWS](https://img.shields.io/badge/AWS-CloudFormation-FF9900?logo=amazonaws&logoColor=white)
![IaC](https://img.shields.io/badge/Infrastructure%20as%20Code-YAML-blue)
![EC2](https://img.shields.io/badge/EC2-Ubuntu%20t3.micro-orange)
![Three.js](https://img.shields.io/badge/Three.js-3D%20viewer-black)

**Automated AWS Cloud Infrastructure & Web Presentation Deployment.**
One CloudFormation **YAML** template builds a complete network and a web server. The server then publishes an interactive **3D architecture viewer** (Three.js) by itself at boot, with no manual setup.

![3D viewer running on the EC2 instance](docs/screenshots/11-live-3d-viewer.png)

> Deployed and tested in **ap-south-1 (Mumbai)**: the stack reached `CREATE_COMPLETE` and the page above was served by the EC2 instance.
>
> **Live demo of the viewer (GitHub Pages):** `https://<your-username>.github.io/<repo-name>/`

## Objective

Use Infrastructure as Code (IaC) to deploy a secure, repeatable and fully automated cloud environment (VPC, subnet, security group and EC2) with AWS CloudFormation, and present it through an interactive 3D web page.

## What was built

| Component | What it does | How it is done in the template |
|---|---|---|
| **Custom VPC** | An isolated private network, CIDR `10.0.0.0/16` | `AWS::EC2::VPC` with `EnableDnsSupport: true` and `EnableDnsHostnames: true` |
| **Internet Gateway** | Connects the VPC to the public internet | `AWS::EC2::InternetGateway` attached with `AWS::EC2::VPCGatewayAttachment` |
| **Public Subnet** | Holds the public-facing server, CIDR `10.0.1.0/24` | `AWS::EC2::Subnet` with `MapPublicIpOnLaunch: true`, so new instances get a public IP automatically |
| **Route Table** | Sends internet traffic to the gateway | `AWS::EC2::RouteTable` and `AWS::EC2::Route` (`0.0.0.0/0` to the Internet Gateway), linked with `AWS::EC2::SubnetRouteTableAssociation` |
| **Security Group** | Firewall for the server | `AWS::EC2::SecurityGroup` with inbound **22** (SSH) and **80** (HTTP) |
| **EC2 Instance** | Free-tier style Ubuntu web server | `AWS::EC2::Instance`, `t3.micro`, placed in the public subnet with the security group attached |
| **User-Data script** | Sets up the web server on first boot | Installs `apache2`, then writes the 3D viewer to `/var/www/html/index.html` |

## Architecture flow

```
Internet -> Internet Gateway -> Route Table -> Public Subnet (10.0.1.0/24)
         -> Security Group (22, 80) -> EC2 (Ubuntu + Apache) -> 3D viewer page
                         all inside VPC 10.0.0.0/16
```

## Deployment proof (screenshots)

Captured from the AWS Console after deployment (region `ap-south-1`). AWS account IDs are blacked out.

### CloudFormation stack
| | |
|---|---|
| **Stack events:** all events finished with `CREATE_COMPLETE` | ![Stack events](docs/screenshots/01-stack-events.png) |
| **Stack resources (1/2)** | ![Resources part 1](docs/screenshots/02-stack-resources-part1.png) |
| **Stack resources (2/2):** the 9 resources, including the route-table association and gateway attachment | ![Resources part 2](docs/screenshots/03-stack-resources-part2.png) |
| **Outputs:** `InstancePublicIP` and `WebsiteURL` | ![Outputs](docs/screenshots/04-stack-outputs.png) |
| **Template tab:** the YAML template stored in the stack | ![Template](docs/screenshots/05-template-tab.png) |
| **Template Studio:** the same template in the visual editor | ![Template Studio](docs/screenshots/06-template-studio.png) |

### Network and server
| | |
|---|---|
| **VPC** `10.0.0.0/16`, DNS resolution and DNS hostnames enabled | ![VPC](docs/screenshots/07-vpc.png) |
| **Route table:** `0.0.0.0/0` to the Internet Gateway, `10.0.0.0/16` local, public subnet associated | ![Route table](docs/screenshots/08-route-table.png) |
| **Security group:** inbound HTTP (80) and SSH (22) | ![Security group](docs/screenshots/09-security-group.png) |
| **EC2 instance:** `t3.micro`, running, status checks passed | ![EC2](docs/screenshots/10-ec2-instances.png) |

## Repository structure

```
.
├── cloudformation/
│   └── template.yaml        # the full CloudFormation stack
├── docs/
│   ├── index.html           # the 3D viewer (also served by GitHub Pages)
│   └── screenshots/         # deployment proof (account IDs blacked out)
└── README.md
```

## How to deploy

**Option A: AWS Console**
1. Open **CloudFormation > Create stack > With new resources**.
2. Upload `cloudformation/template.yaml`, name the stack and create it.
3. Wait for `CREATE_COMPLETE`, then 1-2 more minutes for Apache to install.
4. Open `WebsiteURL` from the **Outputs** tab. Use `http://`, not `https://`.

**Option B: AWS CLI**
```bash
aws cloudformation create-stack \
  --stack-name aws-3d-viewer \
  --template-body file://cloudformation/template.yaml \
  --region ap-south-1

aws cloudformation describe-stacks --stack-name aws-3d-viewer \
  --query "Stacks[0].Outputs"
```

**Clean up** (to avoid charges):
```bash
aws cloudformation delete-stack --stack-name aws-3d-viewer
```

## How the web page gets onto the server

The viewer is larger than the 16 KB EC2 User-Data limit, so `docs/index.html` is stored in the template as **gzip + base64** text. On first boot the script installs Apache and decodes the page:

```bash
echo '<gzip+base64 text>' | base64 -d | gunzip > /var/www/html/index.html
```

To regenerate the encoded text after editing the page:
```bash
gzip -9 -n -c docs/index.html | base64 -w0
```
Keep the whole User-Data under 16,384 bytes.

## Challenges and fixes

| Challenge | Fix |
|---|---|
| The page contains `${...}` text. With `Fn::Sub` CloudFormation would treat it as a variable and the template would fail | Use plain `Fn::Base64` (not `!Sub`) for the User-Data |
| The full page (about 27 KB) is bigger than the 16 KB User-Data limit | Store the page as gzip + base64 (about 13 KB in total) and decode it on the server |
| The EC2 instance could boot before the internet route exists, so `apt-get` would fail and Apache would not be installed | `DependsOn` on the route and the subnet association, plus retry loops around `apt-get` |
| The first deployment timed out in the browser (`ERR_CONNECTION_TIMED_OUT`) | After deploying again the page loaded normally. The exact cause of the first timeout was not isolated |
| The AMI ID is region-specific | The AMI in the template worked in `ap-south-1`. In another region, use an Ubuntu AMI from that region |

## Troubleshooting

| Symptom | What to check |
|---|---|
| `ERR_CONNECTION_TIMED_OUT` | Use `http://` (not `https://`). Check the security group allows port 80, the route table has `0.0.0.0/0` to the IGW, and the subnet is associated with that route table. Some office or college networks block port 80 |
| Connection refused | Apache is not running yet. Wait 1-2 minutes, then check the log below |
| Default "Apache2 Ubuntu" page | The step that writes `index.html` did not finish. Check the log below |
| `InvalidAMIID` | Use an Ubuntu AMI from your own region |
| Blank page | The browser needs internet to load Three.js from the CDN |

Boot log on the server:
```bash
sudo tail -50 /var/log/cloud-init-output.log
```
In the EC2 console you can also use **Actions > Monitor and troubleshoot > Get system log**.

## Run the viewer without AWS (GitHub Pages)

1. Push this repo to GitHub.
2. **Settings > Pages > Deploy from a branch**, select `main` and the `/docs` folder.
3. The viewer will be live at `https://<your-username>.github.io/<repo-name>/`.

## Security notes

- SSH (port 22) is open to `0.0.0.0/0` for demo purposes. For anything beyond a demo, restrict it to your own IP.
- Delete the stack when you are done, so the resources do not keep running.

## Tech stack

AWS CloudFormation (YAML), Amazon VPC, EC2, Apache2 on Ubuntu, Three.js, HTML/CSS/JavaScript.

## Author

**Shantanu Gawai** | [GitHub](https://github.com/Shaan027) | [LinkedIn](https://in.linkedin.com/in/shantanu-gawai-529258242)
