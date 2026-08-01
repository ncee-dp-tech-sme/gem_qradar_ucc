Prompt: Create a QRadar Universal Cloud Connector that retrieves open issues from Guardium Exposure Manager (GEM).

Date completed: 2026-08-01

---

## Files created

| File | Purpose |
|---|---|
| `GEM_QRadar_UCC/GEM-Workflow.xml` | Main workflow — fetches open GEM issues and posts them to QRadar |
| `GEM_QRadar_UCC/GEM-WorkflowParameterValues.xml` | Parameter values template to fill in before deploying |

---

## Investigation findings

### Schema namespaces
- Workflow: `http://qradar.ibm.com/UniversalCloudRESTAPI/Workflow/V2`
- Parameter values: `http://qradar.ibm.com/UniversalCloudRESTAPI/WorkflowParameterValues/V2`

### Authentication
- The XSD supports `<BasicAuthentication>`, `<BearerAuthentication>`, and manual `<RequestHeader>`.
- The Randori example uses `base64_encode()` as a built-in function for query-parameter encoding.
- For GEM Basic auth, `apikey:apisecret` is assembled at runtime via `<Set>` and then base64-encoded inline:
  `<RequestHeader name="authorization" value="Basic ${base64_encode(/gem/credentials)}"/>`

### Date range
- `time()` returns current epoch in milliseconds.
- Today UTC midnight = `time() - (time() mod 86400000)`
- Yesterday UTC midnight = today midnight − 86400000
- Formatted with `<FormatDate pattern="yyyy-MM-dd'T'HH:mm:ss'Z'" timeZone="UTC"/>`

### URL-encoded date filter
- Decoded: `{"preset":"custom","from":"<from>","to":"<to>"}`
- Encoded: `%7B%22preset%22%3A%22custom%22%2C%22from%22%3A%22<from>%22%2C%22to%22%3A%22<to>%22%7D`

### Pagination
- `DoWhile condition="${count(/gem/response/body/issues)} = ${/gem/limit}"` — runs while a full page (20) is returned; exits on partial/empty page.
- Offset incremented by `${/gem/offset + /gem/limit}` after each page.

### Event emission
- `<PostEvents path="/gem/response/body/issues" source="${/gem_host}"/>` — batch post of all issues on the page.

### Error handling
- `<If condition="${/gem/response/status_code} != 200">` → `<Log type="ERROR">` + `<Abort>`

---

## Validation results

| Check | Result |
|---|---|
| Workflow namespace matches XSD | ✅ |
| Parameter-values namespace matches XSD | ✅ |
| Parameters declared in workflow match values file | ✅ gem_host, api_key, api_secret |
| `secret="true"` on api_key and api_secret | ✅ |
| Date filter URL encoding correct | ✅ %7B%22preset%22%3A... |
| Authorization header format | ✅ `Basic ${base64_encode(/gem/credentials)}` |
| All 4 query parameters present | ✅ offset, limit, filters.status[], filters.date_detected |
| Pagination DoWhile terminates on partial page | ✅ |
| PostEvents used for batch emission | ✅ |
| Error abort on non-200 | ✅ |
| Tests block: DNS, TCP, SSL, Proxy | ✅ |
