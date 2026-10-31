
# ExpenseTracker: Serverless Expense Management with Receipt Scanning (AWS)

> **Course**: CSCI 5409 – Advanced Topics in Cloud Computing  
> **Tech**: AWS **Lambda**, **API Gateway**, **DynamoDB**, **S3**, **SNS**, **EventBridge**, **Textract** · **React (Vite + Tailwind)** · **Docker** · **CloudFormation**

## Overview
**ExpenseTracker** is a full‑stack, serverless application that lets users register/login, upload or enter expenses, and visualize monthly summaries. The backend is built on AWS managed services, and the frontend is a React single‑page app that can be containerized and hosted on EC2.

**AI Capability:** Uploaded receipt images are processed using **AWS Textract** (managed machine learning for intelligent document processing). Textract extracts line items/total amounts, enabling **AI‑assisted expense capture and categorization** with minimal manual entry.

### Key Features
- **User auth & notifications**: Register/login, password reset and change flows; **SNS** notifications for events (e.g., expense added, monthly reports).
- **Add expense** in two ways:
  - Enter **amount** manually, or
  - **Upload receipt image** → **Textract** extracts the **total** and stores the image in **S3**.
- **Current & previous months**: Fetch current-month totals and historical (previous months) expenses from **DynamoDB**.
- **Monthly reports**: A scheduled Lambda (triggered by **EventBridge**) computes month summaries and sends users a report via **SNS**.
- **IaC**: **CloudFormation** templates provision the Lambdas, API Gateway, SNS topic and EventBridge schedule, plus a Docker‑ready EC2 instance for the frontend.
- **Dev UX**: Frontend built with **Vite + React + Tailwind**; single `dockerfile` to build and serve static assets.

---

## Architecture

```
React (Vite + Tailwind)  -->  API Gateway  -->  Lambda Functions  -->  DynamoDB
        |                         |                 |                   |
        |                         |                 +--> S3 (receipt images)
        |                         |                 +--> Textract (OCR/amount extraction)
        |                         |                 +--> SNS (notifications)
        |                         |
        +--> Docker on EC2  <-----+          EventBridge -> Monthly Report Lambda
```

**Core AWS resources**
- **API Gateway (REST)** — Entry point for the SPA to call backend APIs.
- **Lambda (Node.js 20)** — Business logic:
  - `Register-lambda`: user signup (+ SNS subscription)
  - `login-lambda`: user login
  - `Reset-Password`: password reset flow
  - `change-password`: change existing password
  - `Upload-expense`: add expense by **amount** or **receipt image** (S3 upload + **Textract** → total extraction)
  - `Get-Current-Month-Expense`: compute current month totals
  - `Get-Previous-Expense`: fetch historical expense data
  - `Send-Reports-lambda`: **EventBridge**-triggered monthly report via **SNS**
- **DynamoDB** — Table: `Term-Project-Expense-Table` (PK: `useremail`) with nested month maps for expenses.
- **S3** — Bucket for receipt images: `term-project-expense-tracker-bucket` (path: `/{email}/{YYYY-MM}/{expenseName}.jpg`).
- **SNS** — Topic for notifications (subscriptions managed during registration).
- **EventBridge** — Schedule to trigger monthly report Lambda.
- **IAM** — Roles/policies for Lambda to access DynamoDB, S3, Textract, SNS, and logs.

---

## Repository Layout

```
IAC/
  cloudFormation.yaml            # Backend stack (Lambdas, DynamoDB, SNS, EventBridge, etc.)
  cloudFormationFrontEnd.yaml    # EC2 + Security Group to host the frontend Docker container

lambda-functions/
  Upload-expense/index.js        # S3 upload + Textract AnalyzeDocument + DynamoDB update + SNS notify
  Get-Current-Month-Expense/...  # Read current month totals from DynamoDB
  Get-Previous-Expense/...       # Fetch historical expenses
  Register-lambda/...            # Register + subscribe to SNS topic
  login-lambda/...               # Authenticate user
  Reset-Password/...             # Password reset flow
  change-password/...            # Change password
  Send-Reports-lambda/...        # Scheduled monthly summary + SNS email

CSCI5409-Term-Project/           # React SPA (Vite + Tailwind)
  src/
    pages/                       # Login, Register, AddExpense, GetCurrentMonthExpense, GetAllPreviousExpense, etc.
    Components/                  # Navbar, Sidebar, ExpenseCard, etc.
    AuthContext/                 # Simple context/reducer for auth state
    config/backendURL.js         # <-- Set API Gateway base URL here
  dockerfile                     # Containerize SPA and serve with `serve -s dist -l 3000`
```

---

## Prerequisites
- **AWS Account** with permissions to deploy CloudFormation stacks, Lambda, API Gateway, DynamoDB, S3, SNS, EventBridge, and EC2.
- **IAM Role ARN** for Lambda execution (update in `IAC/cloudFormation.yaml`).
- **DynamoDB tables and S3 bucket created beforehand.** The CloudFormation template does not create them, and the Lambdas expect these names:
  - `Term-Project-Application-Users` (partition key `useremail`): user accounts
  - `Term-Project-Expense-Table` (partition key `useremail`): expenses
  - `Term-Project-Reset-Password-Code` (partition key `useremail`): password-reset codes
  - S3 bucket `term-project-expense-tracker-bucket`: receipt images. Bucket names are global, so you may need to change it in the Lambda code.
