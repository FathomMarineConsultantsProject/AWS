

# Day 3: Amazon EC2 – A Deep Dive & Deploying Jenkins on AWS

Amazon EC2 (Elastic Compute Cloud) is where most AWS adventures truly begin. It’s the service that lets you rent virtual computers in the cloud and run virtually any application on them. In this guide, we’ll break EC2 down using a simple “renting a computer” analogy, explore its core building blocks, and then walk through a real project: deploying the Jenkins automation server.

---

## The “Renting a Computer” Analogy for EC2

Imagine you need a powerful computer for a short project—maybe to host a website, run a machine learning model, or test new software. Instead of buying an expensive machine, you go to a provider that rents computers by the hour. That’s exactly what EC2 does, only the computers live in Amazon’s giant data centers.

Let’s map the analogy:

- **Amazon EC2** → The rental service itself, available on-demand, 24/7.
- **An EC2 Instance** → One specific virtual computer you’ve rented.
- **AMI (Amazon Machine Image)** → The installation disc that determines which operating system and software come preloaded (Windows, Ubuntu, Red Hat, etc.).
- **Instance Type** → The hardware specs of the computer: CPU, RAM, storage, network speed.
- **Security Group** → A personal security guard standing at the computer’s door, checking every connection attempt and only letting approved visitors in.
- **Key Pair** → The digital key you use to log in securely, instead of a password.

Just like renting a physical computer, you can return (terminate) the EC2 instance when you’re done and stop paying.

---

## Deep Dive: The Core Components of EC2

When you launch an EC2 instance in the AWS Console, you’re effectively building a server step by step. Let’s look at each piece in detail.

### 1. Amazon Machine Image (AMI) – The Operating System & Software Setup

An AMI is a template that contains the operating system, any pre-installed applications, and configuration settings needed to launch a virtual server. Think of it as the “restore point” or “installation DVD” that AWS uses to create your computer.

- **Choosing an OS:** AWS provides free AMIs with common Linux distributions (Amazon Linux 2, Ubuntu, Red Hat Enterprise Linux) and Windows Server. For learning, Ubuntu and Amazon Linux are excellent starting points.
- **Pre‑configured AMIs:** The AWS Marketplace offers AMIs that come with popular software stacks already installed. Need a WordPress blog or a LAMP stack? You can launch an instance that has it ready to go, saving you manual setup time.
- **Custom AMIs:** Once you’ve set up an EC2 instance exactly the way you like it—installed your tools, tweaked the firewall, customized performance—you can save it as your own private AMI. This becomes your golden image, allowing you to spin up identical copies whenever you need more capacity, ensuring consistency and speed.

> **Pro tip:** Always note the AMI ID you use. If you ever need to launch the same kind of instance in another region, you can search for that AMI (or a copy of it) there.

### 2. Instance Types – Picking the Right Hardware

The instance type determines the hardware of your rented computer: how many virtual CPUs (vCPUs), how much RAM, what kind of storage, and the network performance. AWS offers a vast selection grouped into families optimized for different workloads.

- **General Purpose (T‑family)** – Balanced CPU and memory. Perfect for web servers, small applications, and most learning exercises. The `t2.micro` and `t3.micro` types are free‑tier eligible, making them ideal for this tutorial.
- **Compute Optimized (C‑family)** – High‑performance processors for compute‑intensive tasks like scientific modelling, batch processing, or game servers.
- **Memory Optimized (R‑family)** – Lots of RAM for memory‑hungry applications like large in‑memory databases or real‑time big data analytics.
- **Accelerated Computing (P, G, F families)** – Instances with GPUs or FPGAs for machine learning, graphics rendering, or financial computations.

The beauty of EC2 is that you can start with a small instance for testing and then seamlessly switch to a more powerful one when you go live, with just a few clicks (or an API call).

> **Free tier tip:** Stick with `t2.micro` (or `t3.micro`) to stay within the AWS free tier for your first year.

### 3. Key Pairs – Secure Login Without Passwords

When you launch a Linux EC2 instance, you don’t set a password. Instead, you use public‑key cryptography via SSH (Secure Shell). AWS generates a **key pair**:

- **Public Key** – Stored inside the EC2 instance automatically.
- **Private Key** (the `.pem` file) – Downloaded to your local computer once, at creation time. **You must keep this file safe and secure.** It acts as the sole credential for initial access.

To connect from your terminal, you’ll run a command like:

```bash
ssh -i /path/to/your-key.pem ubuntu@<public-ip-of-instance>
```

