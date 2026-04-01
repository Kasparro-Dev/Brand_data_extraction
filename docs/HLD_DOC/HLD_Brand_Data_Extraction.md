# High-Level Design (HLD) Document

**Project Name:** Kasparro Brand Data Extraction & Attribution Pipeline
**Version:** 1.0
**Author:** Kasparro Team
**Date:** 01/04/2026
**Generated from:** Codebase analysis (code is source of truth)

---

## 1. Overview

The Kasparro Brand Data Extraction & Attribution Pipeline is a per-client data extraction system that pulls product intelligence from Shopify stores, traffic and attribution data from Google Analytics 4 (GA4), and search performance data from Google Search Console (GSC) -- then assembles everything into structured Excel workbooks for brand audit and ongoing attribution tracking.

Each brand onboards by sharing credentials for their own Shopify Custom App (Client ID + Client Secret), GA4 Property ID, and GSC Site URL. Kasparro runs the extraction pipeline on demand to generate brand audit reports and track week-over-week attribution metrics.

- **System Name:** Brand Data Extraction Pipeline (internally `brand-extraction`)
- **Purpose:** Automate the 60-90 minute manual process of pulling product data from Shopify, GA4, and GSC into a structured brand audit workbook -- and provide continuous attribution measurement every week.
- **Target Users:** (1) Kasparro Brand Engineers who run extractions on demand per client. (2) The brand's own team who receives the Excel workbook for validation. (3) Downstream Kasparro services (BIB, Pillar 2 audit) that consume extraction data.
- **Business Goal:** Eliminate manual extraction work at onboarding, establish a frozen baseline per brand, and measure Kasparro's week-over-week impact on organic traffic, search visibility, and revenue attribution.

---

## 2. Requirements

### 2.1 Functional Requirements

- **Per-Client Extraction:** Run the same extraction pipeline for any Shopify store by substituting credentials (Client ID, Client Secret, GA4 Property ID, GSC Site URL). Each brand's data is completely isolated.
- **Shopify Data Extraction:** Extract all product data including names, prices, variants, descriptions, tags, collections, images, metafields, SEO metadata, and product status (active/draft). Two modes:
  - Admin API (GraphQL): Full data including metafields, collections, SEO, images -- requires Client ID + Client Secret OAuth flow
  - Public JSON: Product names, prices, descriptions, variants, tags, images -- no auth, metafields not available
- **Metafield Auto-Discovery:** Query the store's `metafieldDefinitions` schema to discover custom fields, then auto-map them to audit fields using keyword matching. No hardcoded store-specific logic.
- **3-Layer Extraction Per Product:** Layer 1 (guaranteed Shopify fields) -> Layer 2 (auto-mapped metafields) -> Layer 3 (description/tag fallback parsing). Consistent data quality across any Shopify store.
- **Review App Detection:** Detect which review app is installed (Judge.me, Yotpo, Loox, Stamped, Shopify Reviews, Amazon via Reputon) and extract rating + review count.
- **GA4 Data Extraction:** Pull 8 report types via GA4 Data API (Summary, Channels, Top Pages, Age Groups, Gender, Cities, Countries, Devices, Weekly Trend) for the last 30 days.
- **GSC Data Extraction:** Pull 5 report types via GSC Search Console API (Summary, Top Queries, Top Pages, Countries, Devices, Zero-Click Queries) for the last 90 days.
- **Excel Workbook Generation:** Compile all data into a multi-sheet Excel workbook with: 1 summary sheet, N product sheets (one per product), 6 GA4 sheets, 4 GSC sheets. Color-coded: white = auto-filled, yellow/orange = manual fill required.
- **TOP 4 Products Only:** Always extract and report on the top 4 products by some ordering signal (not all products). Maximum detail per product.
- **Metaobject Handling:** When a metafield contains a metaobject GID reference (e.g., `gid://shopify/Metaobject/...`), display a placeholder `[metaobject references]` and note that `read_metaobjects` scope is needed to resolve them.
- **GA4-Only Mode:** A `--ga4-only` flag skips Shopify extraction entirely. Useful for weekly light extraction when Shopify product data has not changed.
- **Command-Line Interface:** Single Python script executed from the command line. No web server, no API, no database. Input via CLI flags, output to filesystem.
- **Multi-Client Output:** Output files saved to the `./sheets/` directory with naming pattern: `{store}_{audit}_{timestamp}.xlsx`.

### 2.2 Non-Functional Requirements

| Attribute | Requirement |
| :--- | :--- |
| Availability | 99.9% -- manual execution; failures surface as CLI errors |
| Latency | Shopify GraphQL: <15s timeout per page. GA4: <30s. GSC: <60s. |
| Scalability | Single-threaded CLI. Handles 1 client at a time. |
| Consistency | Synchronous execution -- no partial writes. File saved only after all sources complete. |
| Durability | Excel files are the output artifact. Stored locally in `./sheets/`. |
| Security | Shopify Client ID + Secret never stored in code. GA4 credentials file (`ga4_credentials.json`) excluded from git. |
| Brand Isolation | Each brand's credentials are independent. No cross-brand data access. |
| Token Expiry | Shopify tokens expire every 24 hours. The script requests a fresh token on every run via OAuth client credentials grant. |
| Rate Limits | Shopify: 40 req/min per store (per app). GA4: 10,000 req/day per property. GSC: no explicit limit, row limit 25,000/request. |

### 2.3 Capacity Estimation

| Metric | Estimate |
| :--- | :--- |
| Brands Supported | < 50 concurrent clients |
| Products Per Brand | 4 (top 4) per extraction |
| Fields Per Product | ~50 extracted fields (40 audit fields + 10 internal fields) |
| GA4 Reports | 8 per extraction |
| GSC Reports | 5 per extraction |
| Output File Size | ~100-500 KB per Excel workbook |
| Storage Per Brand | ~10-50 MB per year (52 weekly extractions) |
| API Calls Per Run | ~10-20 Shopify (paginated GraphQL), 8 GA4, 5 GSC = ~25-35 total |

---

## 3. Architecture Overview

### 3.1 Architecture Style

**Standalone CLI Script with 3-Layer Extraction Pipeline.** The system is a single Python file (`brand_audit_extractor.py`) executed from the command line. There is no web server, no database, no container, and no cloud infrastructure in the current implementation. The script orchestrates three independent external API integrations (Shopify, GA4, GSC) in sequence, then assembles the output into an Excel workbook on the local filesystem.

**Justification from code:** `brand_audit_extractor.py` is a 1,728-line standalone script with no FastAPI, no `uvicorn`, no Docker, and no CI/CD. The `main()` function parses CLI arguments, calls `get_shopify_token()`, `fetch_products_graphql()`, `fetch_ga4()`, `fetch_gsc()` sequentially, then `build_excel()`. There are no background tasks, no message queues, and no persistent state beyond the output Excel file.

### 3.2 High-Level Architecture Diagram

