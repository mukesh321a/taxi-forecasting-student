# Student Guide: Request Your Windows Course Image

## What You Are Setting Up

The Taxi Forecasting course provides a preconfigured Windows Amazon Machine
Image (AMI). An AMI is a template for creating your own virtual computer in AWS.
It includes Python, VS Code, course files and the main Python dependencies.

You will use **your own AWS account**. After your instructor approves access,
you can launch your own Windows computer from the shared image. Your files,
Windows password, AWS resources and bills belong to your account.

**To request access, send your instructor your 12-digit AWS account ID using the
template in Step 4. Do not send your AWS password, MFA codes or security keys.**

Creating an account does not automatically grant image access. Wait for the
instructor's confirmation before launching an instance.

## 1. Create Or Choose Your AWS Account

If you already have an active AWS account that you are authorized to use for
coursework, use it and continue to Step 2. Do not create another account just
for this guide unless you need a separate account.

If you do not have an account:

1. Open [Create an AWS account](https://portal.aws.amazon.com/billing/signup).
2. Enter an email address you control, choose an account name, and complete
   the email verification and password steps shown by AWS.
3. Complete the contact, payment and identity/phone verification steps requested
   by AWS. Enter payment information only on the official AWS website.
4. Review the current account-plan choices and pricing. This course requires
   billable Windows EC2 resources; do not assume a free plan, promotional credits
   or Free Tier covers the required instance. Ask your instructor before choosing
   or upgrading a plan if you are unsure. Paid support is not required by the course.
5. Wait for AWS to confirm account activation, then sign in to the AWS Console.

If you use an employer, university or AWS Academy account, ask its administrator
whether external encrypted AMIs, KMS permissions and Windows EC2 instances are
allowed. Restricted lab accounts may not support this workflow. Do not bypass
institutional restrictions or use an account without permission.

## 2. Secure The Account And Review Costs

- Enable multi-factor authentication (MFA) for the account's root user and keep
  recovery details safe.
- Use an administrator-approved IAM or IAM Identity Center identity for daily
  work. Reserve root access for account-management tasks that require it.
- **Do not create AWS access keys just to request or launch this image.** The
  browser-based AWS Console is sufficient for the steps in this guide.
- In **Billing and Cost Management > Budgets**, create a budget with email alerts
  at an amount you can afford. Ask your account administrator if billing access
  is restricted. Budget alerts are notifications, not spending caps.
- Review the current EC2, EBS storage and public IPv4 prices before launch.

The tested configuration is `m6i.xlarge` with 4 vCPUs, 16 GiB RAM and an 80 GiB
encrypted gp3 disk in **US East (N. Virginia), `us-east-1`**. For planning only,
the Windows On-Demand compute price checked on 20 September 2026 was about
**USD 0.376 per running hour**, excluding disk, public IPv4, taxes and other
charges. Ten running hours would be about USD 3.76 for compute alone. Prices and
account eligibility can change; check your own account before spending.

## 3. Find Your 12-Digit AWS Account ID

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/).
2. Open the account menu at the top right.
3. Find **Account ID** and copy the 12-digit number.
4. If the number is displayed with hyphens, remove the hyphens before sending it.
   Preserve any leading zeroes.
5. Confirm this is the account where you intend to run and pay for the course
   instance, especially if you can switch between multiple accounts.

Your account ID is **not** your email address, IAM username, account alias,
credit-card number or an access key beginning with `AKIA` or `ASIA`.

An account ID is an identifier, not a password. It lets the instructor grant
your account permission to use the course image; it does not let the instructor
sign in to or administer your account. Still, send it through the instructor's
private email or learning-platform message, not a public class discussion.

Official reference: [Find your AWS account ID](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-identifiers.html).

## 4. Send This Access Request To Your Instructor

Replace the bracketed fields. Send this message privately using the course's
normal instructor contact channel. If you use a spreadsheet, format the account
ID column as text so leading zeroes are not removed.

```text
Subject: Taxi Forecasting - Windows image access request

Name: [Your name]
Course / batch: [Your course or batch]
Course email: [Your email]
AWS account ID: [Your 12-digit account ID]
Requested region: us-east-1 (N. Virginia)
Account type: [Personal / university-managed / employer-managed / other]

I am authorized to use this account for the course.
I understand that resources launched in my account may incur AWS charges.
I will stop my instance when not in use and delete test resources when finished.

Please grant my account access to the Windows course image and confirm when
the image and its encryption-key permissions are ready.
```