The SSH client uses your private key to prove you own the matching public key. If you lose the private key, AWS cannot recover it for you (because it never stores it). Your only option is to stop the instance, detach its boot volume, attach it to another instance as a secondary disk, and manually recover your data—a painful process. So, store your `.pem` file in a secure place and consider setting a strong file permission (`chmod 400 your-key.pem`) to prevent accidental exposure.

### 4. Security Groups – The Virtual Firewall

A **Security Group** acts as a stateful firewall that controls traffic to and from your EC2 instance. It’s a simple but powerful tool for locking down your server.

- **Default behaviour:** All inbound traffic is denied; all outbound traffic is allowed.
- **Rules are permissive only:** You can only create *allow* rules. There is no “deny” rule—if a port or IP is not explicitly allowed, it’s automatically blocked.
- **Stateful:** If you allow incoming traffic on a port (e.g., port 22 for SSH), the response traffic is automatically allowed, regardless of outbound rules. This makes management intuitive.

**Common port rules for web applications:**
| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH (secure terminal access) |
| 80 | TCP | HTTP (unencrypted web) |
| 443 | TCP | HTTPS (encrypted web) |
| 8080 | TCP | Jenkins default web interface |

For our Jenkins deployment, you’ll create a Security Group that opens **port 22** (so you can log in and configure the server) and **port 8080** (so you can reach the Jenkins dashboard from your browser). Optionally, you might restrict the source IP to your own home or office IP to reduce exposure.

> **Security best practice:** Avoid opening ports to `0.0.0.0/0` (everyone on the internet) unless absolutely necessary. For SSH, restrict it to your own IP address whenever possible.

---

## EC2 User Data – Automation at Boot Time

When launching an instance, you’ll find an “Advanced Details” section with a box labelled **User Data**. This is a golden feature for DevOps engineers.

- **What it does:** You can paste a shell script (for Linux) or PowerShell script (for Windows) into this box. That script will execute automatically, with root privileges, exactly once—the very first time the instance boots.
- **Why it’s powerful:** Instead of manually SSH‑ing in, updating packages, installing Node.js or Java, and cloning your repository step by step, you can write a script that does everything. By the time the instance is fully running, your application is already installed and started.

**Example User Data script (Ubuntu) that updates the system and installs Apache:**
```bash
#!/bin/bash
apt update -y
apt install -y apache2
systemctl enable apache2
systemctl start apache2
echo "<h1>Hello from EC2 User Data!</h1>" > /var/www/html/index.html
```
Just like that, your instance becomes a live web server right after launch—no manual intervention required.

> **Note:** User Data scripts only run at the very first boot. If you stop and start the instance later, the script does not run again. For recurring boot‑time tasks, you’d need to integrate something like cloud‑init scripts or a configuration management tool.

---

## Practical Project: Deploying Jenkins on EC2

Now let’s put theory into practice by setting up **Jenkins**, a popular open‑source automation server used for Continuous Integration and Continuous Delivery (CI/CD). We’ll walk through every step in a clear, repeatable way.

### Objective
Launch a fresh Ubuntu EC2 instance, install Jenkins, and access its web dashboard from your browser.

### Step 1: Launch an Ubuntu EC2 Instance
- **AMI:** Search for “Ubuntu Server 22.04 LTS” (or the latest LTS) and select the free‑tier eligible one.
- **Instance type:** Choose `t2.micro` (free tier).
- **Key pair:** Create a new key pair or use an existing one. Download the `.pem` file and keep it safe.
- **Network settings:** Allow SSH (port 22) and add a custom TCP rule for port **8080** (source `0.0.0.0/0` for learning, but restrict later for production).
- **Storage:** Leave the default 8 GiB (or increase if needed; free tier allows up to 30 GB of EBS General Purpose SSD).
- **Launch the instance** and wait for it to pass status checks.

### Step 2: Connect via SSH
Open a terminal (Linux/Mac) or use PowerShell/Windows Terminal with OpenSSH installed.

```bash
chmod 400 your-key.pem   # Only once, to set correct permissions
ssh -i your-key.pem ubuntu@<public-ip-of-instance>
```
Replace `<public-ip-of-instance>` with the Public IPv4 address shown in the AWS Console.

### Step 3: Install Java (Jenkins Dependency)
Jenkins is written in Java, so we must install a compatible Java Development Kit (JDK). **As of recent Jenkins releases, Java 17 is required.** Older tutorials may mention Java 8 or 11, but those are now deprecated for new Jenkins versions.