```
+---------------------------------------------------------------------------------+
|           Kasparro Brand Data Extraction Pipeline (Local CLI)                    |
|                                                                                 |
|  +---------------------------------------------------------------------------+   |
|  |  Python Runtime (brand_audit_extractor.py)                                  |   |
|  |                                                                           |   |
|  |  +-----------------------------+                                           |   |
|  |  |  CLI Argument Parser        |                                           |   |
|  |  |  (sys.argv)                 |                                           |   |
|  |  +-----------------------------+                                           |   |
|  |                                |                                              |   |
|  |  +-----------------------------+-----------------------+                    |   |
|  |  |         Extraction Orchestrator (main())            |                    |   |
|  |  |                                                       |                    |   |
|  |  |  Phase 1: Shopify (sequential)                       |                    |   |
|  |  |    +---------------------+  +-----------------------+  |                    |   |
|  |  |    | OAuth Token Request |  | GraphQL Product      |  |                    |   |
|  |  |    | client_credentials  |  | Fetch (paginated)    |  |                    |   |
|  |  |    +---------------------+  +-----------------------+  |                    |   |
|  |  |    | Metafield Schema    |  +-----------------------+  |                    |   |
|  |  |    | Discovery           |  | extract_product_      |  |                    |   |
|  |  |    | (metafieldDefs)     |  | universal()           |  |                    |   |
|  |  |    +---------------------+  +-----------------------+  |                    |   |
|  |  |                                                       |                    |   |
|  |  |  Phase 2: GA4 (if credentials provided)               |                    |   |
|  |  |    +----------------------------------------+        |                    |   |
|  |  |    | fetch_ga4()                             |        |                    |   |
|  |  |    | BetaAnalyticsDataClient (8 reports)     |        |                    |   |
|  |  |    +----------------------------------------+        |                    |   |
|  |  |                                                       |                    |   |
|  |  |  Phase 3: GSC (if credentials provided)               |                    |   |
|  |  |    +----------------------------------------+        |                    |   |
|  |  |    | fetch_gsc()                             |        |                    |   |
|  |  |    | googleapiclient (SearchConsole API)     |        |                    |   |
|  |  |    +----------------------------------------+        |                    |   |
|  |  |                                                       |                    |   |
|  |  |  Phase 4: Excel Assembly                            |                    |   |
|  |  |    +----------------------------------------+        |                    |   |
|  |  |    | build_excel()                            |        |                    |   |
|  |  |    | openpyxl (multi-sheet workbook)         |        |                    |   |
|  |  |    +----------------------------------------+        |                    |   |
|  |  +---------------------------------------------------------------------+-----+   |
|  +-------------------------------------------------------------------------|-------+
|                                                                                 |
+--------------------------------------------|--------------------------------------+
                                             |
                    +------------------------+------------------------+
                    |                        |                        |
                    v                        v                        v
         +------------------+     +--------------------+    +---------------------+
         | Shopify Admin API |     | GA4 Data API       |    | GSC Search Console  |
         | (OAuth + GraphQL) |     | (Service Account)  |    | API (Service Account)|
         |                  |     |                    |    |                     |
         | OAuth endpoint:   |     | Credentials:       |    | Same credentials:    |
         | {store}/admin/   |     | ga4_credentials.json|    | ga4_credentials.json|
         | oauth/           |     |                    |    |                     |
         | access_token     |     | Property ID:        |    | Site URL format:     |
         |                  |     | 526018830           |    | sc-domain:domain.com|
         | Client ID/Secret|     |                    |    |                     |
         | from brand       |     | 8 report types:     |    | 5 report types:     |
         |                  |     | summary, channels,  |    | summary, queries,   |
         | 250+ fields per |     | pages, age, gender, |    | pages, countries,   |
         | product fetched  |     | cities, countries,  |    | devices, zero-click |
         |                  |     | devices, weekly     |    |                     |
         +------------------+     +--------------------+    +---------------------+

+--------------------------------------------|--------------------------------------+
                                             |                                              |
                                             v                                              v
                              +-----------------------------+    +--------------------------------+
                              |  ./sheets/                  |    |  ga4_credentials.json          |
                              |  {brand}_audit_            |    |  (GCP service account JSON)    |
                              |  {timestamp}.xlsx          |    |  Excluded from git             |
                              |                             |    |  Same file for GA4 and GSC    |
                              +-----------------------------+    +--------------------------------+

Per-Brand Credentials (passed via CLI flags per run):
  Brand A: --client-id X --client-secret Y --ga4-property 111 --gsc-site sc-domain:a.com
  Brand B: --client-id P --client-secret Q --ga4-property 222 --gsc-site sc-domain:b.com
```

### 3.3 Data Flow Diagram

```
CLI: python brand_audit_extractor.py https://skinq.myshopify.com/admin \
    --client-id CLIENT_ID \
    --client-secret CLIENT_SECRET \
    --ga4-property 526018830 \
    --ga4-credentials ./ga4_credentials.json \
    --gsc-site sc-domain:skinq.com \
    --output ./sheets/

                                    ┌─────────────────────────────────────┐
                                    │         STEP 1: AUTHENTICATE        │
                                    │  POST /admin/oauth/access_token     │
                                    │  Content-Type: application/        │
                                    │  x-www-form-urlencoded              │
                                    │  grant_type=client_credentials      │
                                    └──────────────────┬──────────────────┘
                                                       │ access_token
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   STEP 2: DISCOVER METAFIELD SCHEMA  │
                                    │  POST /admin/api/2026-01/graphql.json│
                                    │  Query: metafieldDefinitions        │
                                    │  Returns: namespaces, keys, types   │
                                    │  Build: field_map (keyword matching) │
                                    └──────────────────┬──────────────────┘
                                                       │ definitions + field_map
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   STEP 3: FETCH ALL PRODUCTS        │
                                    │  POST /admin/api/2026-01/graphql.json│
                                    │  Query: products(first: 50, after:) │
                                    │  Pagination: cursor-based, all pages │
                                    │  Per product: metafields (50 max)   │
                                    └──────────────────┬──────────────────┘
                                                       │ raw_products[]
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   STEP 4: EXTRACT PER PRODUCT        │
                                    │  extract_product_universal()         │
                                    │                                       │
                                    │  Layer 1: Guaranteed fields           │
                                    │    (title, price, type, status, etc.)│
                                    │                                       │
                                    │  Layer 2: Metafield mapping           │
                                    │    (find_metafield(field_map))       │
                                    │    -> hero molecules, certs, claims   │
                                    │                                       │
                                    │  Layer 3: Description/tag fallback   │
                                    │    parse_description_sections()       │
                                    │    extract_molecules_from_text()      │
                                    │    classify_tags()                     │
                                    │                                       │
                                    │  Output: product_dict (~50 fields)   │
                                    └──────────────────┬──────────────────┘
                                                       │ products[4]
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   STEP 5: FETCH GA4 DATA            │
                                    │  BetaAnalyticsDataClient()          │
                                    │  8 x run_report() calls              │
                                    │  Date range: 30 days ago -> today   │
                                    └──────────────────┬──────────────────┘
                                                       │ ga4_data{}
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   STEP 6: FETCH GSC DATA            │
                                    │  googleapiclient.build('webmasters') │
                                    │  5 x searchAnalytics().query()      │
                                    │  Date range: 90 days ago -> today   │
                                    └──────────────────┬──────────────────┘
                                                       │ gsc_data{}
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   STEP 7: BUILD EXCEL WORKBOOK       │
                                    │  build_excel(products, ga4, gsc)     │
                                    │                                       │
                                    │  Sheet 1:  Audit Summary             │
                                    │  Sheets 2-5: Product N (per product) │
                                    │  Sheet 6:  GA4 Summary               │
                                    │  Sheet 7:  GA4 Channels              │
                                    │  Sheet 8:  GA4 Top Pages             │
                                    │  Sheet 9:  GA4 Demographics          │
                                    │  Sheet 10: GA4 Locations             │
                                    │  Sheet 11: GA4 Devices               │
                                    │  Sheet 12: GA4 Weekly Trend          │
                                    │  Sheet 13: GSC Summary               │
                                    │  Sheet 14: GSC Top Queries           │
                                    │  Sheet 15: GSC Top Pages             │
                                    │  Sheet 16: GSC Zero-Click Queries    │
                                    │                                       │
                                    │  Color coding:                        │
                                    │    White = auto-filled                │
                                    │    Orange (#FFF2CC) = manual fill    │
                                    └──────────────────┬──────────────────┘
                                                       │ .xlsx file
                                                       v
                                    ┌─────────────────────────────────────┐
                                    │   OUTPUT FILE                        │
                                    │  ./sheets/{brand}_audit_{ts}.xlsx   │
                                    └─────────────────────────────────────┘
```

---

## 4. API Design

**Protocol:** CLI execution (not a REST API). The script is invoked from the command line with flags. No HTTP server runs in production.

### 4.1 CLI Command Reference

**Full extraction (Shopify + GA4 + GSC):**

```bash
python brand_audit_extractor.py https://CLIENT_STORE.myshopify.com/admin \
    --client-id CLIENT_CLIENT_ID \
    --client-secret CLIENT_CLIENT_SECRET \
    --ga4-property CLIENT_GA4_PROPERTY_ID \
    --ga4-credentials ./ga4_credentials.json \
    --gsc-site sc-domain:CLIENT_DOMAIN.com \
    --output ./sheets/
```

**Shopify Admin API with access token (pre-obtained):**

```bash
python brand_audit_extractor.py https://CLIENT_STORE.myshopify.com/admin \
    --access-token shpat_xxxxxxxxxxxxxxxxxxxx \
    --output ./sheets/
```

