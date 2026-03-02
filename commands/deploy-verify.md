---
description: Post-deploy verification with health checks and smoke tests
allowed-tools: Bash, Read
---

# Deploy Verify

Verify deployment: $ARGUMENTS

## Instructions

1. **Identify deployment target**:
   - If argument is a URL: use that directly
   - If no argument: check for deployment URL in config, environment, or recent deploy output

2. **Health checks**:
   ```bash
   # HTTP status check
   curl -s -o /dev/null -w "%{http_code}" <url>

   # Response time check
   curl -s -o /dev/null -w "%{time_total}" <url>

   # Check key pages
   for path in "/" "/api/health" "/docs"; do
     status=$(curl -s -o /dev/null -w "%{http_code}" "<url>$path")
     echo "$path: $status"
   done
   ```

3. **Smoke tests**:
   - Verify homepage loads (200 OK)
   - Verify API health endpoint (if applicable)
   - Verify critical user flows are accessible
   - Check for error responses or redirects

4. **Rollback criteria**:
   - Any 5xx errors on critical paths → ROLLBACK
   - Response time > 5x baseline → INVESTIGATE
   - Missing critical content → ROLLBACK
   - API health check failing → ROLLBACK

5. **Monitoring window**:
   - Recommend 24h monitoring period
   - Define what metrics to watch
   - Set thresholds for escalation

6. **Report**:
   ```
   ## Deploy Verification Report

   URL: <deployment url>
   Status: PASS | FAIL | WARN
   Time: <timestamp>

   ### Health Checks
   | Endpoint | Status | Response Time |
   |----------|--------|---------------|
   | / | 200 | 120ms |

   ### Rollback Needed: YES / NO
   ### Reason: <if YES>
   ```