**Do not attach or send any of the following:**

- AWS or email passwords, MFA codes, recovery codes or session cookies.
- AWS access keys, secret access keys, session tokens or credentials files.
- EC2 private keys (`.pem` / `.ppk`) or Windows Administrator passwords.
- Payment details or screenshots containing personal/security information.

You do not need to invite the instructor as an administrator, share your account
login, create an IAM user for the instructor, or pay for a running instance just
to request image access.

## 5. Wait For Access Confirmation

The instructor must grant access to **both the private AMI and its encryption
key**. Your account's launching identity must also have the necessary EC2 and
KMS permissions. If your account is managed, its administrator may need to apply
the instructor-provided, narrowly scoped permissions.

The instructor will confirm the AMI ID, owner account, region and launch details.
Do not launch an unrelated public image with a similar name. An AMI link alone
does not grant permission, and the image is only visible in the shared region.

The instructor may test access with one student first before adding the rest of
the class. Sharing the image does not share the instructor's live Windows
instance, GitHub access, AWS credentials or already-trained models.

## 6. Launch After Approval

1. Sign in to **your own account** and select **US East (N. Virginia), `us-east-1`**.
2. Open **EC2 > AMIs**, select **Private images**, and search for the AMI ID sent
   by the instructor. Check that the owner matches the instructor's confirmation.
3. Choose **Launch instance from AMI**. Use the instructor's approved size and
   disk settings; the tested configuration is `m6i.xlarge` and 80 GiB encrypted gp3.
4. Create your **own RSA key pair**, download its private key securely and keep
   it on your computer. Never reuse the instructor's key or send yours to anyone.
5. Follow the instructor's network and connection instructions. Each account
   needs its own network, security group and any required instance role. These
   settings are not supplied by the AMI itself.
6. Require IMDSv2, set instance-initiated shutdown behavior to **Stop**, and check
   **Delete on termination** for the root disk. Review the cost before launching.
7. Wait for the instance health checks to pass, then connect using the agreed
   method and open the **Taxi Course** desktop shortcut.

For browser-based **Fleet Manager Remote Desktop**, your instance needs an EC2
role with `AmazonSSMManagedInstanceCore`, outbound connectivity to Systems
Manager, and your Console identity needs the appropriate Fleet Manager/SSM
permissions. Select your private key only in the official AWS connection flow.
No public inbound RDP rule is needed. Direct RDP, if approved instead, must be
restricted to your own current public IP `/32`, never `0.0.0.0/0`.

In the remote VS Code terminal, run:

```powershell
Set-Location C:\taxi-forecasting-student
.\.venv\Scripts\python.exe --version
.\.venv\Scripts\python.exe -m pip check
.\.venv\Scripts\python.exe -m pytest
```

Expect Python 3.11.9, no broken requirements and passing tests (120 in the
initial image). The environment is preinstalled: **do not delete or recreate
`.venv` during first setup**. Follow the setup and lab documents in the course
folder to generate data, train models and select the project notebook kernel.
Open localhost API/dashboard URLs in a browser inside the remote desktop.

Docker Desktop is not supported on Windows Server. The container lab needs a
separate supported environment. The bundled course folder is a source snapshot,
not a Git checkout; Git/CI work requires your own authorized repository clone.

## 7. Stop Costs And Ask For Help Safely

- Closing VS Code, the browser or Remote Desktop **does not stop the instance**.
  Use **EC2 > Instances > Instance state > Stop instance** after each session and
  verify that its state becomes **Stopped**.
- Stopped instances still incur storage charges. Other retained resources may
  also incur charges. Budget alerts do not automatically stop a machine.
- **The shared image does not include an automatic stop schedule.** Set a
  reminder or an approved shutdown schedule for each session; do not rely on
  the instructor's temporary test-instance schedule.
- At the end of the course, back up work you need, terminate the instance and
  confirm that unwanted volumes and other chargeable resources are deleted.
  Termination can permanently delete files on the root disk.
- If the AMI is missing, check your account ID, region and the **Private images**
  filter before contacting the instructor.
- If launch reports a KMS/permission error or the instance terminates immediately,
  stop retrying and ask for a permission check. If a quota error appears, your
  account may need an approved regional vCPU quota increase; do not switch to a
  more expensive size as a workaround.

For help, send your account ID privately, region, instance ID (if one exists),
the step that failed and the exact non-sensitive error message. Redact secrets,
payment information and unrelated personal details from screenshots. Never send
credentials or private key files to troubleshoot a connection.