**Public JSON only (no auth, no metafields):**

```bash
python brand_audit_extractor.py https://www.skinq.com/products.json \
    --output ./sheets/
```

**GA4-only mode (skip Shopify, e.g., for weekly light extraction):**

```bash
python brand_audit_extractor.py https://CLIENT_STORE.myshopify.com/admin \
    --client-id CLIENT_CLIENT_ID \
    --client-secret CLIENT_CLIENT_SECRET \
    --ga4-property CLIENT_GA4_PROPERTY_ID \
    --ga4-credentials ./ga4_credentials.json \
    --gsc-site sc-domain:CLIENT_DOMAIN.com \
    --ga4-only \
    --output ./sheets/
```

### 4.2 Shopify Authentication API

**Endpoint:** `POST {store}/admin/oauth/access_token`

```
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
client_id={CLIENT_ID}
client_secret={CLIENT_SECRET}

Response (200):
{
  "access_token": "shpat_xxxx",
  "expires_in": 86400,
  "associated_user_scope": "read_products,read_content"
}

Response (401):
{
  "errors": "API permission request requires you to be authenticated as a Shopify user."
}
```

**IMPORTANT:** The OAuth endpoint must be `{store}/admin/oauth/access_token`. Using `https://shopify.com/authentication/oauth/token` or `https://accounts.shopify.com/oauth/token` will fail.

**IMPORTANT:** `Content-Type` must be `application/x-www-form-urlencoded`, NOT `application/json`.

### 4.3 Shopify GraphQL API

**Endpoint:** `POST {store}/admin/api/2026-01/graphql.json`

**Authentication:** `X-Shopify-Access-Token: {access_token}`

**Metafield Schema Discovery Query:**

```graphql
{
  metafieldDefinitions(ownerType: PRODUCT, first: 250) {
    edges {
      node {
        name
        namespace
        key
        type { name }
        description
      }
    }
  }
  productsCount { count }
}
```

**Product Fetch Query (paginated):**

```graphql
query Products($cursor: String) {
  products(first: 50, after: $cursor) {
    edges {
      node {
        id
        title
        handle
        vendor
        productType
        tags
        status
        descriptionHtml
        category { name fullName }
        seo { title description }
        onlineStoreUrl
        totalInventory
        createdAt
        updatedAt
        publishedAt
        collections(first: 20) {
          edges { node { title handle } }
        }
        variants(first: 20) {
          edges {
            node {
              title
              price
              compareAtPrice
              sku
              barcode
              availableForSale
              inventoryQuantity
              selectedOptions { name value }
            }
          }
        }
        images(first: 5) {
          edges { node { url altText } }
        }
        metafields(first: 50) {
          edges {
            node {
              namespace
              key
              value
              type
            }
          }
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

### 4.4 GA4 Data API

**Endpoint:** `POST https://analyticsdata.googleapis.com/v1beta/{property}:runReport`

**Authentication:** Service account (`ga4_credentials.json`)

**Sample Request (Channels Report):**

```json
{
  "property": "properties/526018830",
  "dateRanges": [{"startDate": "30daysAgo", "endDate": "today"}],
  "dimensions": [{"name": "sessionDefaultChannelGroup"}],
  "metrics": [
    {"name": "sessions"},
    {"name": "totalUsers"},
    {"name": "bounceRate"},
    {"name": "engagementRate"}
  ]
}
```

**Sample Response:**

```json
{
  "rows": [
    {
      "dimensionValues": [{"value": "Organic Search"}],
      "metricValues": [
        {"value": "14164"},
        {"value": "11200"},
        {"value": "0.52"},
        {"value": "0.48"}
      ]
    }
  ]
}
```

### 4.5 GSC Search Console API

**Endpoint:** `POST https://www.googleapis.com/webmasters/v3/sites/{siteUrl}/searchAnalytics/query`

**Authentication:** Service account (`ga4_credentials.json`)

**Sample Request (Top Queries):**

```json
{
  "startDate": "2025-12-31",
  "endDate": "2026-03-31",
  "dimensions": ["query"],
  "rowLimit": 100
}
```

**Sample Response:**

```json
{
  "rows": [
    {
      "keys": ["best vitamin c serum"],
      "clicks": 34,
      "impressions": 14087,
      "ctr": 0.0024,
      "position": 5.3
    }
  ]
}
```

**IMPORTANT:** Site URL format must be `sc-domain:domain.com`, NOT `https://domain.com`.

---

## 5. Component / Service Design

| Component / Service | Responsibility | Tech Stack |
| :--- | :--- | :--- |
| **CLI Entry Point** (`brand_audit_extractor.py`, `main()`) | Parse CLI arguments, orchestrate all phases, handle errors | Python 3 stdlib (`sys.argv`) |
| **Shopify OAuth Handler** (`get_shopify_token()`) | Exchange Client ID + Secret for access token using `client_credentials` grant | `requests` |
| **Metafield Schema Discovery** (`discover_metafield_schema()`) | Query `metafieldDefinitions` to discover custom fields on the store | `requests`, Shopify GraphQL |
| **Metafield Mapper** (`build_metafield_map()`) | Match discovered metafield definitions to audit fields using keyword rules | Python stdlib |
| **GraphQL Product Fetcher** (`fetch_products_graphql()`) | Cursor-based pagination to fetch all products with metafields | `requests`, Shopify GraphQL |
| **Public JSON Fetcher** (`fetch_products_public_json()`) | Paginated fetch from `/products.json` for stores without credentials | `requests` |
| **Universal Product Extractor** (`extract_product_universal()`) | 3-layer extraction: guaranteed fields -> metafields -> description parsing | Python stdlib, `re`, `json` |
| **Metafield Finder** (`find_metafield()`) | Find metafield value by namespace.key match, then keyword fallback | Python stdlib |
| **Metafield Value Parser** (`parse_metafield_value()`) | Parse JSON lists, JSON objects, HTML content from metafield values | `json` |
| **Description Section Parser** (`parse_description_sections()`) | Split description HTML on h1-h6 tags, classify by heading keywords | `re` |
| **Ingredient Extractor** (`extract_molecules_from_text()`) | Scan text for known molecule names (30+ skincare actives) | Python stdlib |
| **Skin Type Extractor** (`extract_skin_types_from_text()`) | Map text to skin type labels using keyword dictionary | Python stdlib |
| **Certification Extractor** (`extract_certs_from_text()`) | Detect certification keywords in text | Python stdlib |
| **Tag Classifier** (`classify_tags()`) | Bucket Shopify tags into: skin_types, concerns, ingredients, categories | Python stdlib |
| **Review App Detector** (`extract_rating()`) | Detect installed review app (Judge.me, Yotpo, Loox, Stamped, Shopify, Reputon) and extract rating + count | `re`, `json` |
| **GA4 Fetcher** (`fetch_ga4()`) | Run 8 GA4 Data API reports, aggregate results | `google-analytics-data-v1beta` SDK |
| **GSC Fetcher** (`fetch_gsc()`) | Run 5 GSC Search Console API queries | `google-api-python-client` |
| **Excel Builder** (`build_excel()`) | Assemble multi-sheet Excel workbook with color coding | `openpyxl` |
| **GA4 Sheet Builder** (`_build_ga4_sheets()`) | Generate 6 GA4 sheets: Summary, Channels, Top Pages, Demographics, Locations, Devices, Weekly Trend | `openpyxl` |
| **GSC Sheet Builder** (`_build_gsc_sheets()`) | Generate 4 GSC sheets: Summary, Top Queries, Top Pages, Zero-Click Queries | `openpyxl` |

---

## 6. Database Design

### 6.1 Database Selection

| Storage Type | Technology | Purpose |
| :--- | :--- | :--- |
| **No database** | N/A | The pipeline produces Excel files as the output artifact. No persistent data store is used. |
| **File-based output** | Local filesystem (`./sheets/`) | Output Excel workbooks stored as files. Local in development. |
| **Future (not yet implemented)** | Google Cloud Storage (GCS) | Production file storage for brand audit files. Planned per System Design doc. |
| **Credentials storage** | Local `ga4_credentials.json` file | GA4/GSC service account JSON. Excluded from git. |
| **Future (not yet implemented)** | GCP Secret Manager | Shopify tokens stored securely per brand. Planned per System Design doc. |

