# 🛠️ AWS-Lambda-Volume-Backup-Automation-Project (With SNS, EventBridge)

## 🎯 Objective:

To automate EBS volume snapshot creation across **all AWS regions** for volumes in the **"in-use"** state using AWS Lambda. This project includes launching EC2, static site deployment, AMI operations, volume creation, snapshot automation, SNS notifications, and scheduled execution using EventBridge.

---
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/lambda%20infrastructure.png)
## 📌 Project Description 

* 🔄 Automates backup of all **“in-use” EBS volumes** across **all AWS regions**
* ⚙️ Uses an **AWS Lambda function** written in Python
* 🔍 Lambda scans each region for attached EBS volumes
* 📸 Creates **snapshots** of each volume
* 🔔 Sends **SNS notifications** after snapshot creation
* 🕒 Scheduled  with **Amazon EventBridge (CloudWatch Events)** for regular execution
* 🛡️ Ensures **automated, region-wide backup** without manual effort

---

## ✅ Prerequisites 

1. **AWS Account** – With admin access to EC2, Lambda, IAM, SNS, and EventBridge.

2. **Basic AWS Knowledge** – Familiarity with EC2, EBS, Lambda, IAM, SNS, and EventBridge.

3. **EC2 Instance** – Amazon Linux 2 with a web server (Apache), key pair, and open ports 22 & 80.

4. **IAM Role for Lambda** – Attach policies:-

   * `AmazonEC2FullAccess`
   * `AmazonSNSFullAccess`
   * `CloudWatchLogsFullAccess`

5. **SNS Topic** – Created with an email subscription to receive snapshot notifications.

6. **Lambda Function** – Python 3.8+, configured with the IAM role and code to back up volumes.

7. **EventBridge Rule** – Schedule Lambda using a cron expression (e.g., daily at 12 AM).

8. **Regions & Volumes** – EBS volumes in ‘in-use’ state across multiple regions.
****
## 🔧 Step-by-Step Implementation:
### 🔹 1. **Launch an EC2 Instance in `us-east-1`**

* Go to **EC2 Dashboard** > Launch instance
* Choose **Amazon Linux 2 AMI**
* Instance type: `t2.micro` (Free Tier eligible)
* Key pair: Create/choose one for SSH access
* Security Group:

  * Allow SSH (port 22)
  * Allow HTTP (port 80)
* Launch the instance

---

### 🔹 2. **Deploy Static Website on EC2**

1. Connect via SSH:

   ```bash
   ssh -i your-key.pem ec2-user@<Public-IP>
   ```

2. Install and start Apache:

   ```bash
   sudo yum update -y
   sudo yum install httpd -y
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

3. Upload HTML template:

   ```bash
   sudo mv website.html /var/www/html/index.html
   ```

4. Visit `http://<Public-IP>` to verify the site.

---

### 🔹 3. **Create an AMI of EC2**

* Select EC2 instance
* Click **Actions > Image and templates > Create image**
* Provide name (e.g., `WebServerAMI`)
* Click **Create Image**
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-11%20001319.png)
---

### 🔹 4. **Copy AMI to Another Region**

* Go to **AMI > Copy AMI**
* Destination region: `ap-south-1` (Mumbai)
* Click **Copy**

---

### 🔹 5. **Launch EC2 from Copied AMI in `ap-south-1`**

* Navigate to `ap-south-1`
* Launch instance using the copied AMI
* Ensure security group and key pair are configured

---
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-11%20001609.png)
### 🔹 6. **Create Extra EBS Volumes in Both Regions**

* Go to **EC2 > Elastic Block Store > Volumes**
* Create volumes (8 GB, General Purpose SSD)
* Keep **state** as **Available**

---
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-10%20235252.png )

### 🔹 7.  **Create the Lambda Function**

* Go to **AWS Lambda Console** → Create function.
* Choose **Author from scratch**.
* Name it (e.g., `EBSVolumeSnapshotBackup`).
* Select **Python 3.x** runtime.
* Assign the **IAM role** created earlier (`Lambda_EBS_Snapshot_Role`).
* Paste your Lambda function code into the editor.
#### ✅ Python Code:

```python
import boto3
import json
import datetime

def lambda_handler(event, context):
    regions = []
    ec2_client = boto3.client('ec2')
    regions_response = ec2_client.describe_regions()
    for region in regions_response['Regions']:
        regions.append(region['RegionName'])

    snapshots_created = []
    timestamp = datetime.datetime.now().strftime("%Y-%m-%d_%H-%M-%S")

    for region in regions:
        ec2 = boto3.client('ec2', region_name=region)
        try:
            volumes = ec2.describe_volumes(Filters=[{'Name': 'status', 'Values': ['in-use']}])['Volumes']
            for volume in volumes:
                volume_id = volume['VolumeId']
                snapshot = ec2.create_snapshot(
                    VolumeId=volume_id,
                    Description=f"Snapshot of {volume_id} - {timestamp}"
                )
                snapshots_created.append({
                    "Region": region,
                    "VolumeId": volume_id,
                    "SnapshotId": snapshot['SnapshotId']
                })
        except Exception as e:
            print(f"Error in region {region}: {str(e)}")

    # SNS notification
    sns = boto3.client('sns')
    sns.publish(
        TopicArn="arn:aws:sns:REGION:ACCOUNT_ID:TOPIC_NAME",
        Subject="EBS Snapshots Created",
        Message=json.dumps(snapshots_created)
    )

    return {
        'statusCode': 200,
        'body': json.dumps(snapshots_created)
    }
```
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-10%20235048.png)
---

### 🔹 8. **Create SNS Topic and Subscribe**

1. Go to **SNS > Topics > Create topic**

   * Name: `SnapshotNotification`
2. Create **Email Subscription**

   * Enter your email
   * Confirm from inbox
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-11%20000856.png)
---

### 🔹 9. **Assign IAM Role to Lambda**

* Go to **IAM > Roles > Create Role**

  * Trusted entity: Lambda
  * Attach policies:

    * `AmazonEC2FullAccess`
    * `AmazonSNSFullAccess`
  * Name: `LambdaEC2SnapshotRole`
* Go to **Lambda > Configuration > Permissions**

  * Click on **Edit Role**
  * Attach `LambdaEC2SnapshotRole`

---

### 🔹 10. **Test Lambda Function**

* Go to **Test tab**
* Create a dummy test event `{}` and run
* If successful:

  * Snapshots will be created
  * Email notification via SNS

---

### 🔹 11. **Schedule Automation with EventBridge**
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-11%20000947.png)
* Go to **Amazon EventBridge > Rules > Create rule**
* Name: `SnapshotScheduler`
* Event Source: **Schedule**

  * Cron: `cron(0 2 * * ? *)` → every day at 2 AM UTC
* Target: **Lambda function** (`volume_backup`)

---

## 🧪 Final Validation:

* ✅ Snapshots created daily in **all regions**
* ✅ Notification email sent via **SNS**
* ✅ EC2 instances created from AMI work in both regions
* ✅ Volumes created and remain available
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/Screenshot%202025-06-11%20001143.png)
---

## 📦 Project Summary

| Component       | Description                                |
| --------------- | ------------------------------------------ |
| **EC2**         | Hosting website, creating AMIs             |
| **EBS**         | Volumes to back up                         |
| **AMI**         | Created and copied for regional redundancy |
| **Lambda**      | Automates snapshot creation                |
| **SNS**         | Sends notifications after backup           |
| **EventBridge** | Schedules daily snapshot job               |
| **IAM Role**    | Grants Lambda necessary permissions        |

---

## 🔍 Project Summary

This project automates the backup of EBS volumes using a **serverless architecture** in AWS. An **AWS Lambda function**, written in Python, scans **all available AWS regions** to find EBS volumes in the **“in-use” state**. For each such volume, it creates a **snapshot**, providing a point-in-time backup.

The Lambda function is triggered on a **schedule using Amazon EventBridge**, allowing backups to run automatically at defined intervals (e.g., daily). After snapshot creation, **Amazon SNS** sends notifications (e.g., email) to inform administrators about the snapshot status, including volume and region details.

The entire setup requires **no manual intervention**, ensures **cross-region snapshot coverage**, and enhances **disaster recovery** readiness and **data protection** for EC2-based infrastructure.
![](https://github.com/gaurav3972/AWS-Lambda-Volume-Backup-Automation-Project/blob/main/IMAGES/3.png)
### ✅ Key Features:

* Multi-region EBS volume scanning
* Snapshot creation automation
* Scheduled execution with EventBridge
* Real-time notification via SNS
* Minimal operational overhead

---
