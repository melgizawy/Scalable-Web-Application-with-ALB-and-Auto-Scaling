![Scalable Web Application with ALB and Auto Scaling](https://github.com/user-attachments/assets/b2c0429e-f8f7-4711-b89c-d422e9c66e2d)

## 📋 Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Create VPC and Network Infrastructure](#step-1-create-vpc-and-network-infrastructure)
- [Step 2: Create Security Groups](#step-2-create-security-groups)
- [Step 3: Create IAM Role](#step-3-create-iam-role)
- [Step 4: Create Launch Template](#step-4-create-launch-template)
- [Step 5: Create Application Load Balancer](#step-5-create-application-load-balancer)
- [Step 6: Create Auto Scaling Group](#step-6-create-auto-scaling-group)
- [Step 7: Set Up CloudWatch Monitoring](#step-7-set-up-cloudwatch-monitoring)
- [Step 8: Testing the Application](#step-8-testing-the-application)
- [Troubleshooting Tips](#troubleshooting-tips)

## 🎯 Prerequisites

Before starting, ensure you have:
- AWS Account with appropriate permissions
- Basic understanding of AWS services
- Email address for notifications

## Step 1: Create VPC and Network Infrastructure

### 1.1 Create VPC

1. **Navigate to VPC Dashboard**
   - Go to AWS Console → Services → VPC
   - Click "Create VPC"

2. **Configure VPC Settings**
   ```
   VPC Settings:
   - Resources to create: VPC and more
   - Name tag auto-generation: WebApp
   - IPv4 CIDR block: 10.0.0.0/16
   - IPv6 CIDR block: No IPv6 CIDR block
   - Tenancy: Default
   
   Number of Availability Zones: 2
   Number of public subnets: 2
   Number of private subnets: 0
   
   NAT gateways: None
   VPC endpoints: None
   DNS hostnames: Enable
   DNS resolution: Enable
   ```

3. **Click "Create VPC"** and wait for completion

### 1.2 Verify Network Setup

1. **Check Created Resources**
   - VPC: `WebApp-vpc`
   - Subnets: `WebApp-subnet-public1-us-east-1a`, `WebApp-subnet-public2-us-east-1b`
   - Internet Gateway: `WebApp-igw`
   - Route Table: `WebApp-rtb-public`

![image](https://github.com/user-attachments/assets/827d0532-708a-4849-81c8-6c64ac577280)

## Step 2: Create Security Groups

### 2.1 Create Web Server Security Group

1. **Navigate to Security Groups**
   - In VPC Dashboard → Security Groups
   - Click "Create security group"

2. **Configure Web Server Security Group**
   ```
   Basic details:
   - Security group name: WebServer-SG
   - Description: Security group for web servers
   - VPC: Select your WebApp-vpc
   
   Inbound rules:
   Rule 1: HTTP
   - Type: HTTP
   - Protocol: TCP
   - Port range: 80
   - Source: 0.0.0.0/0
   - Description: Allow HTTP traffic
   
   Rule 2: HTTPS
   - Type: HTTPS
   - Protocol: TCP
   - Port range: 443
   - Source: 0.0.0.0/0
   - Description: Allow HTTPS traffic
   
   Rule 3: SSH
   - Type: SSH
   - Protocol: TCP
   - Port range: 22
   - Source: 0.0.0.0/0 (or your IP for better security)
   - Description: Allow SSH access
   
   Outbound rules: Keep default (All traffic allowed)
   ```

3. **Click "Create security group"**
   ![image](https://github.com/user-attachments/assets/7af7c303-b42a-437e-82c7-c04e3d42a4ca)


### 2.2 Create Application Load Balancer Security Group

1. **Create another Security Group**
   ```
   Basic details:
   - Security group name: ALB-SG
   - Description: Security group for Application Load Balancer
   - VPC: Select your WebApp-vpc
   
   Inbound rules:
   Rule 1: HTTP
   - Type: HTTP
   - Protocol: TCP
   - Port range: 80
   - Source: 0.0.0.0/0
   - Description: Allow HTTP from internet
   
   Rule 2: HTTPS
   - Type: HTTPS
   - Protocol: TCP
   - Port range: 443
   - Source: 0.0.0.0/0
   - Description: Allow HTTPS from internet
   ```

2. **Click "Create security group"**
![image](https://github.com/user-attachments/assets/a5933f4b-a505-4384-8170-b371c3765929)

## Step 3: Create IAM Role

### 3.1 Create IAM Role for EC2

1. **Navigate to IAM**
   - Go to AWS Console → Services → IAM
   - Click "Roles" in the left sidebar
   - Click "Create role"

2. **Configure Role**
   ```
   Step 1: Select trusted entity
   - Trusted entity type: AWS service
   - Use case: EC2
   - Click "Next"
   
   Step 2: Add permissions
   - Search and select: "CloudWatchAgentServerPolicy"
   - Search and select: "AmazonSSMManagedInstanceCore"
   - Click "Next"
   
   Step 3: Name and review
   - Role name: WebApp-EC2-Role
   - Description: IAM role for WebApp EC2 instances
   - Click "Create role"
   ```

![image](https://github.com/user-attachments/assets/f31aa3cc-0918-4188-a686-75a7c3ac7058)

![image](https://github.com/user-attachments/assets/10e158a4-778d-4c4d-807c-ba78d1650ca5)

![image](https://github.com/user-attachments/assets/ac741982-dfe5-4e3c-8d29-84bdbb352388)

![Screenshot 2025-06-01 205811](https://github.com/user-attachments/assets/a9546beb-b7c0-4ec6-b002-9ceb5604a6b8)


## Step 4: Create Launch Template

### 4.1 Create Key Pair (if needed)

1. **Navigate to EC2 Dashboard**
   - Go to AWS Console → Services → EC2
   - Click "Key Pairs" in the left sidebar
   - Click "Create key pair"

2. **Configure Key Pair**
   ```
   - Name: WebApp-KeyPair
   - Key pair type: RSA
   - Private key file format: .pem
   - Click "Create key pair"
   ```

3. **Download and save the .pem file securely**

### 4.2 Create Launch Template

1. **Navigate to Launch Templates**
   - In EC2 Dashboard → Launch Templates
   - Click "Create launch template"

2. **Configure Launch Template**
   ```
   Launch template name and description:
   - Launch template name: WebApp-LaunchTemplate
   - Template version description: Initial version for web application
   
   Application and OS Images:
   - Quick Start: Amazon Linux
   - Amazon Linux 2023 AMI (latest version)
   
   Instance type:
   - Instance type: t3.micro
   
   Key pair:
   - Key pair name: WebApp-KeyPair (or your existing key pair)
   
   Network Settings:
   - Subnet: Don't include in launch template
   - Security groups: WebServer-SG
   
   Advanced details:
   - IAM instance profile: WebApp-EC2-Role
   - User data: (Copy the script below)
   ```

3. **User Data Script**
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   
   # Get instance metadata
   INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
   AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
   INSTANCE_TYPE=$(curl -s http://169.254.169.254/latest/meta-data/instance-type)
   LOCAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
   
   # Create web page
   cat <<EOF > /var/www/html/index.html
   <!DOCTYPE html>
   <html>
   <head>
       <title>Scalable Web Application - AWS Project</title>
       <style>
           body { 
               font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
               margin: 0; 
               padding: 20px; 
               background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
               color: white;
           }
           .container { 
               max-width: 1000px; 
               margin: 0 auto; 
               background: rgba(255,255,255,0.1);
               padding: 30px;
               border-radius: 15px;
               backdrop-filter: blur(10px);
           }
           .header {
               text-align: center;
               margin-bottom: 30px;
           }
           .server-info { 
               background: rgba(255,255,255,0.2); 
               padding: 20px; 
               border-radius: 10px; 
               margin: 20px 0;
           }
           .features {
               display: grid;
               grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
               gap: 20px;
               margin: 30px 0;
           }
           .feature-card {
               background: rgba(255,255,255,0.15);
               padding: 20px;
               border-radius: 10px;
               text-align: center;
           }
           .status-indicator {
               display: inline-block;
               width: 12px;
               height: 12px;
               background: #4CAF50;
               border-radius: 50%;
               margin-right: 8px;
           }
           .stress-button {
               background: #ff6b6b;
               color: white;
               border: none;
               padding: 15px 30px;
               border-radius: 25px;
               cursor: pointer;
               font-size: 16px;
               margin: 10px;
               transition: all 0.3s;
           }
           .stress-button:hover {
               background: #ff5252;
               transform: translateY(-2px);
           }
       </style>
   </head>
   <body>
       <div class="container">
           <div class="header">
               <h1>🚀 AWS Scalable Web Application</h1>
               <p>High Availability • Auto Scaling • Load Balancing</p>
           </div>
           
           <div class="server-info">
               <h3>🖥️ Server Information</h3>
               <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
               <p><strong>Availability Zone:</strong> $AZ</p>
               <p><strong>Instance Type:</strong> $INSTANCE_TYPE</p>
               <p><strong>Local IP:</strong> $LOCAL_IP</p>
               <p><strong>Timestamp:</strong> <span id="timestamp"></span></p>
           </div>
           
           <div class="features">
               <div class="feature-card">
                   <h4>🛡️ High Availability</h4>
                   <p><span class="status-indicator"></span>Multi-AZ deployment</p>
                   <p><span class="status-indicator"></span>Fault tolerance</p>
               </div>
               <div class="feature-card">
                   <h4>📈 Auto Scaling</h4>
                   <p><span class="status-indicator"></span>CPU-based scaling</p>
                   <p><span class="status-indicator"></span>Cost optimization</p>
               </div>
               <div class="feature-card">
                   <h4>⚖️ Load Balancing</h4>
                   <p><span class="status-indicator"></span>Traffic distribution</p>
                   <p><span class="status-indicator"></span>Health monitoring</p>
               </div>
               <div class="feature-card">
                   <h4>📊 Monitoring</h4>
                   <p><span class="status-indicator"></span>CloudWatch metrics</p>
                   <p><span class="status-indicator"></span>SNS notifications</p>
               </div>
           </div>
           
           <div style="text-align: center; margin-top: 30px;">
               <h3>🧪 Test Auto Scaling</h3>
               <p>Click the button below to simulate high CPU load and trigger auto scaling:</p>
               <button class="stress-button" onclick="stressTest()">Start CPU Stress Test</button>
               <p id="stress-status"></p>
           </div>
       </div>
       
       <script>
           // Update timestamp
           function updateTimestamp() {
               document.getElementById('timestamp').textContent = new Date().toLocaleString();
           }
           updateTimestamp();
           setInterval(updateTimestamp, 1000);
           
           // CPU stress test function
           function stressTest() {
               const button = document.querySelector('.stress-button');
               const status = document.getElementById('stress-status');
               
               button.disabled = true;
               button.textContent = 'Running Stress Test...';
               status.textContent = 'Generating high CPU load for 60 seconds...';
               status.style.color = '#ff6b6b';
               
               // CPU intensive operation
               const startTime = Date.now();
               const duration = 60000; // 60 seconds
               
               function cpuIntensiveTask() {
                   const endTime = Date.now() + 100; // 100ms chunks
                   while (Date.now() < endTime) {
                       Math.random() * Math.random();
                   }
                   
                   if (Date.now() - startTime < duration) {
                       setTimeout(cpuIntensiveTask, 1);
                   } else {
                       button.disabled = false;
                       button.textContent = 'Start CPU Stress Test';
                       status.textContent = 'Stress test completed! Check CloudWatch for scaling events.';
                       status.style.color = '#4CAF50';
                   }
               }
               
               cpuIntensiveTask();
           }
       </script>
   </body>
   </html>
   EOF
   
   # Create health check endpoint
   echo "OK" > /var/www/html/health.html
   
   # Install CloudWatch agent
   wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
   rpm -U ./amazon-cloudwatch-agent.rpm
   
   # Start CloudWatch agent with basic config
   cat <<EOF > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
   {
       "metrics": {
           "namespace": "WebApp/EC2",
           "metrics_collected": {
               "cpu": {
                   "measurement": [
                       "cpu_usage_idle",
                       "cpu_usage_iowait",
                       "cpu_usage_user",
                       "cpu_usage_system"
                   ],
                   "metrics_collection_interval": 60
               },
               "disk": {
                   "measurement": [
                       "used_percent"
                   ],
                   "metrics_collection_interval": 60,
                   "resources": [
                       "*"
                   ]
               },
               "mem": {
                   "measurement": [
                       "mem_used_percent"
                   ],
                   "metrics_collection_interval": 60
               }
           }
       }
   }
   EOF
   
   /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
       -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
   ```

4. **Click "Create launch template"**

![Screenshot 2025-06-01 210617](https://github.com/user-attachments/assets/1c37f20a-061f-446f-a0de-88985b9ec40e)

![Screenshot 2025-06-01 210807](https://github.com/user-attachments/assets/f314e7de-927d-4dfa-b63c-3447bc8f0cd8)


## Step 5: Create Application Load Balancer

### 5.1 Create Target Group First

1. **Navigate to Target Groups**
   - In EC2 Dashboard → Target Groups
   - Click "Create target group"

2. **Configure Target Group**
   ```
   Basic configuration:
   - Choose a target type: Instances
   - Target group name: WebApp-TG
   - Protocol: HTTP
   - Port: 80
   - VPC: WebApp-vpc
   
   Health checks:
   - Health check protocol: HTTP
   - Health check path: /
   - Advanced health check settings:
     - Port: Traffic port
     - Healthy threshold: 2
     - Unhealthy threshold: 5
     - Timeout: 5 seconds
     - Interval: 30 seconds
     - Success codes: 200
   ```

3. **Click "Next" → Skip target registration → "Create target group"**
![Screenshot 2025-06-01 211102](https://github.com/user-attachments/assets/ee0dee11-3783-4be4-9750-90079233aa52)
![Screenshot 2025-06-01 211252](https://github.com/user-attachments/assets/a8524c94-1463-40d5-9073-0752af643670)


### 5.2 Create Application Load Balancer

1. **Navigate to Load Balancers**
   - In EC2 Dashboard → Load Balancers
   - Click "Create Load Balancer"

2. **Select Application Load Balancer**
   - Click "Create" under Application Load Balancer

3. **Configure Load Balancer**
   ```
   Basic configuration:
   - Load balancer name: WebApp-ALB
   - Scheme: Internet-facing
   - IP address type: IPv4
   
   Network mapping:
   - VPC: WebApp-vpc
   - Mappings: Select both availability zones
     - us-east-1a: WebApp-subnet-public1-us-east-1a
     - us-east-1b: WebApp-subnet-public2-us-east-1b
   
   Security groups:
   - Remove default security group
   - Select: ALB-SG
   
   Listeners and routing:
   - Protocol: HTTP
   - Port: 80
   - Default action: Forward to WebApp-TG
   ```

4. **Click "Create load balancer"**

5. **Wait for Load Balancer to become "Active"** (may take 2-3 minutes)

![Screenshot 2025-06-01 211512](https://github.com/user-attachments/assets/e5e8e6b9-811b-4876-bc57-c3e8f4cbfcae)
![Screenshot 2025-06-01 214220](https://github.com/user-attachments/assets/b3f7ec24-1167-4967-b930-4b45d57c3bbf)

## Step 6: Create Auto Scaling Group

### 6.1 Create Auto Scaling Group

1. **Navigate to Auto Scaling Groups**
   - In EC2 Dashboard → Auto Scaling Groups
   - Click "Create Auto Scaling group"

2. **Step 1: Choose launch template**
   ```
   - Auto Scaling group name: WebApp-ASG
   - Launch template: WebApp-LaunchTemplate
   - Version: Latest
   - Click "Next"
   ```
   ![Screenshot 2025-06-01 214534](https://github.com/user-attachments/assets/6aaf41dd-310f-41a3-acf0-0673e3c0608c)


3. **Step 2: Choose instance launch options**
   ```
   Network:
   - VPC: WebApp-vpc
   - Availability Zones and subnets:
     ✓ us-east-1a | WebApp-subnet-public1-us-east-1a
     ✓ us-east-1b | WebApp-subnet-public2-us-east-1b
   
   Instance type requirements: Keep default
   Click "Next"
   ```
![Screenshot 2025-06-01 214555](https://github.com/user-attachments/assets/3276a1b1-6102-46c1-b86e-0b4cbcce0385)
![Screenshot 2025-06-01 214624](https://github.com/user-attachments/assets/b3f3d6cd-e6bf-41e9-a927-0a5d3d2c7353)


4. **Step 3: Configure advanced options**
   ```
   Load balancing:
   ✓ Attach to an existing load balancer
   - Choose from your load balancer target groups: WebApp-TG
   
   Health checks:
   ✓ Turn on Elastic Load Balancing health checks
   - Health check grace period: 300 seconds
   
   Additional settings: Keep defaults
   Click "Next"
   ```
![Screenshot 2025-06-01 214646](https://github.com/user-attachments/assets/93693e85-e260-4374-9706-5c8b8bf240dc)
5. **Step 4: Configure group size and scaling policies**
   ```
   Group size:
   - Desired capacity: 2
   - Minimum capacity: 2
   - Maximum capacity: 6
   
   Scaling policies:
   ✓ Target tracking scaling policy
   - Policy name: WebApp-ScalingPolicy
   - Metric type: Average CPU Utilization
   - Target value: 70
   - Instance warmup: 300 seconds
   
   Instance scale-in protection: Keep unchecked
   Click "Next"
   ```
![Screenshot 2025-06-01 214754](https://github.com/user-attachments/assets/502d60cc-8b4c-4e97-a040-45352cfb208e)

6. **Step 5: Add notifications**
   ```
   ✓ Add notification
   - SNS Topic: Create new topic
   - Topic name: WebApp-Notifications
   - Recipients: your-email@example.com
   - Event types: Select all
   Click "Next"
   ```
![Screenshot 2025-06-01 214842](https://github.com/user-attachments/assets/a30d1a8f-8156-4ef6-ae70-add4847aa3f1)

7. **Step 6: Add tags**
   ```
   Add tag:
   - Key: Name
   - Value: WebApp-ASG-Instance
   ✓ Tag new instances
   Click "Next"
   ```

8. **Step 7: Review** → Click "Create Auto Scaling group"

![Screenshot 2025-06-01 214919](https://github.com/user-attachments/assets/c3a6886e-6a6c-40c9-abad-25906a45b20c)

## Step 7: Set Up CloudWatch Monitoring

### 7.1 Create CloudWatch Dashboard

1. **Navigate to CloudWatch**
   - Go to AWS Console → Services → CloudWatch
   - Click "Dashboards" → "Create dashboard"

2. **Configure Dashboard**
   ```
   - Dashboard name: WebApp-Dashboard
   - Click "Create dashboard"
   ```

3. **Add Widgets**
   
   **Widget 1: EC2 CPU Utilization**
   ```
   - Widget type: Line
   - Metrics: EC2 → By Auto Scaling Group → WebApp-ASG → CPUUtilization
   - Statistic: Average
   - Period: 5 minutes
   ```
   
   **Widget 2: ALB Request Count**
   ```
   - Widget type: Line  
   - Metrics: ApplicationELB → Per AppELB Metrics → RequestCount
   - Select your load balancer
   ```
   
   **Widget 3: Healthy/Unhealthy Hosts**
   ```
   - Widget type: Line
   - Metrics: ApplicationELB → Per TG Metrics → HealthyHostCount & UnHealthyHostCount
   - Select your target group
   ```

### 7.2 Create CloudWatch Alarms

1. **Navigate to Alarms**
   - In CloudWatch → Alarms → All alarms
   - Click "Create alarm"

2. **Create High CPU Alarm**
   ```
   Step 1: Specify metric
   - Browse: EC2 → By Auto Scaling Group
   - Select: WebApp-ASG → CPUUtilization
   - Statistic: Average
   - Period: 5 minutes
   
   Step 2: Specify conditions
   - Threshold type: Static
   - Condition: Greater than 80
   
   Step 3: Configure actions
   - Alarm state trigger: In alarm
   - SNS topic: WebApp-Notifications
   - Email: your-email@example.com
   
   Step 4: Add name and description
   - Alarm name: WebApp-HighCPU
   - Description: Alarm when CPU exceeds 80%
   ```

3. **Create Unhealthy Hosts Alarm**
   ```
   Step 1: Specify metric
   - Browse: ApplicationELB → Per Target Group Metrics
   - Select: UnHealthyHostCount for WebApp-TG
   
   Step 2: Specify conditions
   - Threshold type: Static
   - Condition: Greater than or equal to 1
   
   Step 3: Configure actions
   - SNS topic: WebApp-Notifications
   
   Step 4: Add name
   - Alarm name: WebApp-UnhealthyHosts
   ```

![CloudWatch Setup](https://via.placeholder.com/800x300/FF5722/white?text=CloudWatch+Monitoring+Configured)

## Step 8: Testing the Application

### 8.1 Access Your Application

1. **Get Load Balancer DNS Name**
   - Go to EC2 → Load Balancers
   - Select WebApp-ALB
   - Copy the DNS name from the Description tab

2. **Test Application**
   - Open browser and navigate to: `http://your-alb-dns-name`
   - You should see your web application
   - Refresh the page multiple times to see different instance information

### 8.2 Test Auto Scaling

1. **Trigger CPU Load**
   - Click the "Start CPU Stress Test" button on your web page
   - Wait 5-10 minutes
   - Check Auto Scaling Group for new instances

2. **Monitor Scaling Events**
   - Go to EC2 → Auto Scaling Groups → WebApp-ASG
   - Check "Activity" tab for scaling activities
   - Check "Monitoring" tab for metrics

### 8.3 Test High Availability

1. **Terminate an Instance**
   - Go to EC2 → Instances
   - Select one of your WebApp instances
   - Actions → Instance State → Terminate

2. **Observe Auto Scaling**
   - Auto Scaling Group should launch a new instance
   - Load Balancer should stop sending traffic to terminated instance
   - Application should remain available

![Testing Complete](https://via.placeholder.com/800x300/8BC34A/white?text=Application+Testing+Successful)

## 🔍 Troubleshooting Tips

### Common Issues and Solutions

**Issue 1: Instances not appearing in Target Group**
```
Solutions:
1. Check security group allows traffic on port 80
2. Verify instances are in the same VPC as target group
3. Check user data script for errors in /var/log/cloud-init-output.log
4. Ensure web server is running: sudo systemctl status httpd
```

**Issue 2: Auto Scaling not working**
```
Solutions:
1. Check CloudWatch metrics are being published
2. Verify scaling policies are correctly configured
3. Check if instances are in cooldown period
4. Review Auto Scaling Group activity history
```

**Issue 3: Load Balancer returning 502/503 errors**
```
Solutions:
1. Check target group health status
2. Verify security group rules allow ALB to reach instances
3. Ensure web server is responding on correct port
4. Check application logs for errors
```

**Issue 4: No CloudWatch metrics**
```
Solutions:
1. Verify IAM role has CloudWatch permissions
2. Check CloudWatch agent is installed and running
3. Review CloudWatch agent configuration
4. Check /var/log/amazon/amazon-cloudwatch-agent/ for errors
```

### Monitoring Commands (via SSH)

```bash
# Check web server status
sudo systemctl status httpd

# View web server logs
sudo tail -f /var/log/httpd/access_log
sudo tail -f /var/log/httpd/error_log

# Check CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
    -a query

# Monitor CPU usage
top
htop

# Check cloud-init logs
sudo tail -f /var/log/cloud-init-output.log
```

## 🧹 Clean Up Resources

To avoid charges, delete resources in this order:

1. **Delete Auto Scaling Group**
   - Set desired capacity to 0
   - Wait for instances to terminate
   - Delete the Auto Scaling Group

2. **Delete Launch Template**
   - EC2 → Launch Templates → Delete

3. **Delete Load Balancer**
   - EC2 → Load Balancers → Delete

4. **Delete Target Group**
   - EC2 → Target Groups → Delete

5. **Delete CloudWatch Resources**
   - Delete alarms and dashboard

6. **Delete Security Groups**
   - VPC → Security Groups → Delete (delete non-default ones)

7. **Delete VPC**
   - VPC → Your VPCs → Delete VPC

8. **Delete SNS Topic**
   - SNS → Topics → Delete

9. **Delete IAM Role**
   - IAM → Roles → Delete

---

🎉 **Congratulations!** You have successfully implemented a scalable web application using AWS Console. This setup demonstrates high availability, auto scaling, load balancing, and monitoring capabilities of AWS.
