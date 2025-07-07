# Lucres Job Board Integration & Public Jobs API – Master Technical Specification

**Version:** 1.2 - **Release Date:** July 5, 2025 - **Audience**: Lucres Engineering (Backend, Frontend)

---

## Table of Contents

1. [Introduction & Business Context](#1-introduction--business-context)
2. [Integration Requirements](#2-integration-requirements)
   1. [Functional Requirements](#21-functional-requirements)
   2. [Non-Functional Requirements](#22-non-functional-requirements)
   3. [User Stories & Use Cases](#23-user-stories--use-cases)
3. [System Overview & Architecture](#3-system-overview--architecture)
4. [Embedding Widget](#4-embedding-widget)
   1. [Widget Architecture & Lifecycle](#41-widget-architecture--lifecycle)
   2. [Configuration & Initialization Options](#42-configuration--initialization-options)
   3. [Embedding Snippets](#43-embedding-snippets)
   4. [Customization & Theming](#44-customization--theming)
   5. [Events, Metrics & Error Handling](#45-events--metrics--error-handling)
5. [Public Jobs REST API](#5-public-jobs-rest-api)
   1. [Versioning & Base URL](#51-versioning--base-url)
   2. [Authentication & Security](#52-authentication--security)
   3. [Rate Limiting & Throttling](#53-rate-limiting--throttling)
   4. [Endpoints Specification](#54-endpoints-specification)
      - [List Company Jobs](#541-list-company-jobs)
      - [Get Single Job Detail](#542-get-single-job-detail)
   5. [Request & Response Schema](#55-request--response-schema)
   6. [Error Codes & Pagination](#56-error-codes--pagination)
6. [Client Integration Examples](#6-client-integration-examples)
   1. [Vanilla JavaScript Widget](#61-vanilla-javascript-widget)
   2. [TypeScript API Consumption](#62-typescript-api-consumption)
   3. [React Component Example](#63-react-component-example)
   4. [Vue & Angular Component Sketches](#64-vue--angular-component-sketches)
7. [Plugin & SDK Considerations](#7-plugin--sdk-considerations)
8. [Monitoring, Analytics & SLA](#8-monitoring-analytics--sla)
9. [Accessibility & Performance](#9-accessibility--performance)
10. [Compliance & Security](#10-compliance--security)
11. [Testing & QA](#11-testing--qa)
12. [Roadmap & Future Enhancements](#12-roadmap--future-enhancements)
13. [Support & Contact](#13-support--contact)

---

## 1. Introduction & Business Context

Lucres seeks to enable partner companies to **seamlessly showcase** their live job postings on external career sites, driving candidate engagement while maintaining Lucres as the single source of truth for applications. Mirroring AshbyHQ’s integration, this feature supports both a ready-made JavaScript widget and a fully customizable REST API. By embedding Lucres-powered job boards, companies accelerate time-to-market for careers pages, reduce engineering overhead, and leverage Lucres’s recruiting infrastructure.

@Divesh and @SB - This doc is just a reference for the user-flow and tech requirements. The JSON examples are purely illustrative, so feel free to adapt them to your actual backend and database setup.

**Goals:**

- **Brand Consistency:** Display Lucres branding subtly to build trust.
- **Developer Experience:** Minimize integration complexity; support multiple frameworks.
- **Scalability:** Handle high-traffic requests without degradation.
- **Security & Compliance:** Public data only; robust rate limiting and HTTPS.

## 2. Integration Requirements

### 2.1 Functional Requirements

1. **Embed Widget**: Provide a JS snippet that renders job listings in a container element.
2. **Public API**: Expose endpoints to list jobs and retrieve job detail in JSON.
3. **Redirection**: All **Apply** actions redirect to Lucres job pages.
4. **Real-Time Updates**: New, edited, or removed jobs reflect immediately.
5. **Framework Support**: Offer examples for Vanilla JS, React, Vue, Angular.

### 2.2 Non-Functional Requirements

- **Performance**: Widget script <50 KB gzipped; API p95 <200 ms.
- **Availability**: 99.9% uptime, with health-check endpoints.
- **Rate Limiting**: 100 req/IP/hr, customizable per partner.
- **Logging & Metrics**: Instrument events for load times, errors, impressions.
- **Accessibility**: WCAG 2.1 AA compliance for the widget UI.

### 2.3 User Stories & Use Cases

| ID   | Role         | Goal                                                              | Notes                                           |
| ---- | ------------ | ----------------------------------------------------------------- | ----------------------------------------------- |
| US01 | Marketing    | Embed job board on new careers microsite within 1 day             | Uses basic embed snippet                        |
| US02 | Frontend Dev | Consume Lucres API to render a custom job listing page in React   | Requires list & detail endpoints                |
| US03 | Recruiter    | Update job on Lucres and see change immediately on corporate site | Real-time sync via widget/API                   |
| US04 | Settings     | Turn on/off public job board exposure without code changes        | Controlled via **Company Settings** toggle      |
| US05 | Partner      | Integrate via WordPress / other CMSs plugin, no coding            | Plugin fetches API, renders shortcodes / blocks |

## 3. User Flow – Candidate Experience

To ensure a smooth and intuitive experience, the candidate journey is broken down into discrete steps, mapping their actions to system behavior and widget events:

| Step                              | Candidate Action                           | System Behavior                                                      | Widget Event                                               |
| --------------------------------- | ------------------------------------------ | -------------------------------------------------------------------- | ---------------------------------------------------------- |
| **1. Landing on Careers Page**    | Navigates to `https://company.com/careers` | Widget script loads; container element is identified                 | `onInit` fires (show loading indicator)                    |
| **2. Loading Job Listings**       | —                                          | Sends `GET /v1/public/jobs/{companySlug}`; displays skeleton loader  | `onLoaded` fires with job array                            |
| **3. Browsing Jobs**              | Scrolls through job cards or list items    | Renders title, location, type, snippet; supports keyboard/ARIA       | —                                                          |
| **4. Viewing Job Details**        | Clicks a job card or “View Details”        | Expands inline or opens modal with full HTML description             | `onJobClick` fires (payload: `{ jobSlug, applyUrl, ... }`) |
| **5. Applying to a Job**          | Clicks **Apply on Lucres**                 | Records interaction; redirects to Lucres `applyUrl` in new tab       | —                                                          |
| **6. Post-Application Feedback**  | Completes application on Lucres site       | Displays confirmation; (future) webhook/callback notification        | —                                                          |
| **7. Continuous Synchronization** | Revisits careers page                      | Always fetches latest job listings; no stale cache unless configured | —                                                          |

---

## 4. System Overview & Architecture. System Overview & Architecture

Two integration paths share the same backend services (Node.js + Express + MongoDB/Mongoose):

1. **Embedded Widget**: CDN‑hosted JS (`jobs-embed-v1.js`) + minimal CSS → Browser fetch via `/v1/public/jobs/{slug}` → Client rendering.
2. **Public API**: Direct JSON endpoints consumed by server or client code.

All traffic flows through Lucres API gateways, enforcing HTTPS, rate limiting, and CORS. Metrics feed into Grafana/Kibana dashboards.

---

## 4. Embedding Widget

### 4.1 Widget Architecture & Lifecycle

1. **Load**: `<script>` tag injects `LucresWidget` into page.
2. **Bootstrap**: Reads `<div id="lucres_jobs_widget" data-slug>`, merges global `window.LucresWidget` config.
3. **Fetch**: Calls `/v1/public/jobs/{slug}` with query params.
4. **Render**: Builds accessible DOM (ARIA roles, keyboard).
5. **Events**: Emits init, loaded, click, error events.
6. **Tear-down**: Handles SPA route changes, can be reinitialized.

### 4.2 Configuration & Initialization Options

| Config Method       | Parameters                                 | Override Priority         |
| ------------------- | ------------------------------------------ | ------------------------- |
| **Data Attributes** | `data-slug`, `data-max-jobs`, `data-theme` | Base values               |
| **Global Object**   | `window.LucresWidget = { ... }`            | Overrides data attributes |

Supported params:

- `slug` (string, required)
- `maxJobs` (int, default=20)
- `theme` ("light"|"dark"|"auto")
- `language` (ISO 639-1, default="en")
- `customStylesheet` (URL, optional)
- `showCompanyLogo` (bool, default=true)
- `callbacks` (onInit, onLoaded, onJobClick, onError)

### 4.3 Embedding Snippets

**Basic Embed**

```html
<div id="lucres_jobs_widget" data-slug="AcmeCorp"></div>
<script src="https://cdn.lucres.com/widgets/jobs-embed-v1.js" async defer></script>
```

**Advanced Embed with Callbacks**

```html
<script>
  window.LucresWidget = {
    slug: 'AcmeCorp',
    maxJobs: 10,
    theme: 'dark',
    callbacks: {
      onInit: () => console.log('Widget init'),
      onLoaded: jobs => console.log('Loaded', jobs.length),
      onJobClick: job => window.location = job.applyUrl,
      onError: err => console.error(err)
    }
  };
</script>
<script src="https://cdn.lucres.com/widgets/jobs-embed-v1.js"></script>
```

### 4.4 Customization & Theming

- **CSS Variables** for colors, fonts, spacing.
- **Light/Dark Theme** auto-detect via `prefers-color-scheme`.
- **Logo Injection** via `data-logo-url`.
- **Font Overrides** via `--lucres-widget-font-family`.

### 4.5 Events, Metrics & Error Handling

| Event        | Payload                 | Usage                            |
| ------------ | ----------------------- | -------------------------------- |
| `onInit`     | `{ slug, settings }`    | Track widget instantiation       |
| `onLoaded`   | `{ jobs: Job[] }`       | Count listings, analytics        |
| `onJobClick` | `{ jobSlug, applyUrl }` | Custom redirection or logging    |
| `onError`    | `{ code, message }`     | Display fallback, Sentry capture |

**Error Handling:**

- Retries: 2 attempts with backoff (500ms → 1s).
- Fallback UI: "Unable to load jobs" message and support link.

---

## 5. Public Jobs REST API

### 5.1 Versioning & Base URL

- **Base URL:** `https://api.lucres.com/v1`
- **Resource Path:** `/public/jobs/...`

### 5.2 Authentication & Security

- **Public Access:** No auth needed for reading jobs.
- **HTTPS Only:** All requests must use TLS.
- **Reserved Header:** `X-Lucres-ApiKey` for future private use.

### 5.3 Rate Limiting & Throttling

- Default: 100 requests/IP/hour.
- Custom limits per partner available upon request.
- Response headers: `X-RateLimit-*`, `Retry-After`.

### 5.4 Endpoints Specification

#### 5.4.1 List Company Jobs

```
GET /v1/public/jobs/{companySlug}
```

- **Query Params:**
  - `page` (number, default=1): Page number
  - `perPage` (number, default=20, max=100): Number of items per page
  - `sortBy` (string): Field to sort by (e.g., `publishedAt`, `title`, `location`)
  - `order` (string): Sort order (`asc` or `desc`)
  - `jobType` (string[]): Filter by job type (e.g., `["Full time", "Part time"]`)
  - `location` (string): Filter by location (supports partial matching)
  - `industry` (string[]): Filter by industry (e.g., `["Aviation and Aerospace", "IT"]`)
  - `location` (string): Filter by location (partial match, e.g., "Bengaluru")
  - `q` (string): Full-text search across title and description
  - `postedAfter` (ISO 8601 date): Filter jobs posted after this date
  - `fields` (string[]): Comma-separated list of fields to include in response

- **200 OK:** 
  ```json
  {
    "apiVersion": "1",
    "meta": {
      "page": 1,
      "perPage": 20,
      "total": 45,
      "totalPages": 3
    },
    "jobs": [/* Job[] */]
  }
  ```
- **400 Bad Request:** Invalid query parameters
- **404 Not Found:** Company not found or job board disabled
- **429 Too Many Requests:** Rate limit exceeded

#### 5.4.2 Get Single Job Detail

```
GET /v1/public/jobs/{companySlug}/{jobSlug}
```

- **Query Params:**
  - `fields` (string[]): Comma-separated list of fields to include in response
  - `include` (string[]): Related resources to include (e.g., `["hiringTeam", "department"]`)

- **200 OK:** 
  ```json
  {
    "apiVersion": "1",
    "job": {
      "id": "job_abc123",
      "title": "Senior Software Engineer",
      "jobSlug": "senior-software-engineer-xyz",
      "applyUrl": "https://lucres.com/apply/job_abc123",
      "descriptionHtml": "<p>Job description here...</p>",
      "descriptionPlain": "Job description here...",
      "company": {
        "name": "Axiscades Engineering Technology Ltd",
        "logoUrl": "https://example.com/axiscades-logo.png"
      },
      "location": "Bengaluru, India",
      "jobType": "Full time",
      "salary": "₹ 3lakh - 15lakh Yearly",
      "experience": "3 Years",
      "industry": "Aviation and Aerospace",
      "openings": 10,
      "degree": "Bachelors",
      "skills": [
        "stress analysis",
        "finite element analysis",
        "structural engineering"
      ],
      "jobBrief": "Join the dynamic team at Axiscades Engineering Technology Ltd as a Stress Engineer, where innovation meets engineering excellence in the Aviation and Aerospace sector...",
      "description": "Conduct detailed stress analysis on aircraft structures and components to ensure safety and performance standards.....",
      "publishedAt": "2025-07-01T00:00:00Z"
    }
  }
  ```
- **404 Not Found:** Job or company not found
- **410 Gone:** Job has been filled or removed

### 5.5 Request & Response Schema - @DL Please update this acc to backend structure and current Lucres API.

```json
{
  "apiVersion": "1",
  "meta": { /* pagination info */ },
  "jobs": [ /* Job[] */ ]
}
```

**Job Object:**

  - `title` (string): Job title (e.g., "Stress Engineer")
  - `company` (object): Company information
    - `name` (string): Company name (e.g., "Axiscades Engineering Technology Ltd")
    - `logoUrl` (string, optional): URL to company logo
  - `location` (string): Job location (e.g., "Bengaluru, India")
  - `jobType` (string): Employment type (e.g., "Full time")
  - `salary` (string): Salary range (e.g., "₹ 3lakh - 15lakh Yearly")
  - `experience` (string): Required experience (e.g., "3 Years")
  - `industry` (string): Industry sector (e.g., "Aviation and Aerospace")
  - `openings` (number): Number of open positions (e.g., 10)
  - `degree` (string): Required education level (e.g., "Bachelors")
  - `skills` (string[]): Array of required skills (e.g., ["stress analysis", "finite element analysis"])
  - `description` (string): Full job description
  - `jobBrief` (string): Short job summary
  - `publishedAt` (ISO 8601 date): When the job was published
  - `jobSlug` (string): URL-friendly job identifier
  - `applyUrl` (string): Direct application URL

### 5.6 Error Codes & Pagination

- **400 BadRequest** – invalid params
- **404 NotFound** – slug missing/disabled
- **429 RateLimitExceeded** – throttle
- **500 InternalError** – server fault

**Pagination:** via `meta` object and `Link` header.

---

## 6. Client Integration Examples

### 6.1 Vanilla JavaScript Widget

```html
<div id="lucres_jobs_widget" data-slug="AcmeCorp" data-max-jobs="5"></div>
<script src="https://cdn.lucres.com/widgets/jobs-embed-v1.js" async></script>
```

### 6.2 TypeScript API Consumption

```ts
interface Job {
  title: string;
  jobSlug: string;
  applyUrl: string;
  // ...other fields
}

async function fetchJobs(slug: string, page = 1): Promise<Job[]> {
  const res = await fetch(
    `https://api.lucres.com/v1/public/jobs/${slug}?page=${page}`
  );
  if (!res.ok) throw new Error(`Error ${res.status}`);
  const data: { jobs: Job[] } = await res.json();
  return data.jobs;
}

(async () => {
  try {
    const jobs = await fetchJobs('AcmeCorp');
    console.log('Jobs:', jobs);
  } catch (err) {
    console.error('Fetch error', err);
  }
})();
```

### 6.3 React Component Example

```tsx
import React, { useEffect, useState } from 'react';

export function LucresJobList({ slug }: { slug: string }) {
  const [jobs, setJobs] = useState<Job[]>([]);
  useEffect(() => {
    fetch(
      `https://api.lucres.com/v1/public/jobs/${slug}`
    )
      .then(res => res.json())
      .then(data => setJobs(data.jobs))
      .catch(console.error);
  }, [slug]);
  return (
    <ul>
      {jobs.map(job => (
        <li key={job.jobSlug}>
          <a href={job.applyUrl} target="_blank">
            {job.title}
          </a>
        </li>
      ))}
    </ul>
  );
}
```

### 6.4 Vue & Angular Component Sketches

- **Vue:** `vue-lucres-jobs` component accepting `slug` prop, emits `jobclick`.
- **Angular:** `LucresJobsComponent` with `@Input() slug`, `@Output() jobClick` EventEmitter.

---

## 7. Plugin & SDK Considerations

**Packaged Integrations** to simplify adoption:

- **WordPress Plugin:**
  - Settings UI with preview
  - Gutenberg block + shortcode
  - Caching layer for performance
  - GDPR compliance tools

- **React/Vue/Angular SDKs:**
  - TypeScript types
  - Customizable components
  - State management integration
  - Error boundaries and loading states
  - Test utilities

- **Vanilla JS SDK:**
  - Lightweight (<20KB gzipped)
  - Promise-based API
  - Automatic retries
  - Browser and Node.js support

- **Web Components:**
  - Framework-agnostic
  - Shadow DOM isolation
  - Custom elements for job listings, search, filters

## 8. Rate Limiting & Quotas

### 8.1 Default Limits
- Public API: 100 requests/IP/hour
- Authenticated API: 1,000 requests/client/hour

### 8.2 Headers
- `X-RateLimit-Limit`: Request limit per hour
- `X-RateLimit-Remaining`: Remaining requests
- `X-RateLimit-Reset`: UTC timestamp when limit resets
- `Retry-After`: Seconds to wait after rate limit exceeded

---

## 8. Monitoring, Analytics & SLA

- **Health Checks:** `/healthz` endpoint returns 200.
- **Metrics:** Expose Prometheus metrics: request rates, latencies, error counts.
- **SLA:** 99.9% uptime, <1% errors p95.

---

## 9. Accessibility & Performance

- **WCAG 2.1 AA** compliance for color contrast, keyboard nav.
- **Lazy Load** cards beyond viewport.
- **Script Size** ≤50 KB gzipped, bundle minimal polyfills.

---

## 10. Compliance & Security

- **HTTPS Only**;
- **CORS**: Allow all origins for public endpoints.
- **Content Security Policy**: Restrict script sources.

---

## 11. Testing & QA

- **Unit Tests:** Endpoint mocks, schema validation (Jest).
- **Integration:** Postman/Newman collections.
- **E2E:** Cypress for widget flows.
- **Performance:** Lighthouse CI, API benchmark tests.

---

## 12. Roadmap & Future Enhancements

- **Webhooks** for job create/update.
- **Private CRUD API** for job management.
- **Search & Facets** in public API.
- **Localization** support in widget.
- **ATS Connectors** (Greenhouse, Lever).