### 6.2 Key Data Models

The pipeline works with three sets of data models, each representing a layer of the extraction output:

**Model 1: Shopify Product (output from `extract_product_universal()`)**

```
product_dict {
  # Layer 1: Guaranteed fields (always present from GraphQL)
  "Product Name": str                      # title
  "PDP URL (Clean)": str                   # onlineStoreUrl or constructed
  "Product Subtitle": str                   # from metafield
  "Selling Price (INR)": str               # e.g., "Rs.551"
  "MRP / Compare-at (INR)": str            # e.g., "Rs.599"
  "Discount %": str                         # e.g., "8%"
  "SKU / EAN": str                          # from metafield or variant sku
  "Product Type": str                       # productType
  "Pack Size": str                          # from metafield or regex from title
  "Total Inventory": str                    # totalInventory
  "Category (Taxonomy)": str                # category.fullName

  # Layer 2: Metafield-based fields
  "Rating": str                             # e.g., "4.84 / 5.0 (165 reviews)"
  "Review Count": str                       # e.g., "165"
  "Amazon Reviews": str                     # from Reputon metafield
  "Hero Molecules": str                     # e.g., "Niacinamide, Ceramide, Hyaluronic Acid"
  "Full Ingredient List": str                # INCI list from metafield
  "Clinical Claims": str                    # clinical trial results from metafield
  "Clinical Method": str                    # study methodology from description
  "How It Works": str                       # mechanism from metafield
  "Target Skin Types": str                  # e.g., "Dry Skin, Sensitive Skin"
  "Who It's For": str                       # targeting from metafield
  "Who It's NOT For": str                   # contraindications from metafield
  "Certifications": str                     # e.g., "Dermatologist Formulated, Vegan"
  "Dr. Involvement": str                   # formulator involvement from metafield
  "Doctor Profiles": str                    # metaobject refs placeholder
  "Results (Why You'll Love It)": str       # benefits from metafield
  "Texture / Experience": str              # from metafield
  "How To Use": str                         # usage instructions from metafield
  "Shelf Life & Storage": str               # from metafield
  "Before/After Images": str                # image URLs from metafield
  "Collections": str                        # collection membership
  "Related Products": str                   # metaobject refs placeholder
  "SEO Title": str                          # from seo.title
  "SEO Description": str                    # from seo.description
  "Breadcrumb": str                         # from metafield
  "Google Shopping Labels": str             # from mm-google-shopping metafields
  "Short Description": str                  # from metafield
  "Product USPs": str                       # from metafield
  "FAQ Content": str                         # from metafield
  "Published Date": str                      # publishedAt (date only)
  "Price Positioning": str                   # always "—" (manual fill)

  # Layer 3: Description/tag fallback fields (enriched if Layer 2 empty)
  # (Same fields as Layer 2, populated by regex/text parsing when metafields empty)

  # Internal fields
  "_status": str                            # ACTIVE / DRAFT
  "_tags": list                             # raw Shopify tags
  "_source": str                            # "graphql" or "public"
  "_vendor": str                            # brand/vendor name
  "_sku": str                               # variant SKU
  "_primary_image": str                     # first product image URL
}
```

**Model 2: GA4 Data (output from `fetch_ga4()`)**

```
ga4_data {
  "summary": {
    "total_sessions": int,
    "total_users": int,
    "total_pageviews": int,
    "avg_bounce_rate": float,
    "avg_engagement_rate": float,
    "date_range": "Last 30 days"
  },
  "channels": [
    {"channel": str, "sessions": int, "users": int, "bounce_rate": float, "engagement_rate": float},
    ...
  ],
  "top_pages": [
    {"page": str, "views": int, "sessions": int, "engagement_rate": float},
    ...  # top 20
  ],
  "age_groups": [{"age_group": str, "users": int}, ...],
  "gender": [{"gender": str, "users": int}, ...],
  "top_cities": [{"city": str, "sessions": int}, ...],       # top 15
  "top_countries": [{"country": str, "sessions": int}, ...],   # top 10
  "devices": [{"device": str, "sessions": int}, ...],
  "weekly_trend": [{"date": str, "sessions": int, "users": int}, ...]  # YYYYMMDD format
}
```

**Model 3: GSC Data (output from `fetch_gsc()`)**

```
gsc_data {
  "summary": {
    "total_clicks": int,
    "total_impressions": int,
    "avg_ctr": float,
    "avg_position": float,
    "date_range": "YYYY-MM-DD to YYYY-MM-DD"
  },
  "top_queries": [
    {"query": str, "clicks": int, "impressions": int, "ctr": float, "position": float},
    ...  # top 100, sorted by impressions descending
  ],
  "top_pages": [
    {"page": str, "clicks": int, "impressions": int, "ctr": float, "position": float},
    ...  # top 50
  ],
  "countries": [
    {"country": str, "clicks": int, "impressions": int, "ctr": float, "position": float},
    ...  # top 20
  ],
  "devices": [
    {"device": str, "clicks": int, "impressions": int, "ctr": float, "position": float},
    ...  # top 10
  ],
  "zero_clicks": [
    {"query": str, "clicks": 0, "impressions": int, "ctr": 0.0, "position": float},
    ...  # impressions >= 100 AND clicks == 0, sorted by impressions
  ]
}
```

### 6.3 Schema Diagram

```
EXTERNAL SYSTEMS (credentials owned by brand)
==================================================================================

  BRAND A (e.g., SkinQ)
  ┌──────────────────────────────┐
  │ Shopify Store                 │
  │ skinq.myshopify.com           │
  │ Custom App: read_products,    │
  │ read_content, read_metaobjects│
  │ Client ID: xxx, Secret: yyy   │
  │ Access Token: shpat_xxx       │
  │ (expires every 24h)           │
  └──────────────────┬────────────┘
                     │
  ┌──────────────────▼────────────┐
  │ GA4 Property ID: 526018830    │
  │ (Viewer role for Kasparro     │
  │  service account)             │
  └──────────────────┬────────────┘
                     │
  ┌──────────────────▼────────────┐
  │ GSC Site: sc-domain:skinq.com │
  │ (Viewer role for Kasparro     │
  │  service account)             │
  └──────────────────┬────────────┘

  BRAND B (e.g., Innovist)
  ┌──────────────────────────────┐
  │ Shopify Store                 │
  │ innovist.myshopify.com        │
  │ Different Client ID/Secret   │
  └──────────────────┬────────────┘
                     │
  ┌──────────────────▼────────────┐
  │ GA4 Property ID: 987654321    │
  │ (Separate GA4 property)       │
  └──────────────────┬────────────┘
                     │
  ┌──────────────────▼────────────┐
  │ GSC Site: sc-domain:innovist.com│
  └─────────────────────────────────┘


PIPELINE EXECUTION (per run)
==================================================================================

  CLI Flag Inputs (per run)
  ┌─────────────────────────────────────────────────┐
  │ --client-id       Shopify Client ID              │
  │ --client-secret   Shopify Client Secret           │
  │ --ga4-property    GA4 Property ID (integer)     │
  │ --ga4-credentials ./ga4_credentials.json         │
  │ --gsc-site        sc-domain:domain.com           │
  │ --output          ./sheets/                      │
  │ --ga4-only        (optional flag)               │
  └─────────────────────────────────────────────────┘
           │
           v
  ┌─────────────────────────────────────────────────┐
  │ EXTRACTION: brand_audit_extractor.py             │
  │                                               │
  │  Phase 1: Shopify                             │
  │    get_shopify_token()                        │
  │      -> access_token (24h TTL)                │
  │    discover_metafield_schema()                 │
  │      -> field_map (audit_field -> ns.key list)│
  │    fetch_products_graphql()                   │
  │      -> raw_products[] (cursor pagination)    │
  │    extract_product_universal() x N             │
  │      -> product_dict[] (50 fields each)       │
  │                                               │
  │  Phase 2: GA4                                 │
  │    fetch_ga4(property_id, credentials_path)     │
  │      -> ga4_data (8 report types)             │
  │                                               │
  │  Phase 3: GSC                                 │
  │    fetch_gsc(site_url, credentials_path)       │
  │      -> gsc_data (5 report types)             │
  │                                               │
  │  Phase 4: Excel Assembly                       │
  │    build_excel(products, ga4, gsc)             │
  │      -> {brand}_audit_{ts}.xlsx               │
  └─────────────────────────────────────────────────┘
           │
           v
  ┌─────────────────────────────────────────────────┐
  │ OUTPUT ARTIFACT                                │
  │                                               │
  │ ./sheets/{brand}_audit_{timestamp}.xlsx        │
  │                                               │
  │ Sheets:                                       │
  │  [1] Audit Summary (product list)             │
  │  [2] Product 1 (50 audit fields)               │
  │  [3] Product 2 (50 audit fields)               │
  │  [4] Product 3 (50 audit fields)               │
  │  [5] Product 4 (50 audit fields)               │
  │  [6] GA4 Summary (traffic overview)            │
  │  [7] GA4 Channels (channel breakdown)          │
  │  [8] GA4 Top Pages (page performance)         │
  │  [9] GA4 Demographics (age + gender)          │
  │ [10] GA4 Locations (cities + countries)       │
  │ [11] GA4 Devices (device breakdown)           │
  │ [12] GA4 Weekly Trend (30-day daily trend)    │
  │ [13] GSC Summary (search overview)           │
  │ [14] GSC Top Queries (100 queries by imp)     │
  │ [15] GSC Top Pages (50 pages by impressions) │
  │ [16] GSC Zero-Click Queries (zero-click analysis)│
  └─────────────────────────────────────────────────┘
```