Run these commands inside your SSH session:

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version   # Verify installation (should show OpenJDK 17)
```

### Step 4: Install Jenkins
Add the official Jenkins repository and install the package:

```bash
# Add the Jenkins repository key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add the repository to sources
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install
sudo apt update
sudo apt install -y jenkins
```

### Step 5: Start Jenkins and Enable Autostart
Jenkins should start automatically after installation. Confirm it’s running:

```bash
sudo systemctl status jenkins
```
If it’s not active, start it with `sudo systemctl start jenkins`. Enable it to launch on boot:
```bash
sudo systemctl enable jenkins
```

### Step 6: Access the Jenkins Dashboard
Open your web browser and go to:
```
http://<public-ip>:8080
```
You’ll be greeted by an “Unlock Jenkins” page, asking for an initial administrator password. Retrieve it with:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the password, paste it into the web form, and proceed. Follow the setup wizard: install suggested plugins, create your first admin user, and configure the Jenkins URL. When finished, you’ll land on the Jenkins dashboard—ready to create your first CI/CD pipeline.

> **Troubleshooting tip:** If the page doesn’t load, double‑check that the Security Group allows port 8080 from your IP, and that Jenkins is actually listening (`sudo netstat -tlnp | grep 8080`).

---

## Key Takeaways & Best Practices

- **Understand the analogy:** EC2 instances are rented virtual computers. AMI = OS, Instance Type = hardware, Security Group = firewall, Key Pair = digital key.
- **Start small, scale later:** Use `t2.micro` for learning and seamlessly upgrade instance types when your workload grows.
- **Lock down access:** Never open ports wider than necessary. Use Security Groups effectively and restrict SSH to known IPs.
- **Automate from day one:** Use User Data to bootstrap instances, reducing manual steps and making your infrastructure repeatable.
- **Keep your private keys safe:** Lose your `.pem` file, and you lose access (for all practical purposes). Store it securely and back it up.
- **Stay current with dependencies:** Jenkins now demands Java 17; always check official documentation before installing to avoid version conflicts.

---

## What’s Next?

With Jenkins running on EC2, you’ve taken a major step toward building automated CI/CD pipelines. In subsequent days of the course, you’ll likely learn how to integrate Jenkins with source control (GitHub), run automated tests, and deploy applications to other AWS services. This hands‑on foundation makes those advanced topics much more approachable because you now understand the “computer” layer behind the scenes.

---

# How to Launch a General EC2 Instance (Step-by-Step)

We'll use the AWS Management Console because it's the most visual way to learn. After that, I'll briefly show how to do it with the AWS CLI.

---

## 1. Open the EC2 Dashboard and Start

- Log into your [AWS Console](https://console.aws.amazon.com/).
- Search for **EC2** in the top search bar and open it.
- Click the orange **Launch instance** button.

---

## 2. Name and Tags

- In the **Name** field, give your instance a descriptive name, like `my-first-server` or `web-server-dev`.
- Tags are optional but recommended. You can add extra tags (e.g., `Environment: Dev`, `Project: Learning`) to organise resources later.

---

## 3. Choose an Amazon Machine Image (AMI)

The AMI is your starting point—the operating system and any pre-installed software.

- Scroll through the **Quick Start** list. You'll see free-tier eligible options clearly marked.
- **Common choices:**
  - **Amazon Linux 2023 AMI** – AWS's own Linux, optimised for EC2, good for general purpose and tutorials.
  - **Ubuntu Server 22.04 LTS** – Very popular, huge community support.
  - **Red Hat Enterprise Linux** or **SUSE Linux** – More enterprise-oriented.
  - **Windows Server** – If you need a Windows environment.
- Click **Select** on the AMI you want. Make sure it says **Free tier eligible** if you're using the free tier.

> If you need a specific software stack (e.g., WordPress, LAMP, Deep Learning), switch to the **AWS Marketplace** tab and search for an AMI that already includes it.

---

## 4. Select an Instance Type

This decides how much virtual CPU, memory, and network performance you get.

- For learning and light workloads, choose **t2.micro** or **t3.micro**. Both are free-tier eligible.
- Use the **Filter by:** dropdown and select **Free tier only** to see only those options.
- For production or heavier tasks, you'd later pick types from the General Purpose, Compute Optimised, or Memory Optimised families.

---

## 5. Key Pair (Login Credential)

For **Linux** instances, you'll need a key pair to SSH in. For **Windows**, you'll still need a key pair to decrypt the Administrator password.

- If you already have a key pair, select it from the dropdown.
- If not, click **Create new key pair**:
  - **Key pair name:** e.g., `my-ec2-key`
  - **Key pair type:** **RSA** (most compatible)
  - **Private key file format:**  
    - `.pem` for Linux/Mac or Windows with OpenSSH  
    - `.ppk` if using PuTTY on Windows
- The private key file will download. **Store it securely** – you can't download it again. Set permissions with `chmod 400 my-ec2-key.pem` on Linux/Mac.

---

## 6. Network Settings (Security Group – Firewall)

Think of this as the firewall around your instance. By default, no one can connect.

- Click **Edit** to expand the network settings.
- **Auto-assign public IP:** Set to **Enable** so you can reach the instance from the internet.
- **Firewall (security groups):** Choose **Create security group**.
- Give it a name, e.g., `general-sg`.

Now add rules for the traffic you want to allow.

### Common Inbound Rules:

| Type | Protocol | Port | Purpose | Source (recommended) |
|------|----------|------|---------|----------------------|
| SSH | TCP | 22 | Terminal access for Linux | **My IP** (your current IP) |
| RDP | TCP | 3389 | Remote Desktop for Windows | **My IP** |
| HTTP | TCP | 80 | Web server (unencrypted) | **Anywhere** (0.0.0.0/0) |
| HTTPS | TCP | 443 | Web server (encrypted) | **Anywhere** (0.0.0.0/0) |

- Click **Add security group rule** for each additional port.
- You can always change these rules later without stopping the instance.

> **Pro tip:** For SSH/RDP, using **My IP** automatically fills in your current public IP. This limits exposure. If your IP changes, you'll need to update the rule.

---

## 7. Configure Storage (Hard Disk)

- The default root volume is usually 8 GiB (gp3 SSD). The free tier allows up to 30 GB of EBS General Purpose SSD.
- If you need more space or want to add extra volumes, click **Add new volume**.
- For a general purpose Linux server, the default is fine. For Windows, you may want to increase it to 30 GiB (Windows needs more space).

You can also enable **Delete on termination** (default for the root volume) so the storage is cleaned up when you terminate the instance.

---

## 8. Advanced Details (Optional)

Expand the **Advanced details** section if you want to:

- **IAM instance profile:** Attach an IAM role to give the instance permissions (e.g., access to S3) without storing credentials on the server.
- **User data:** Paste a script (shell for Linux, PowerShell for Windows) that runs on first boot. For example, a script to install a web server:
  ```bash
  #!/bin/bash
  yum update -y
  yum install -y httpd
  systemctl enable httpd
  systemctl start httpd
  echo "Hello World" > /var/www/html/index.html
  ```
  This automates setup so your server is ready immediately.

For a basic test instance, you can skip all of this for now.

---

## 9. Review and Launch

- Scroll down and click **Launch instance**.
- After a few seconds, you'll see a success banner. Click **View all instances** to go to your instances list.

---

## 10. Find Your Instance and Connect

- Select your new instance. In the details pane, look for:
  - **Instance state:** Should be **Running**
  - **Status check:** Should eventually show **2/2 checks passed**
  - **Public IPv4 address:** This is your server's public IP (e.g., `54.123.45.67`). Copy it.

### To connect (Linux via SSH):
From a terminal, run:
```bash
ssh -i /path/to/your-key.pem ec2-user@<public-ip>   # for Amazon Linux
ssh -i /path/to/your-key.pem ubuntu@<public-ip>     # for Ubuntu
```
The default username depends on the AMI:
- Amazon Linux: `ec2-user`
- Ubuntu: `ubuntu`
- RHEL: `ec2-user` or `root`
- SUSE: `ec2-user`

### To connect (Windows via RDP):
1. In the console, select the instance, click **Connect** → **RDP client**.
2. Click **Get password** and upload your private key to decrypt the Administrator password.
3. Use Remote Desktop with the public IP and the decrypted password.

---

## Quick: Launch via AWS CLI

If you prefer the command line, a minimal launch command looks like this:

```bash
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --instance-type t2.micro \
    --key-name my-ec2-key \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --associate-public-ip-address \
    --count 1
```

- Replace `ami-0abcdef...` with a current AMI ID for your region (you can grab it from the console or `aws ec2 describe-images`).
- The security group and subnet must already exist.
- Add `--user-data file://script.sh` if you want to bootstrap.

---

## Wrapping Up

You now know how to launch **any** EC2 instance. The process is the same whether you're deploying a simple test server, a web application, or a complex multi-service setup. The choices you make (AMI, instance type, security groups, user data) simply adapt to the workload.

Next time you hear "spin up an EC2 instance," you can do it confidently in under two minutes. Happy building!
