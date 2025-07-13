# Lexora.ai — System Implementation Flow

## 1. Project Overview
Lexora.ai is an AI-powered job application toolkit that helps users generate an ATS-optimized résumé, tailored cover letter, and cold outreach email from their existing résumé and a job description.

---

## 2. Tech Stack

| Layer          | Technology                      |
|----------------|--------------------------------|
| Frontend       | React (TypeScript)             |
| Backend        | Spring Boot (Java 17+)         |
| AI Integration | Google Gemini 1.5 Pro (Vertex AI API) |
| Parsing        | Apache Tika, PDFBox, Apache POI|
| File Storage   | AWS S3 or Google Cloud Storage |
| Database       | PostgreSQL (optional)          |
| Deployment     | AWS EC2 / GCP / Vercel         |

---

## 3. System Components

### Frontend (React)
- Upload résumé (PDF/DOCX)
- Paste job description
- Show ATS keyword match & preview
- Download generated application bundle

### Backend (Spring Boot)

| Module                   | Responsibility                                  |
|--------------------------|------------------------------------------------|
| ResumeParserService       | Extract plain text from résumé files            |
| JobDescriptionService     | Analyze job post for keywords and tone          |
| GeminiPromptService       | Build prompts and call Google Gemini API        |
| DocumentGeneratorService  | Create DOCX/PDF files with Apache POI and iText |
| StorageService            | Upload and serve documents from cloud storage  |
| BundleService             | Package résumé, cover letter, cold email in ZIP |

### API Endpoints

| Method | Path                   | Description                          |
|--------|------------------------|------------------------------------|
| POST   | /api/upload-resume     | Upload résumé, extract text         |
| POST   | /api/submit-job        | Submit job description              |
| POST   | /api/generate          | Generate AI documents               |
| GET    | /api/download/{id}     | Download ZIP bundle                 |

---

## 4. Google Gemini Integration

- Enable Vertex AI API in GCP Console
- Create service account and generate OAuth 2.0 credentials
- Use REST API calls from Spring Boot to send prompts and receive AI-generated content
- Example prompt for cover letter:
"Given the résumé: {resume_text} and job description: {job_text}, generate a professional cover letter tailored to this job."

yaml
Copy
Edit

---

## 5. Document Generation & Bundling

- Use Apache POI to generate DOCX files
- Optionally convert to PDF using iText
- Bundle all generated documents into a ZIP archive
- Upload ZIP to cloud storage (S3 or GCS)
- Return download URL to frontend

---

## 6. ATS Scoring (Optional)

- Compare résumé keywords with job description
- Provide match percentage and highlight missing skills
- Display results as a heatmap on frontend

---

## 7. Deployment Strategy

| Component     | Service               | Notes                             |
|---------------|-----------------------|----------------------------------|
| Backend       | AWS EC2 / Elastic Beanstalk / GCP | Dockerized Spring Boot app       |
| Frontend      | Vercel / Netlify      | React SPA hosting                 |
| Storage       | AWS S3 / Google Cloud Storage | Document hosting                  |
| Database      | RDS PostgreSQL / Cloud SQL | User data and analytics (optional) |
| CI/CD         | GitHub Actions        | Build, test, and deploy pipelines |
| Monitoring    | Prometheus / Grafana  | App health and metrics            |

---

## 8. Security & Auth

- Sanitize file uploads
- Rate limit API requests
- Optional Google OAuth for user accounts
- Protect API keys with secrets manager

---

## 9. Development Milestones

| Week | Goals                                   |
|-------|----------------------------------------|
| 1     | Scaffold frontend + backend, file upload|
| 2     | Parse résumé and job description        |
| 3     | Integrate Gemini AI prompt generation   |
| 4     | Document generation and bundling        |
| 5     | ATS scoring and UI polish                |
| 6     | Deployment and monitoring                |

---

## 10. User Flow Diagram

```plaintext
[User React App]
    |
    ▼
[Upload résumé & job description]
    |
    ▼
[Spring Boot Backend]
    |
    ├─> ResumeParserService
    ├─> JobDescriptionService
    ├─> GeminiPromptService (calls Gemini API)
    ├─> DocumentGeneratorService
    ├─> BundleService (creates ZIP)
    └─> StorageService (uploads ZIP, returns URL)
    |
    ▼
[React Frontend]
    |
    ▼
[User downloads application bundle]
