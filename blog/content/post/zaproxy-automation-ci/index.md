+++
author = "Eduard Marbach"
title = "API Security Scanning with ZAP Automation Framework in CI Pipelines"
date = "2025-11-12"
description = "Learn how to integrate OWASP ZAP's Automation Framework directly into your CI/CD pipelines for automated API security scanning without wrapper scripts"
tags = [
    "security",
    "zaproxy",
    "api-testing",
    "ci-cd",
    "devsecops",
    "automation",
]
categories = [
    "security",
    "devops",
]
+++

Security scanning should be an integral part of your CI/CD pipeline, not an afterthought. In this guide, I'll show you how to use OWASP ZAP's Automation Framework directly in your pipelines—no wrapper scripts needed—for comprehensive API security testing.

<!--more-->

## Why the Automation Framework?

OWASP ZAP (Zed Attack Proxy) is a powerful open-source security testing tool. While many CI integrations rely on wrapper scripts or custom actions, the **ZAP Automation Framework** provides a native, configuration-as-code approach that gives you:

- ✅ **Full control** over scan configuration
- ✅ **No dependency** on third-party wrappers
- ✅ **Version-controlled** security policies
- ✅ **Flexible authentication** handling
- ✅ **Consistent behavior** across environments

## The Challenge: API Authentication

Most APIs require authentication, which presents a challenge for automated scanning. ZAP's traditional authentication methods can be complex to configure. The Automation Framework solves this elegantly with **inline scripts** that inject headers dynamically.

## Project Structure

Here's how to organize your ZAP automation files:

```
.ci/
└── zap/
    ├── automation.yaml           # Main scan configuration
    ├── automation-auth-check.yaml # Authentication verification
    └── localtest.sh              # Local testing script
```

## Configuration Breakdown

### 1. Authentication Check

Before running a full scan, verify your credentials work:

```yaml
# automation-auth-check.yaml
env:
  name: API Scan Environment
  contexts:
    - name: "TestContext"
      urls:
        - "${ZAP_TARGET}"
  parameters:
    failOnError: true
    failOnWarning: true

jobs:
  # Inject authentication headers via inline script
  - type: script
    parameters:
      action: add
      type: httpsender
      engine: "ECMAScript : Graal.js"
      name: AddHeader.js
      inline: |
        function sendingRequest(msg, initiator, helper) {
            msg.getRequestHeader().setHeader("Client-ID", 
              java.lang.System.getenv("CLIENT_ID"));
            msg.getRequestHeader().setHeader("Api-Key", 
              java.lang.System.getenv("API_KEY"));
        }
        
        function responseReceived(msg, initiator, helper) {
            // Do nothing
        }

  # Test a known endpoint
  - type: requestor
    description: "Test authentication headers"
    requests:
      - url: "${ZAP_TARGET}/health"
        method: "GET"
        responseCode: 200

  - type: exitStatus
    parameters:
      errorLevel: High
      warnLevel: Low
```

**Key points:**
- Uses `httpsender` script to inject headers on every request
- Reads credentials from environment variables
- Tests authentication with a simple health check
- Fails fast if authentication doesn't work

### 2. Main Scan Configuration

The full scan imports your OpenAPI spec and runs security tests:

```yaml
# automation.yaml
env:
  name: API Scan Environment
  contexts:
    - name: "TestContext"
      urls:
        - "${ZAP_TARGET}"
  parameters:
    failOnError: true
    failOnWarning: false
    progressToStdout: true

jobs:
  # Filter false positives
  - type: alertFilter
    parameters:
      deleteGlobalAlerts: false
    alertFilters:
      - ruleId: 40018  # SQL injection false positive
        newRisk: "False Positive"
        url: ".*/api/v2/.*"
        urlRegex: true

  # Import OpenAPI specification
  - type: openapi
    description: "Import OpenAPI and set target"
    parameters:
      apiFile: ${OPENAPI_FILE}
      targetUrl: "${ZAP_TARGET}"

  # Inject authentication headers
  - type: script
    parameters:
      action: add
      type: httpsender
      engine: "ECMAScript : Graal.js"
      name: AddHeader.js
      inline: |
        function sendingRequest(msg, initiator, helper) {
            msg.getRequestHeader().setHeader("Client-ID", 
              java.lang.System.getenv("CLIENT_ID"));
            msg.getRequestHeader().setHeader("Api-Key", 
              java.lang.System.getenv("API_KEY"));
        }
        
        function responseReceived(msg, initiator, helper) {}

  # Run active security scan
  - type: activeScan
    parameters:
      policy: "API-minimal"

  # Generate HTML report
  - type: report
    parameters:
      template: traditional-html
      reportTitle: "ZAP Scanning Report"
      reportDir: "${ZAP_WORK_DIR}/zap-report-html"

  # Set exit code based on findings
  - type: exitStatus
    parameters:
      errorLevel: High
      warnLevel: Low
```

