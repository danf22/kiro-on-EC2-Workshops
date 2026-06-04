# Kiro IDE on EC2 — Workshop Deployment

Deploy multiple EC2 instances pre-loaded with **Kiro IDE** and accessible via a web browser using **Amazon DCV** remote desktop. Designed for workshops where each participant gets their own dedicated instance.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  AWS Cloud                                                │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  VPC (10.0.0.0/16)                                  │ │
│  │                                                     │ │
│  │  ┌───────────────────────────────────────────────┐  │ │
│  │  │  Public Subnet (10.0.1.0/24)                  │  │ │
│  │  │                                               │  │ │
│  │  │   ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │ │
│  │  │   │ Instance │  │ Instance │  │ Instance │   │  │ │
│  │  │   │    #1    │  │    #2    │  │   #N     │   │  │ │
│  │  │   │  Kiro +  │  │  Kiro +  │  │  Kiro +  │   │  │ │
│  │  │   │   DCV    │  │   DCV    │  │   DCV    │   │  │ │
│  │  │   └────┬─────┘  └────┬─────┘  └────┬─────┘   │  │ │
│  │  │        │              │              │         │  │ │
│  │  └────────┼──────────────┼──────────────┼─────────┘  │ │
│  │           │              │              │            │ │
│  └───────────┼──────────────┼──────────────┼────────────┘ │
│              │              │              │               │
└──────────────┼──────────────┼──────────────┼───────────────┘
               │              │              │
         ┌─────┴─────┐ ┌─────┴─────┐ ┌─────┴─────┐
         │ Browser   │ │ Browser   │ │ Browser   │
         │ User #1   │ │ User #2   │ │ User #N   │
         └───────────┘ └───────────┘ └───────────┘
              https://<IP>:8443
```

## What's Included Per Instance

- **Amazon Linux 2023** with GNOME Desktop
- **Amazon DCV** (remote desktop via browser on port 8443)
- **Kiro IDE** v0.12.263 (latest stable)
- **Kiro CLI** (latest)
- **AWS CLI v2**
- **Node.js** + npm
- **Git**

## Prerequisites

- An AWS account with permissions to create VPCs, EC2 instances, IAM roles, and AutoScaling groups.
- An existing EC2 Key Pair in the target region (for optional SSH fallback access).

## Deploy

### Option 1: AWS Console

1. Go to [CloudFormation Console](https://console.aws.amazon.com/cloudformation)
2. Click **Create Stack** → **With new resources**
3. Upload `kiro-workshop-cfn.yaml`
4. Fill in parameters:
   - **NumberOfInstances**: How many participants in the workshop
   - **InstanceType**: `t3.xlarge` recommended (4 vCPU, 16 GiB)
   - **VolumeSize**: 50 GB default
   - **KeyPairName**: Select your key pair
   - **AllowedCIDR**: Restrict to your network (e.g., `203.0.113.0/24`) or leave `0.0.0.0/0` for open access
   - **DCVPassword**: Password all participants will use to log in
5. Check the IAM capabilities checkbox and create the stack
6. Wait ~15–20 minutes for all instances to initialize

### Option 2: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name kiro-workshop \
  --template-body file://kiro-workshop-cfn.yaml \
  --parameters \
    ParameterKey=NumberOfInstances,ParameterValue=10 \
    ParameterKey=InstanceType,ParameterValue=t3.xlarge \
    ParameterKey=KeyPairName,ParameterValue=my-key-pair \
    ParameterKey=DCVPassword,ParameterValue=MySecurePass123 \
    ParameterKey=AllowedCIDR,ParameterValue=0.0.0.0/0 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## Accessing the Instances

1. In the EC2 console, find instances tagged with your stack name
2. Note the **Public IP** of each instance
3. Open a browser and navigate to: `https://<PUBLIC_IP>:8443`
4. Accept the self-signed certificate warning
5. Log in with:
   - **Username**: `kirouser`
   - **Password**: the `DCVPassword` you set during deployment
6. Double-click the **Kiro IDE** desktop shortcut to launch it

## Workshop Facilitator Tips

- **Assign instances**: List public IPs from EC2 console and assign one per participant
- **Security**: Restrict `AllowedCIDR` to your venue's IP range
- **Cost**: Each `t3.xlarge` instance costs ~$0.17/hr. 10 instances for a 4-hour workshop ≈ $7
- **Cleanup**: Delete the CloudFormation stack when the workshop ends to stop all charges

## Cleanup

```bash
aws cloudformation delete-stack --stack-name kiro-workshop --region us-east-1
```

Or delete via the CloudFormation console.

## Estimated Cost

| Component | Cost (per instance/hour) |
|-----------|-------------------------|
| t3.xlarge | ~$0.1664/hr |
| EBS 50GB (gp3) | ~$0.006/hr |
| Data transfer | Variable |

**Example**: 20 instances × 8 hours = ~$28 total

## Troubleshooting

- **Can't connect on port 8443**: Check the Security Group allows your IP. Verify the instance is in `running` state and finished initialization (check `/var/log/kiro-setup.log` via SSH or SSM).
- **DCV session not available**: SSH in and run `sudo dcv create-session --owner kirouser --type=console kiro-session`
- **Kiro not launching**: Try running `/usr/local/bin/kiro --no-sandbox` from a terminal inside the DCV session.
- **Certificate warning**: Expected behavior with self-signed certs. Click "Advanced" → "Proceed" in your browser.