### 6.4 Migrations

N/A -- there is no database. All state is transient (CLI execution) or file-based (Excel output).

### 6.5 State Management

| State | Where Stored | Lifetime |
| :--- | :--- | :--- |
| Shopify access token | In-memory (returned by `get_shopify_token()`) | Per run (24h max TTL) |
| GA4 credentials | `ga4_credentials.json` file on disk | Until rotated |
| Extraction output | `./sheets/{brand}_audit_{ts}.xlsx` | Persistent file |
| Metafield schema | In-memory (returned by `discover_metafield_schema()`) | Per run |

---

## 7. Caching Strategy

| Cache Layer | What is Cached | TTL | Tool |
| :--- | :--- | :--- | :--- |
| Shopify products | Not cached -- fetched fresh on every run | N/A | N/A |
| GA4 data | Not cached -- 8 fresh API calls per run | N/A | N/A |
| GSC data | Not cached -- 5 fresh API calls per run | N/A | N/A |
| Metafield schema | Not cached -- queried on every run | N/A | N/A |
| Metaobject data | Metaobject GIDs are not resolved (would need `read_metaobjects` scope and separate API call) | N/A | N/A |

**Cache Invalidation Strategy:** No caching layer exists. Every extraction run fetches all data fresh. This is appropriate because:
1. Shopify product data changes frequently (prices, inventory, descriptions)
2. GA4/GSC data is time-sensitive (last 30/90 days)
3. The script is manually triggered (not polling in a loop)
4. Brand isolation means no cross-request state

**Future consideration:** If weekly light extraction (`--ga4-only`) is implemented as a scheduled job, caching Shopify product data for 7 days would save API calls. This is not yet implemented.

---

## 8. Message Queue & Async Processing

N/A -- the system does not use a message queue or async processing framework. The extraction runs as a synchronous, sequential pipeline:

```
main():
    1. get_shopify_token()     -- synchronous HTTP POST
    2. discover_metafield_schema()  -- synchronous GraphQL POST
    3. fetch_products_graphql()  -- synchronous GraphQL POSTs (paginated)
    4. extract_product_universal() -- in-memory processing
    5. fetch_ga4()              -- synchronous SDK calls (8 reports)
    6. fetch_gsc()              -- synchronous SDK calls (5 queries)
    7. build_excel()            -- in-memory Excel generation
    8. wb.save()                -- single file write
```

**Rationale:** Each step's output is an input to the next step (except GA4/GSC which are independent of Shopify but both feed into `build_excel()`). There is no opportunity for parallelization of the core pipeline. The only parallel opportunity (fetching GA4 and GSC simultaneously while processing Shopify data) is not used because:
1. GA4/GSC credentials are only available after Shopify token is obtained
2. Excel assembly requires all three data sources to be complete
3. The script is simple and adding async complexity would reduce maintainability

---

## 9. Load Balancing & Scalability

### 9.1 Load Balancing

N/A -- the system is a single-threaded CLI script. There is no server, no load balancer, and no concurrent requests.

### 9.2 Scaling Strategy

| Component | Current Limit | Bottleneck |
| :--- | :--- | :--- |
| Brands | 1 extraction at a time | Single CLI process |
| Shopify products | All products fetched (pagination handles large stores) | GraphQL pagination (40 req/min rate limit) |
| GA4 reports | 8 API calls per extraction | GA4 quota: 10,000 req/day per property |
| GSC reports | 5 API calls per extraction | GSC row limit: 25,000 rows/request |
| Excel rows | Memory-bound by openpyxl | ~10,000 rows maximum per sheet before performance issues |
| Concurrent extractions | Manual -- run script multiple times for multiple brands | No automated queue |

**Horizontal scaling:** If parallel execution for multiple brands is needed, the operator runs the script multiple times (each with different credentials). No infrastructure change is required -- just additional CLI invocations.

**Vertical scaling:** The script's memory footprint is small (~50-200 MB for typical extraction). It runs comfortably on any laptop or server.

**Rate limit considerations:**
- Shopify: 40 req/min per store per app. For 4 focus products, this is well within limits even with pagination.
- GA4: 10,000 req/day per property. Even running 50 extractions per day per brand, this is 400 GA4 requests/day -- well within quota.
- GSC: No documented rate limit. The script uses 5 requests per extraction.

---

## 10. Security Design

| Concern | Solution |
| :--- | :--- |
| **Shopify Credentials** | Client ID + Client Secret passed via CLI flags per run. Never stored in code or committed to git. |
| **GA4/GSC Credentials** | `ga4_credentials.json` file on disk (excluded from git via `.gitignore`). Same file used for both GA4 and GSC. Single GCP service account. |
| **Service Account** | `kasparro-ga4-reader-kasparro-p@kasparro-audit-pillar-b.iam.gserviceaccount.com`. Viewer role only (read-only). Same account for all brands. |
| **GSC Site URL Format** | Must use `sc-domain:domain.com` format. Using `https://domain.com` will fail silently. The script validates this format. |
| **Brand Data Isolation** | Each extraction run is independent. Credentials are passed per-run. No shared state between runs. |
| **Shopify Token Expiry** | Tokens expire every 24 hours. The script requests a fresh token on every run via OAuth client credentials grant. No token storage or refresh logic needed. |
| **Metaobject GIDs** | Metaobject references (`gid://shopify/Metaobject/...`) in metafield values are detected and shown as placeholders. `read_metaobjects` scope is needed to resolve them. |
| **File Permissions** | Output files (`./sheets/*.xlsx`) are stored on the local filesystem. No access control beyond OS file permissions. |
| **Secrets in Logs** | No secrets are printed to stdout. The script only logs: store URL (domain only), brand name, record counts, error messages. |

**IMPORTANT SECURITY RULES (from auto-memory, enforced in code):**
- OAuth endpoint must be `{store}/admin/oauth/access_token` -- NOT `https://shopify.com/authentication/oauth/token`
- Content-Type for Shopify token request must be `application/x-www-form-urlencoded` -- NOT `application/json`
- GSC site URL must use `sc-domain:domain.com` format -- NOT `https://domain.com`

---

## 11. Fault Tolerance & Reliability