**Configuration highlights:**
- **Alert filtering**: Suppress known false positives
- **OpenAPI import**: Automatically discover all endpoints
- **Policy-based scanning**: Use built-in policies like `API-minimal`
- **Traditional HTML reports**: Avoid exposing secrets in modern template
- **Exit status**: Fail pipeline on high-severity findings

## CI/CD Integration

### GitLab CI Example

```yaml
was:generate_openapi_schema:
  stage: test
  image: python:3.11
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  script:
    - python generate_schema.py
    - mkdir -p openapi-schema
    - cp api.yaml openapi-schema/api.yaml
  artifacts:
    paths:
      - openapi-schema/
    expire_in: "1 hour"

was:execute:
  stage: test
  needs:
    - job: was:generate_openapi_schema
  image:
    name: "zaproxy/zap-stable:20251021"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  variables:
    ZAP_TARGET: "https://api.example.com"
    ZAP_WORK_DIR: "${CI_PROJECT_DIR}"
    ZAP_AUTOMATION_FILE: "${CI_PROJECT_DIR}/.ci/zap/automation.yaml"
    ZAP_AUTOMATION_AUTH_FILE: "${CI_PROJECT_DIR}/.ci/zap/automation-auth-check.yaml"
    OPENAPI_FILE: "${CI_PROJECT_DIR}/openapi-schema/api.yaml"
  script:
    - |
      mkdir -p zap-report-html
      
      export CLIENT_ID=${SECRET_CLIENT}
      export API_KEY=${SECRET_KEY}
      
      echo "Running auth check..."
      zap.sh -cmd -autorun ${ZAP_AUTOMATION_AUTH_FILE}
      
      echo "Running full scan..."
      zap.sh -cmd -autorun ${ZAP_AUTOMATION_FILE} || exit_code=$?
      
      REPORT_URL="${CI_SERVER_PROTOCOL}://${CI_PROJECT_ROOT_NAMESPACE}.${CI_PAGES_DOMAIN}/-/${CI_PROJECT_PATH#"${CI_PROJECT_ROOT_NAMESPACE}/"}/-/jobs/$CI_JOB_ID/artifacts/zap-report-html/report.html"
      echo "Security Report: $REPORT_URL"
      
      if [ "$exit_code" = "1" ]; then
        echo "High-severity findings detected!"
        exit 1
      fi
  artifacts:
    paths:
      - zap-report-html/
    expire_in: "7 days"
```

### GitHub Actions Example

```yaml
name: Security Scan

on:
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday
  workflow_dispatch:

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate OpenAPI Schema
        run: |
          python generate_schema.py
          mkdir -p openapi-schema
          cp api.yaml openapi-schema/api.yaml
      
      - name: Run ZAP Authentication Check
        uses: docker://zaproxy/zap-stable:20251021
        env:
          CLIENT_ID: ${{ secrets.CLIENT_ID }}
          API_KEY: ${{ secrets.API_KEY }}
          ZAP_TARGET: https://api.example.com
        with:
          args: zap.sh -cmd -autorun .ci/zap/automation-auth-check.yaml
      
      - name: Run ZAP Security Scan
        uses: docker://zaproxy/zap-stable:20251021
        env:
          CLIENT_ID: ${{ secrets.CLIENT_ID }}
          API_KEY: ${{ secrets.API_KEY }}
          ZAP_TARGET: https://api.example.com
          ZAP_WORK_DIR: /github/workspace
          OPENAPI_FILE: /github/workspace/openapi-schema/api.yaml
        with:
          args: zap.sh -cmd -autorun .ci/zap/automation.yaml
      
      - name: Upload Security Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-report
          path: zap-report-html/
          retention-days: 7
```

## Local Testing

Test your configuration locally before committing:

