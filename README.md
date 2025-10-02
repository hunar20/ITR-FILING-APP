# ITR MVP - AI Prefilled PDF Generator (Option A) - With Postgres & S3

This repository is a scaffold for a minimal MVP that lets clients:
- Fill basic tax details via a web form
- Ask the server to "AI-normalize" (stubbed) the data
- Generate a downloadable PDF preview of the filled ITR (HTML -> PDF)
- Store previous-year returns (Postgres-backed)
- Store files (Form16, PDFs) in AWS S3 with presigned URLs

## What changed
- Backend now uploads user documents and generated PDFs to S3 and stores S3 keys in Postgres.
- Presigned URLs are generated for secure, temporary downloads.

## Setup (quick)
1. Create an S3 bucket (example name: `itr-returns-bucket`) in your preferred region (ap-south-1 recommended for India).
2. Create an IAM user with `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` permissions scoped to the bucket (least privilege).
3. Set environment variables (example `.env`):
   ```
   DATABASE_URL=postgres://postgres:postgres@localhost:5432/itrdb
   JWT_SECRET=your_jwt_secret
   AWS_REGION=ap-south-1
   AWS_BUCKET=itr-returns-bucket
   AWS_ACCESS_KEY_ID=AKIA...
   AWS_SECRET_ACCESS_KEY=...
   SIGNED_URL_EXPIRES=300
   ```
4. Run Postgres (Docker or local). Example Docker:
   ```
   docker run --name itr-postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=itrdb -p 5432:5432 -d postgres:15
   ```
5. Backend:
   ```
   cd backend
   npm install
   node server.js
   ```
6. Frontend:
   ```
   cd frontend
   npm install
   npm run dev
   ```

## Endpoints of interest
- `POST /api/signup` â create user
- `POST /api/login` â login -> returns JWT
- `POST /api/returns` â create draft (auth)
- `POST /api/returns/:id/upload` â upload Form16 (auth) -> stored to S3
- `POST /api/returns/:id/ai-normalize` â run AI mapping (stub)
- `POST /api/returns/:id/generate` â generate PDF -> uploaded to S3
- `GET /api/returns/:id/download` â get presigned URL (auth)
- `GET /api/returns/:id/documents` â list documents with presigned URLs (auth)

## Notes
- This is still a demo scaffold. For production, add:
  - Input validation & sanitization
  - Rate limiting and WAF
  - Use managed secrets for AWS keys (AWS Secrets Manager)
  - Use IAM roles for EC2/ECS rather than static keys where possible
  - Replace Puppeteer local PDF generation with streaming to S3 or use a serverless PDF builder


## AI Auto-fill Setup

To enable AI auto-fill, set the following environment variables in your `.env` or environment:
```
OPENAI_API_KEY=sk-...
AI_MODEL=gpt-4o-mini
```
Then restart the backend. The endpoint `POST /api/returns/:id/auto-fill` will:
- Find uploaded documents for the return, download them from S3, extract text via `pdf-parse`, call OpenAI to create structured JSON, and update the return draft with `data.normalized`.

Note: OpenAI usage may incur costs. Test with a small number of documents first.


## Sample demo dataset

A small sample is included in `sample_demo/`:
- `sample_user.json` â use these credentials to signup (or copy/paste into signup form).
- `sample_form16.txt` â a simple mock Form16 text you can upload as a file in the app (choose the file when uploading).\n
Notes: the mock Form16 is a .txt file to simulate document upload in the demo. For real parsing upload a PDF Form16; the OCR fallback is triggered when pdf-parse returns little text.


## Payments (Razorpay)

To enable payments:
1. Create a Razorpay account and get `KEY_ID` and `KEY_SECRET`.
2. Set in backend env or `.env`:
```
RAZORPAY_KEY_ID=your_key_id
RAZORPAY_KEY_SECRET=your_key_secret
```
3. The frontend uses Razorpay Checkout. For local testing, open the app and click **Pay Now**. The frontend uses a placeholder amount of â¹199; change as needed.

Note: For production, ensure webhooks are configured and you verify payments server-side via Razorpay webhooks as an additional security step.


## Razorpay webhook & Admin

- Set `RAZORPAY_WEBHOOK_SECRET` in backend env and configure the same secret in Razorpay's webhook settings.
- Webhook endpoint: `POST /api/payment/webhook` â it validates signature and marks payments as paid.
- Admin endpoints (use `x-admin-secret` header):
  - `GET /admin/payments` â list recent payments
  - `GET /admin/users` â list users
- Set `ADMIN_SECRET` env var to protect admin endpoints.


## Email Notifications

Configure SMTP in `.env`:
```
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your_email@gmail.com
EMAIL_PASS=your_app_password
```
Emails are sent:
- When payment is confirmed
- When PDF is generated

For Gmail, enable App Passwords and use that as EMAIL_PASS.