| Strategy | Implementation |
| :--- | :--- |
| **Shopify failure** | Script exits with CLI error. GA4 and GSC are skipped. Partial data is NOT saved (fail-fast). |
| **GA4 failure** | Script prints error, continues to GSC and Excel build. Excel output includes Shopify + GSC data only. |
| **GSC failure** | Script prints error, continues to Excel build. Excel output includes Shopify + GA4 data only. |
| **No data from any source** | Script exits with error: "No data found. Exiting." |
| **Token expiry during run** | Shopify tokens are requested at the start of the run. If the token expires mid-run (unlikely in <24h execution), subsequent Shopify API calls would return 401. |
| **Metafield scope missing** | If `read_metaobjects` scope is not granted, metafield definitions may be incomplete. Metaobject GIDs are shown as placeholders `[metaobject references]`. |
| **Pagination boundary** | Cursor-based pagination continues until `hasNextPage: false`. No risk of infinite loops. |
| **Invalid GSC site URL** | GSC API returns error if URL format is wrong. Error is caught and logged. |
| **File write failure** | `openpyxl.Workbook.save()` throws on I/O error. No rollback of partial data. |
| **GA4-only mode failure** | If `--ga4-only` is set but no GA4 or GSC data is available, script exits with error. |
| **Graceful degradation** | The script is resilient to missing metafields -- Layer 3 description fallback always provides some data. |
| **Review app not detected** | Rating/Review Count fields default to `"—"`. Multiple app patterns are checked. |

**Partial data handling:** When one source fails, the Excel workbook is still generated with available data. The brand engineer is notified via error message in stdout. This is logged for investigation.

---

## 12. Monitoring & Observability

| Pillar | What it tracks | Tool |
| :--- | :--- | :--- |
| **Logging** | Structured print statements to stdout. Each phase prints its name, record count, and status. | `print()` to stdout |
| **Error Tracking** | Exceptions printed with traceback. HTTP errors include status code and response text. | `try/except` blocks + `traceback.print_exc()` |
| **Success tracking** | Final summary printed: brand name, product count, active/draft breakdown, GA4/GSC record counts, output file path. | `print()` at end of `main()` |
| **Rate limit monitoring** | No monitoring. Shopify rate limit errors surface as HTTP 429 and cause script failure. | N/A |
| **API call tracking** | Each GraphQL page fetch prints page number and record count. GA4/GSC each print a summary of records extracted. | `print()` in each fetch function |
| **Cost tracking** | GA4 and GSC APIs are free (included in GA4/GSC subscriptions). No per-call cost tracking. | N/A |

**Key metrics output on every run:**
```
Brand Audit Extractor (Universal)
  Store:   skinq.myshopify.com
  Mode:    Admin API + GraphQL
  Brand:   SkinQ
  Fetching GA4 data from property 526018830...
  GA4: 14,164 sessions, 11,200 users (30 days)
  Fetching GSC data from site sc-domain:skinq.com...
  GSC: 4,500 clicks, 615,381 impressions (90 days)
  Building Excel (1 summary + 4 product sheets + 6 GA4 + 4 GSC)...
  Done!
  Products:   24 total (4 active, 20 draft)
  GA4:        14,164 sessions (Last 30 days)
  GSC:        4,500 clicks (90 days)
  Output:     ./sheets/skinq_com_audit_20260401_120000.xlsx
```

---

## 13. Deployment Architecture

### 13.1 Current Deployment (Local CLI)

The pipeline runs entirely on the operator's local machine (laptop/server). No cloud infrastructure is deployed.

| Aspect | Current State |
| :--- | :--- |
| **Runtime environment** | Python 3.11+ on operator's machine |
| **Installation** | `pip install requests openpyxl google-analytics-data google-api-python-client` |
| **Credentials** | `ga4_credentials.json` file on disk at working directory |
| **Execution** | Manual via CLI: `python brand_audit_extractor.py ...` |
| **Output** | `./sheets/` directory on local filesystem |
| **Scheduling** | None -- manual execution per client per need |

### 13.2 Planned Cloud Deployment (from System Design doc)

| Aspect | Planned State |
| :--- | :--- |
| **App hosting** | Google Cloud Run (serverless containers) |
| **File storage** | Google Cloud Storage (GCS) bucket per brand |
| **Credentials storage** | GCP Secret Manager (Shopify tokens per brand) |
| **Scheduling** | Cloud Scheduler (weekly trigger for light extraction) |
| **Trigger** | HTTP POST from onboarding system or Cloud Scheduler |
| **CI/CD** | Google Cloud Build (Docker build + push + Cloud Run deploy) |

**Command template for Cloud Run:**
```
POST https://{cloud-run-url}/v1/extraction/run/{brand_id}
Headers: X-API-Key: {operator_key}
Body: {}  (credentials fetched from Secret Manager)
```

### 13.3 Dependencies

The script requires the following Python packages (no `requirements.txt` file exists -- install manually):

```bash
pip install requests                     # HTTP client for Shopify REST + GraphQL
pip install openpyxl                     # Excel workbook generation
pip install google-analytics-data        # GA4 Data API SDK (BetaAnalyticsDataClient)
pip install google-api-python-client    # GSC Search Console API SDK
pip install google-auth                 # Authentication for GCP APIs
```

---

## 14. Third-Party Integrations

| Integration | Purpose | Provider | Auth Method |
| :--- | :--- | :--- | :--- |
| **Shopify Admin API** | Product data, metafields, collections, SEO, variants, images | Shopify | OAuth client credentials (`client_id` + `client_secret` -> `access_token`) |
| **Shopify Public JSON** | Product data without authentication (no metafields) | Shopify | None (public endpoint) |
| **GA4 Data API** | Traffic, sessions, demographics, channels, top pages, devices, weekly trend | Google Analytics | Service account (`ga4_credentials.json`) |
| **GSC Search Console API** | Search queries, impressions, clicks, CTR, position, zero-click analysis | Google Search Console | Service account (same `ga4_credentials.json`) |
| **Review Apps (Judge.me)** | Product ratings and review counts | Judge.me | Extracted from Shopify metafields |
| **Review Apps (Yotpo)** | Product ratings and review counts | Yotpo | Extracted from Shopify metafields |
| **Review Apps (Loox)** | Product ratings and review counts | Loox | Extracted from Shopify metafields |
| **Review Apps (Stamped)** | Product ratings and review counts | Stamped | Extracted from Shopify metafields |
| **Review Apps (Amazon/Reputon)** | Amazon review data via Shopify metafield | Reputon | Extracted from Shopify metafields |

**Service Account Details:**
- **Email:** `kasparro-ga4-reader-kasparro-p@kasparro-audit-pillar-b.iam.gserviceaccount.com`
- **Project:** `kasparro-audit-pillar-b`
- **Credentials file:** `ga4_credentials.json` (excluded from git)
- **Scopes:** `https://www.googleapis.com/auth/analytics.readonly` (GA4), `https://www.googleapis.com/auth/webmasters.readonly` (GSC)
- **Access model:** Single fixed service account used for ALL brands. Kasparro service account is added as Viewer on each brand's GA4 property and GSC site.

---

## 15. Key Design Decisions & Trade-offs

| Decision | Option A | Option B | Chosen | Reason |
| :--- | :--- | :--- | :--- | :--- |
| Architecture | Web API (FastAPI) with background jobs | Standalone CLI script | **CLI script** | Simpler to operate. No server to maintain. Credentials passed per-run. Appropriate for manual operator-triggered execution. |
| Shopify auth | Per-brand access tokens stored in DB | OAuth client credentials per run | **Client credentials per run** | Tokens expire every 24h. Fetching fresh token each run eliminates token storage complexity. |
| OAuth endpoint | Store-specific (`/admin/oauth/access_token`) | Shopify global endpoint | **Store-specific** | The store-specific endpoint is the correct Shopify OAuth flow for Custom Apps. Global endpoints return errors. |
| Content-Type | `application/json` | `application/x-www-form-urlencoded` | **x-www-form-urlencoded** | Shopify's OAuth token endpoint requires form-encoded body. JSON returns 400 errors. |
| Metafield handling | Hardcoded per-store key names | Auto-discover schema + keyword mapping | **Auto-discover + keyword mapping** | No two Shopify stores have identical metafield names. Auto-discovery makes the script universally reusable without per-client code changes. |
| 3-layer extraction | Only metafields | Only description parsing | **3-layer: guaranteed -> metafields -> description** | Metafields may be empty on some stores. Description parsing provides fallback. Guaranteed fields provide consistency. |
| Metaobject resolution | Skip entirely | Fetch metaobjects separately | **Skip (show placeholder)** | Resolving metaobject GIDs requires `read_metaobjects` scope + additional API calls. Showing placeholder is honest about what the script can do without that scope. |
| Review detection | Single app pattern | Multi-app detection | **Multi-app detection order** | Different brands install different review apps. Checking multiple patterns in priority order maximizes detection coverage. |
| GA4/GSC credentials | Separate files per brand | Single shared service account JSON | **Single file for both GA4 and GSC** | Same GCP service account is used for all brands' GA4 and GSC. One `ga4_credentials.json` file. |
| GSC URL format | `https://domain.com` | `sc-domain:domain.com` | **sc-domain:domain.com** | GSC API only accepts domain-prefixed format. `https://` URLs return 400 errors. |
| Output format | JSON | Excel workbook | **Excel workbook** | Brand teams receive Excel for validation. GA4/GSC reports are tabular. Multi-sheet workbook matches existing manual process. |
| Product scope | All products | Top 4 products | **Top 4 (all products fetched, top 4 in output)** | Manual extraction only covers top products. The spec says "extract top 4 products." Implementation fetches all (for metafield schema discovery) then focuses the output on 4. |
| Concurrent source fetching | Fetch Shopify, GA4, GSC in parallel | Sequential extraction | **Sequential** | GA4/GSC credentials are independent of Shopify but Excel assembly requires all three. Sequential is simpler and sufficient for single-run execution. |
| Excel sheet color coding | Single color | White/yellow/orange per field | **White + Orange per field** | Green = auto-filled, Yellow/Orange = manual fill required. Matches the existing SkinQ validation spreadsheet format. |
| Error handling | Continue on source failure | Fail fast | **Fail fast for Shopify, continue on GA4/GSC** | Shopify failure means no product data (the core output). GA4/GSC failures mean partial output (better than nothing). |