- **EC2 AMI ImageId** and **KeyPair** for the frontend stack (`IAC/cloudFormationFrontEnd.yaml`).
- **Node.js 18+** and **npm** locally (if you want to build the frontend).

---

## Deployment

### 1) Backend (CloudFormation)
1. Open **`IAC/cloudFormation.yaml`** and update the placeholders:
   - `Role: YourRoleArn`
   - Any placeholder values (e.g., SNS topic, if required).
2. Deploy the stack (Console or CLI). Example (CLI):
   ```bash
   aws cloudformation deploy      --template-file IAC/cloudFormation.yaml      --stack-name expense-tracker-backend      --capabilities CAPABILITY_NAMED_IAM
   ```
3. After it completes, note:
   - **API Gateway Base URL**
   - **SNS Topic ARN** (if needed)
   - **Deployed Lambda names**

> The Lambdas use environment variable **`SNS_ARN`** for notifications; the template wires this for you.

### 2) Frontend (CloudFormation + Docker on EC2)
1. Open **`IAC/cloudFormationFrontEnd.yaml`** and set:
   - `ImageId: YourImageId`
   - `KeyName: ec2-key` (replace with your keypair)
2. Deploy:
   ```bash
   aws cloudformation deploy      --template-file IAC/cloudFormationFrontEnd.yaml      --stack-name expense-tracker-frontend
   ```
3. SSH into the EC2 instance and deploy the containerized SPA:
   ```bash
   # On your laptop (in CSCI5409-Term-Project/)
   scp -i your-key.pem -r CSCI5409-Term-Project ec2-user@EC2_PUBLIC_DNS:/home/ec2-user/
   ssh -i your-key.pem ec2-user@EC2_PUBLIC_DNS

   # On the EC2 instance:
   cd /home/ec2-user/CSCI5409-Term-Project
   # Set backend URL for the SPA
   sed -i 's|API Gateway URL|https://your-api-id.execute-api.us-east-1.amazonaws.com/prod|' src/config/backendURL.js
   docker build -t expense-tracker-spa .
   docker run -d --name expense-spa -p 3000:3000 expense-tracker-spa
   ```
4. Open `http://EC2_PUBLIC_DNS:3000` in your browser.

### 3) Local Frontend (optional)
```bash
cd CSCI5409-Term-Project
npm install
# Set API base URL
sed -i 's|API Gateway URL|https://your-api-id.execute-api.us-east-1.amazonaws.com/prod|' src/config/backendURL.js
npm run dev
```

---

## API Endpoints

All endpoints are `POST` routes on the API Gateway `prod` stage, and each has a CORS `OPTIONS` method:

| Path | Lambda | Purpose |
|---|---|---|
| `/register` | `Register-lambda` | Create an account and subscribe the email to SNS notifications |
| `/login` | `login-lambda` | Authenticate a user |
| `/reset-password` | `Reset-Password` | Email a numeric reset code |
| `/change-password` | `change-password` | Set a new password using the code |
| `/add-expense` | `Upload-expense` | Add an expense by amount, or by receipt image (S3 + Textract) |
| `/get-current-expense` | `Get-Current-Month-Expense` | Current month's expenses and total |
| `/get-all-expense` | `Get-Previous-Expense` | Expenses from previous months |

**Scheduled:** an EventBridge rule (`cron(40 14 13 * ? *)`, on the 13th of each month) runs `Send-Reports-lambda`, which emails monthly summaries through SNS.

When a receipt image is uploaded, the API stores it in **S3**, runs **Textract `AnalyzeDocument`** (TABLES and FORMS) to extract the total, saves the expense in **DynamoDB**, and sends an **SNS** notification.

---

## Data Model (DynamoDB)
- **Users** (`Term-Project-Application-Users`): email, SHA-256 password hash, and SNS subscription status
- **Expenses** (`Term-Project-Expense-Table`), partition key `useremail`:
  - `expenses` (map): `{ "YYYY-MM": [ { expenseName, amount, expenseImageUrl } ] }`, a list of expenses for each month


---

## Security & Configuration Notes
- Replace **placeholder values** in CloudFormation (Role ARN, AMI ImageId, KeyName).
- Use **IAM least privilege** for the Lambda role (DynamoDB, S3, Textract, SNS).
- Consider **AWS Secrets Manager/SSM Parameter Store** for secrets (rather than inline env vars).
- S3 bucket must be **globally unique**; consider bucket policies for least-privilege access.
- Validate and size‑limit uploaded images; prefer presigned URLs for uploads at scale.

---

## Cost Considerations
- **AWS Free Tier** covers small usage; **Textract** is a paid service — costs can accrue with many images.
- EventBridge scheduled invocations, DynamoDB read/write capacity, and S3 storage may incur charges.
- Use CloudWatch metrics/alarms to monitor usage and set budgets.

---

## Running Tests
There are no automated tests in this repository. The API was tested manually with Postman/curl against the deployed endpoints.

---

## Roadmap Ideas
- Categorization model (rule‑based + ML) to auto‑tag expenses.
- Export reports (CSV/PDF) and email via SES.
- Multi‑tenant auth via Cognito with JWTs.
- Replace manual image upload with presigned S3 URLs.
- Infrastructure as code: create the DynamoDB tables and S3 bucket in the template, and pass the API URL to the frontend automatically.

---

## About
Built as a course project for CSCI 5409 (Advanced Topics in Cloud Computing) at Dalhousie University.