```bash
#!/usr/bin/env bash
# localtest.sh

IMAGE="zaproxy/zap-stable:20251021"
ZAP_TARGET="https://api.example.com"
CLIENT_ID="${CLIENT_ID:-}"
API_KEY="${API_KEY:-}"

if [ -z "$CLIENT_ID" ] || [ -z "$API_KEY" ]; then
  echo "Error: Set CLIENT_ID and API_KEY environment variables"
  exit 1
fi

mkdir -p zap-report-html

docker run --rm -v "$(pwd)":/zap/wrk \
  -e OPENAPI_FILE="/zap/wrk/api.yaml" \
  -e CLIENT_ID="$CLIENT_ID" \
  -e API_KEY="$API_KEY" \
  -e ZAP_TARGET="$ZAP_TARGET" \
  -e ZAP_WORK_DIR="/zap/wrk" \
  "$IMAGE" \
  zap.sh -cmd -autorun /zap/wrk/automation.yaml

echo "Report available at: $(pwd)/zap-report-html/report.html"
```

Run it:

```bash
CLIENT_ID=your-client-id API_KEY=your-secret bash localtest.sh
```

## Best Practices

### 1. **Pin the ZAP Version**
```yaml
image: "zaproxy/zap-stable:20251021"
```
ZAP updates can change behavior. Pin to a specific version for consistency.

### 2. **Two-Stage Authentication**
First verify credentials work, then run the full scan. This provides faster feedback when auth fails.

### 3. **Use Traditional HTML Reports**
The modern template includes full request/response bodies, which can expose secrets in your injected headers. Use `traditional-html` instead.

### 4. **Filter False Positives**
Use `alertFilter` to suppress known false positives specific to your API:

```yaml
- type: alertFilter
  alertFilters:
    - ruleId: 40018
      newRisk: "False Positive"
      url: ".*/api/v2/.*"
      urlRegex: true
```

### 5. **Fail on High Severity Only**
Start with `errorLevel: High` to avoid breaking builds on minor findings:

```yaml
- type: exitStatus
  parameters:
    errorLevel: High
    warnLevel: Low
```

### 6. **Schedule Scans**
Don't run security scans on every commit. Use scheduled pipelines (nightly/weekly) to avoid slowing down development.

## Common Issues

### Authentication Not Working

**Problem:** All requests return 401/403

**Solution:** Verify your script is injecting headers correctly:
```javascript
function sendingRequest(msg, initiator, helper) {
    // Add debug logging
    print("CLIENT_ID: " + java.lang.System.getenv("CLIENT_ID"));
    msg.getRequestHeader().setHeader("Client-ID", 
      java.lang.System.getenv("CLIENT_ID"));
}
```

### Scan Takes Too Long

**Problem:** Active scan runs for hours

**Solution:** Use a minimal scan policy for APIs:
```yaml
- type: activeScan
  parameters:
    policy: "API-minimal"
```

### Reports Expose Secrets

**Problem:** Modern HTML report shows API keys in requests

**Solution:** Use traditional template:
```yaml
- type: report
  parameters:
    template: traditional-html  # Not "modern"
```

## Complete Example Repository

All the examples and configurations shown in this post are available in my GitHub repository:

**[blog-20251112-zaproxy-automation](https://github.com/BlackDark/blog-20251112-zaproxy-automation)**

The repository includes:
- Complete automation configurations
- GitLab CI pipeline example
- Local testing script
- Sample OpenAPI specification

## Conclusion

By using ZAP's Automation Framework directly, you get:

- ✅ **Native configuration** without wrapper dependencies
- ✅ **Version-controlled** security policies
- ✅ **Flexible authentication** via inline scripts
- ✅ **Consistent results** across environments
- ✅ **Easy local testing** for development

Start small with authentication checks and basic scans, then gradually expand your security testing coverage. Your future self (and security team) will thank you!

## Resources

- [ZAP Automation Framework Documentation](https://www.zaproxy.org/docs/desktop/addons/automation-framework/)
- [ZAP Community Scripts](https://github.com/zaproxy/community-scripts)
- [OpenAPI Import Documentation](https://www.zaproxy.org/docs/desktop/addons/openapi-support/)
- [Authentication Helper](https://www.zaproxy.org/docs/desktop/addons/authentication-helper/)

---

Have you integrated security scanning into your CI/CD pipelines? What challenges did you face? Share your experiences in the comments!