---

## 16. Future Considerations / Out of Scope

- **Web API wrapper (FastAPI):** Converting the CLI script to a FastAPI web service with endpoints for running extractions, checking status, and retrieving results. Enables programmatic access and scheduled triggers. Referenced in System Design doc but not yet implemented.
- **GCP Secret Manager integration:** Storing Shopify access tokens per brand in GCP Secret Manager instead of passing them via CLI flags. Referenced in System Design doc (Section 6). Not yet implemented.
- **Google Cloud Storage output:** Storing output Excel files in GCS instead of the local `./sheets/` directory. Enables multi-operator access and audit trail. Referenced in System Design doc. Not yet implemented.
- **Cloud Run deployment:** Containerizing the script and deploying to Google Cloud Run with Cloud Scheduler for weekly light extraction triggers. Referenced in System Design doc (Section 23-26). Not yet implemented.
- **Cloud Scheduler weekly trigger:** Automated weekly light extraction (`--ga4-only`) running every Sunday at midnight via Cloud Scheduler -> Cloud Run HTTP call. Referenced in System Design doc. Not yet implemented.
- **Baseline capture and delta computation:** Storing the first extraction as a frozen JSON baseline in GCS, then computing week-over-week deltas for attribution tracking. Referenced in System Design doc (Section 10.3-10.4). Not yet implemented.
- **Deployment log:** Notion database or JSON file tracking Kasparro actions (what was deployed, when, which URLs/queries affected, expected impact) for attribution confidence levels (Direct / Correlated / Ambient / External). Not yet implemented.
- **JSON snapshot alongside Excel:** The System Design doc (Section 4.3) specifies that full extraction produces a JSON snapshot in addition to the Excel file. Currently only Excel is produced. JSON output is not implemented.
- **Per-product GA4/GSC attribution:** Currently GA4/GSC data is at the property/site level. Future enhancement: filter GA4/GSC data per product using product page URLs from Shopify. Not yet implemented.
- **Non-Shopify brand support:** The current script is Shopify-only. Future: abstraction layer for other e-commerce platforms (WooCommerce, Magento, BigCommerce). Not yet planned.
- **AI-powered enrichment:** Using LLM (OpenAI GPT-4o-mini) to extract structured data from product descriptions, generate clinical claim summaries, and identify ingredient claims. Referenced in System Design doc. Not yet implemented.
- **ReportLab PDF output:** Automated PDF attribution summary via ReportLab (mentioned in System Design doc Section 10.6). Currently only Excel is produced. Not yet implemented.
- **Webhook notifications:** Sending Slack/email notifications when extraction fails or when weekly delta exceeds threshold. Not yet implemented.
- **Requirements.txt:** No `requirements.txt` file exists. Dependencies are documented in the script's docstring. A proper `requirements.txt` should be created for reproducibility.
- **Tests:** No automated tests exist for the pipeline. Unit tests for individual functions (metafield mapping, molecule extraction, tag classification, review app detection) would improve reliability.

---

## 17. Shopify Product Extraction Detail

### 17.1 Audit Field Schema

The pipeline extracts ~40 audit fields per product, organized into 10 categories:

| Category | Fields |
| :--- | :--- |
| **Basic Product Info** | Product Name, PDP URL (Clean), Product Subtitle, Selling Price (INR), MRP / Compare-at (INR), Discount %, SKU / EAN, Product Type, Pack Size, Total Inventory, Category (Taxonomy) |
| **Ratings & Reviews** | Rating, Review Count, Amazon Reviews |
| **Ingredients & Science** | Hero Molecules, Full Ingredient List, Clinical Claims, Clinical Method, How It Works |
| **Targeting** | Target Skin Types, Who It's For, Who It's NOT For |
| **Trust & Authority** | Certifications, Dr. Involvement, Doctor Profiles |
| **Product Experience** | Results (Why You'll Love It), Texture / Experience, How To Use, Shelf Life & Storage, Before/After Images |
| **Shopability** | Collections, Related Products |
| **SEO & Site** | SEO Title, SEO Description, Breadcrumb, Google Shopping Labels |
| **Marketing** | Short Description, Product USPs, FAQ Content |
| **Admin** | Published Date, Price Positioning |

### 17.2 Metafield Mapping Rules

The `METAFIELD_MAPPING_RULES` dictionary maps audit fields to possible metafield key names. Auto-discovery uses `metafieldDefinitions` to find actual namespace+key pairs on the store, then keyword-matches against this dictionary:

```
hero_molecules:         ["ingredient", "active_ingredient", "key_ingredient", "hero_molecule", "product_ingredients"]
full_ingredient_list:  ["ingredient_list", "inci", "full_ingredient", "active_ingredients"]
target_skin_types:      ["skin_type", "who_is_it_for", "suitable_for", "target_skin"]
clinical_claims:        ["clinical", "proven_result", "efficacy", "clinically_proven"]
how_it_works:           ["how_it_works", "mechanism", "science_behind"]
certifications:         ["certification", "cruelty", "vegan", "organic_cert"]
doctor_involvement:     ["doctor", "formulator", "expert", "dermatologist"]
subtitle:               ["subtitle", "sub_title", "short_description", "tagline"]
who_its_for:            ["who_is_it_for", "suitable_for", "ideal_for"]
how_to_use:             ["how_to_use", "direction", "usage", "application"]
pack_size:              ["size", "volume", "weight", "net_weight", "quantity"]
shelf_life:             ["shelf_life", "dermatologist_recommended", "storage"]
texture_experience:     ["texture_experience"]
who_shouldnt:           ["shouldn_t", "who_shouldn", "should_not"]
results_why_love:       ["results", "why_love", "why_choose", "why_choose_skinq"]
detailed_description:   ["detailed_product_description"]
short_desc_field:       ["short_description", "product_short_description"]
usps:                   ["usps", "product_usps"]
faq_content:            ["faq", "product_faq", "faqs"]
ean_code:               ["ean_code"]
breadcrumb:              ["breadcrumb", "custom_breadcrumb"]
google_labels:          ["custom_label"]
```

### 17.3 Review App Detection

The `extract_rating()` function checks metafields for known review app patterns in priority order:

| Priority | App | Namespace | Rating Key | Count Key | Parser |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Shopify Reviews | `reviews` | `rating` | `rating_count` | JSON (`{"value": "4.81"}`) |
| 2 | Judge.me | `judgeme` | `badge` | (HTML attr) | HTML regex (`data-average-rating`) |
| 3 | Yotpo | `yotpo` | `reviews_average` | `reviews_count` | Direct float |
| 4 | Loox | `loox` | `avg_rating` | `num_reviews` | Direct float |
| 5 | Stamped | `stamped` | `reviews_average` | `reviews_count` | Direct float |
| 6 | Automizely | `automizely_reviews` | `ratings` | `raters` | Direct |
| 7 | Amazon (Reputon) | `reputon` | `rating` (JSON) | `reviewsNumber` | JSON |

### 17.4 Known Molecule List

The `MOLECULES` list contains 30 skincare active ingredients used by `extract_molecules_from_text()` to identify hero molecules from ingredient lists and descriptions:

```
niacinamide, salicylic acid, hyaluronic acid, vitamin c, retinol, azelaic acid,
glycerin, ceramide, squalane, kojic acid, alpha arbutin, tea tree, aloe vera,
collagen, peptide, zinc, centella, cica, sodium hyaluronate, evening primrose,
spf, lactic acid, glycolic acid, benzoyl peroxide, tranexamic acid,
argan oil, rosehip oil, bakuchiol, malic acid
```

---

## 18. Excel Workbook Structure

### 18.1 Sheet Inventory

| # | Sheet Name | Source | Content | Rows (per brand) |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Audit Summary | Shopify | Product list with key fields (name, price, type, rating, category) | 4 + header |
| 2 | Product 1 | Shopify | All 40 audit fields + action mapping | 40 + headers |
| 3 | Product 2 | Shopify | All 40 audit fields + action mapping | 40 + headers |
| 4 | Product 3 | Shopify | All 40 audit fields + action mapping | 40 + headers |
| 5 | Product 4 | Shopify | All 40 audit fields + action mapping | 40 + headers |
| 6 | GA4 Summary | GA4 | Traffic overview (sessions, users, pageviews, bounce/engagement rate) | 10 |
| 7 | GA4 Channels | GA4 | All channel breakdown (sessions, users, bounce rate, engagement rate, % of total) | All channels + header |
| 8 | GA4 Top Pages | GA4 | Top 20 pages (views, sessions, engagement rate, PDP/Collection note) | 20 + header |
| 9 | GA4 Demographics | GA4 | Age groups (users + %) + Gender split (users + %) | Age groups + genders + headers |
| 10 | GA4 Locations | GA4 | Top 15 cities (sessions) + Top 10 countries (sessions) | 25 + headers |
| 11 | GA4 Devices | GA4 | Device breakdown (sessions, % of total) | All devices + header |
| 12 | GA4 Weekly Trend | GA4 | 30-day daily sessions + users (YYYY-MM-DD format) | 30 + header |
| 13 | GSC Summary | GSC | Search overview (clicks, impressions, CTR, avg position, date range) | 9 |
| 14 | GSC Top Queries | GSC | Top 100 queries by impressions (clicks, impressions, CTR, position) | 100 + header |
| 15 | GSC Top Pages | GSC | Top 50 pages by impressions | 50 + header |
| 16 | GSC Zero-Click Queries | GSC | Zero-click queries (impressions >= 100, clicks == 0) | Variable + header |

### 18.2 Cell Color Coding

| Color | Hex | Meaning | Fields |
| :--- | :--- | :--- | :--- |
| White | `#FFFFFF` | Auto-filled by pipeline | All fields except manual fields |
| Orange | `#FFF2CC` | Manual fill required | Price Positioning, Dr. Involvement, Clinical Method |
| Gray | `#D9D9D9` | Column headers | All header rows |
| Light Gray | `#F2F2F2` | Alternating row shading | Summary sheet data rows |

### 18.3 Column Structure (Product Sheets)

Each product sheet has 4 columns:

| Column | Width | Content |
| :--- | :--- | :--- |
| A | 28 | Field name |
| B | 55 | Extracted value (max 500 chars, truncated) |
| C | 55 | Audit Input Mapping (what this field is used for) |
| D | 40 | Action for brand team (Validate / Confirm / FILL / CRITICAL) |

---

## 19. Extraction Scope Rules

### 19.1 TOP 4 Products

The spec states "Always extract TOP 4 products only." The implementation fetches ALL products from Shopify (for metafield schema discovery) but generates product sheets only for the first 4 products returned by the Shopify GraphQL API.

The ordering of products in Shopify GraphQL is not guaranteed to be by sales/revenue. In practice, the first 4 products are typically the ones most recently updated or created. For production use, a handle allowlist should be used to explicitly specify which 4 products to include.

**Current behavior:** `products[0:4]` from the paginated GraphQL response. No ordering by sales data.

**Recommended improvement:** Accept `--focus-product-handles` flag (list of handles) to explicitly select the 4 products, matching the System Design doc specification.

### 19.2 Metaobject GID Handling

Metaobject references appear in metafield values as:
```json
["gid://shopify/Metaobject/123456789"]
```

The `parse_metafield_value()` function detects these and returns `[N metaobject references]` as a placeholder. The `read_metaobjects` scope in the Shopify Custom App is needed to resolve these via a separate GraphQL query:

```graphql
{
  metaobject(id: "gid://shopify/Metaobject/123456789") {
    id
    type
    fields {
      key
      value
    }
  }
}
```

---

## 20. Data Model Reference (Audit Input Mapping)

This section maps each audit field to its downstream use in Kasparro's audit pipeline (from `AUDIT_FIELDS` constant):

| Audit Field | Audit Input Mapping | Action |
| :--- | :--- | :--- |
| Product Name | Display name in report | Validate |
| PDP URL (Clean) | `pdp_url` -> Machine Readiness audit | Confirm canonical URL |
| Product Subtitle | Short tagline / product sub title | Validate |
| Selling Price (INR) | Class B: `{price_range}` -- CRITICAL for pricing hallucination fix | Validate -- actual selling price? |
| MRP / Compare-at (INR) | Discount context | Confirm |
| Discount % | Price positioning signal | -- |
| SKU / EAN | Product identifiers | Validate |
| Product Type | Class A: `{product_type}` | Validate |
| Pack Size | Class B: `{unit}` | FILL -- exact ml/g |
| Total Inventory | Stock level | -- |
| Category (Taxonomy) | Shopify standard taxonomy path | Confirm -- is category correct? |
| Rating | Brand Perception evidence (Judge.me) | -- |
| Review Count | Social proof signal | -- |
| Amazon Reviews | Cross-platform social proof | Confirm listing URL |
| Hero Molecules | Class B: `{attributes}` -- feeds query templates | Validate -- any key ingredients missing? |
| Full Ingredient List | Complete INCI list from metafields | Verify completeness |
| Clinical Claims | Claim Compliance Framework -- must map to clinical trial source | CRITICAL -- provide study references |
| Clinical Method | Evidence grounding | FILL -- study name, sample size, duration |
| How It Works | Mechanism / science behind the product | Validate |
| Target Skin Types | Class C: `{specific_conditions}` -- feeds condition queries | Validate -- any conditions missing? |
| Who It's For | Detailed targeting from metafields | Validate |
| Who It's NOT For | Contraindications | FILL -- any safety concerns? |
| Certifications | Trust signals for E-E-A-T | Confirm -- are certifications correct? |
| Dr. Involvement | Authority signal for E-E-A-T | FILL -- did a formulator/expert create this product? |
| Doctor Profiles | Doctor metaobject references | FILL -- doctor names + credentials |
| Results (Why You'll Love It) | Claims and benefits listed on PDP | Validate |
| Texture / Experience | How the product feels on skin | Validate |
| How To Use | Step-by-step usage instructions | Validate |
| Shelf Life & Storage | Longevity and storage instructions | Confirm |
| Before/After Images | Visuals from clinical photography | Confirm -- can Kasparro use? |
| Collections | Store categorization / collection membership | -- |
| Related Products | Cross-sell opportunities | FILL -- validate recommendations |
| SEO Title | Meta title | Validate |
| SEO Description | Meta description | Validate |
| Breadcrumb | Navigation breadcrumb path | Confirm |
| Google Shopping Labels | `mm-google-shopping` custom labels | FILL -- product categorization |
| Short Description | Product elevator pitch | Validate |
| Product USPs | Unique selling propositions | Validate |
| FAQ Content | Product-specific FAQs from PDP | Review |
| Published Date | When product went live | -- |
| Price Positioning | CRITICAL -- how does brand position pricing? | FILL -- budget, value-clinical, mid-range, or premium? |

---

*End of HLD Document*
*Generated by analyzing the codebase -- code is the source of truth.*
*Reference: `brand_audit_extractor.py` (1,728 lines, the single source of truth for implementation)*
