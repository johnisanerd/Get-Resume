---
name: apify-job-scraper
description: >
  This skill should be used when the user asks to "scrape jobs", "set up Apify",
  "integrate job platforms", "build the job scraping feature", "create scraping runs",
  "handle Apify webhooks", "implement job deduplication", "export jobs to Excel",
  or mentions job scraping, Apify, LinkedIn scraper, Indeed scraper, StepStone,
  scraped jobs, scraping run, webhook ingestion, job data, job listing,
  job search automation, or any job collection feature.
  Triggers on: scraping, Apify, jobs, LinkedIn, Indeed, StepStone, job listing,
  scraping run, webhook, job data, deduplication, job export, Excel export.
---

# Apify Job Scraper

Automated job collection from German job platforms (LinkedIn, Indeed, StepStone) via Apify, with webhook-based ingestion, deduplication, and Excel export.

## Architecture

- **Apify** (cloud service) runs pre-built actors for each job platform
- **Django DRF** webhook endpoint receives scraping results
- **Celery** tasks process and deduplicate incoming job data
- **PostgreSQL** stores structured job listings
- **Vue.js** frontend displays results in filterable table

## Supported Platforms

| Platform | Apify Actor | Coverage |
|----------|------------|---------|
| LinkedIn Jobs | `johnvc/linkedin-jobs-api` | Global + DE |
| Indeed | `johnvc/indeed-jobs-api` | indeed.de |
| StepStone | Custom actor or `misceres/stepstone-scraper` | stepstone.de |

Actor input and the fields to map into `ScrapedJob`:
- LinkedIn: one `keyword` per run, plus `location`, `country: "DE"` and `maxJobs`. Rows carry `jobId`, `title`, `companyName`, `companyUrl`, `location`, `description`, `employmentType` and `jobUrl`.
- Indeed: `mode: "search"`, `keywords` (list), `location`, `country: "DE"`, `locationRadiusMiles` and `maxResultsPerInput`. Rows carry `jobId`, `jobTitle`, `companyName`, `companyUrl`, `location`, `descriptionMarkdown`, `salaryMin`, `salaryMax`, `isRemote` and `jobUrl`.

## Scraping Flow

```
1. User configures search in Vue.js (ScrapingConfig component):
   - Platform(s): LinkedIn, Indeed, StepStone
   - Keywords: "Python Entwickler", "Data Engineer"
   - Location: "Berlin" or PLZ "10115"
   - Radius: 25km / 50km / 100km
   - Job type: Vollzeit / Teilzeit / Mini-Job / Praktikum
   - Schedule: On-demand / Daily / Weekly

2. Vue.js → POST /api/v1/jobs/scraping-runs/
   → Django creates ScrapingRun record
   → Calls Apify API to start actor run
   → Returns run ID to frontend

3. Apify processes scraping (async, minutes)
   → On completion: POST /api/v1/webhooks/apify/ (webhook)

4. Django webhook handler:
   → Validates HMAC signature
   → Triggers Celery task: process_scraping_results

5. Celery task:
   → Fetches results from Apify dataset API
   → Maps to ScrapedJob model fields
   → Deduplicates by hash(company + title + location)
   → Saves to PostgreSQL
   → Updates ScrapingRun status + job count

6. Vue.js polls GET /api/v1/jobs/scraping-runs/{id}/
   → Shows progress → loads results when complete
```

## Django Models

### ScrapingRun
```python
class ScrapingRun(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid4)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    platform = models.CharField(choices=PLATFORM_CHOICES)  # linkedin/indeed/stepstone
    keywords = models.CharField(max_length=500)
    location = models.CharField(max_length=200)
    radius_km = models.IntegerField(default=50)
    job_type = models.CharField(choices=JOB_TYPE_CHOICES, blank=True)
    status = models.CharField(choices=STATUS_CHOICES)  # pending/running/completed/failed
    jobs_found = models.IntegerField(default=0)
    apify_run_id = models.CharField(max_length=100, blank=True)
    started_at = models.DateTimeField(null=True)
    completed_at = models.DateTimeField(null=True)
    schedule = models.CharField(max_length=50, blank=True)  # cron expression
    created_at = models.DateTimeField(auto_now_add=True)
```

### ScrapedJob
```python
class ScrapedJob(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid4)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    scraping_run = models.ForeignKey(ScrapingRun, on_delete=models.CASCADE)
    platform = models.CharField(choices=PLATFORM_CHOICES)
    external_id = models.CharField(max_length=200, blank=True)
    title = models.CharField(max_length=500)
    company_name = models.CharField(max_length=300)
    company_url = models.URLField(blank=True)
    location = models.CharField(max_length=300)
    description = models.TextField()
    requirements = models.TextField(blank=True)
    salary_range = models.JSONField(default=dict, blank=True)
    job_type = models.CharField(max_length=50, blank=True)
    contact_person = models.CharField(max_length=200, blank=True)
    contact_email = models.EmailField(blank=True)
    posting_url = models.URLField()
    application_deadline = models.DateField(null=True, blank=True)
    is_remote = models.BooleanField(default=False)
    scraped_at = models.DateTimeField(auto_now_add=True)
    dedup_hash = models.CharField(max_length=64, unique=True)  # SHA256
    raw_data = models.JSONField(default=dict)
```

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/jobs/` | GET | List scraped jobs (paginated, filterable) |
| `/api/v1/jobs/{id}/` | GET | Job detail with full description |
| `/api/v1/jobs/{id}/` | DELETE | Remove job from list |
| `/api/v1/jobs/scraping-runs/` | GET | List scraping runs |
| `/api/v1/jobs/scraping-runs/` | POST | Start new scraping run |
| `/api/v1/jobs/scraping-runs/{id}/` | GET | Run status + progress |
| `/api/v1/jobs/export/` | GET | Export jobs to XLSX |
| `/api/v1/webhooks/apify/` | POST | Apify webhook callback |

## Filters (Vue.js JobTable)

- Platform: LinkedIn / Indeed / StepStone / All
- Job type: Vollzeit / Teilzeit / Mini-Job / Praktikum
- Location: text search
- Remote: Yes / No / All
- Date range: scraped within last N days
- Search: full-text across title + company + description
- Sort: Newest / Company A-Z / Title A-Z

## Rate Limits (Subscription Tiers)

| Plan | Jobs/month | Scraping Runs |
|------|-----------|---------------|
| Kostenlos | 0 | 0 |
| Premium | 50 | 5 runs/month |
| Business | 500 | 50 runs/month |

## Deduplication

Hash = SHA256(lowercase(company_name) + "|" + lowercase(title) + "|" + lowercase(location))
- If hash exists for user → skip (don't insert duplicate)
- Dedup is per-user (same job can exist for different users)

## Security

- Apify webhook verified via HMAC signature header
- Apify API key stored in Django settings (never exposed to frontend)
- User can only see their own scraped jobs (queryset filtering)
