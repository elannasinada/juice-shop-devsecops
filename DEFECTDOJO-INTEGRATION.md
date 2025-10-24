# Defect Dojo Integration Guide

## Overview
This guide explains how to integrate the security scan reports from your DevSecOps pipeline into Defect Dojo for centralized vulnerability management.

## Prerequisites
- Defect Dojo instance (cloud or self-hosted)
- Defect Dojo API token
- GitHub repository with completed SAST, SCA, DAST, and Container scans

## Step 1: Set up Defect Dojo

### Option A: Self-Hosted Installation
```bash
# Using Docker Compose
git clone https://github.com/DefectDojo/django-DefectDojo
cd django-DefectDojo
docker-compose up -d
```

Access Defect Dojo at `http://localhost:8080`
- Default username: `admin`
- Default password: `admin` (change immediately)

### Option B: Cloud/Hosted Solution
Sign up for a Defect Dojo cloud instance or use your organization's instance.

## Step 2: Create a Product in Defect Dojo

1. Login to Defect Dojo
2. Navigate to **Products** → **Add Product**
3. Fill in product details:
   - **Name**: Juice Shop DevSecOps
   - **Description**: OWASP Juice Shop with DevSecOps Pipeline
   - **Product Type**: Web Application
   - **Business Criticality**: High

## Step 3: Generate API Token

1. Go to **User Settings** → **API Key**
2. Click **Generate API Key**
3. Copy the token and save it securely

## Step 4: Configure GitHub Secrets

Add the following secrets to your GitHub repository:
- Go to **Settings** → **Secrets and variables** → **Actions**
- Add new repository secrets:
  - `DEFECTDOJO_URL`: Your Defect Dojo URL (e.g., `https://your-defectdojo.com`)
  - `DEFECTDOJO_TOKEN`: Your API token
  - `DEFECTDOJO_PRODUCT_ID`: The product ID from Defect Dojo

## Step 5: Import Security Scan Reports

### Manual Import (Current Approach)

1. Download artifacts from GitHub Actions workflow runs:
   - CodeQL SARIF results
   - Snyk JSON report
   - OWASP ZAP HTML/JSON reports
   - Trivy JSON/SARIF results

2. In Defect Dojo, navigate to **Engagements** → **Add Engagement**:
   - **Product**: Juice Shop DevSecOps
   - **Name**: Sprint/Date-based name (e.g., "2025-01-24 Security Scan")
   - **Engagement Type**: CI/CD
   - **Status**: In Progress

3. Import each scan result:
   - Go to the engagement → **Import Scan Results**
   - Select scan type:
     - **CodeQL**: SARIF format
     - **Snyk**: Snyk Scan (JSON)
     - **OWASP ZAP**: ZAP Scan
     - **Trivy**: Trivy Scan (JSON/SARIF)
   - Upload the report file
   - Click **Import**

### Automated Import (Recommended - GitHub Actions)

Create a new workflow file `.github/workflows/defectdojo-import.yml`:

```yaml
name: "Import to Defect Dojo"

on:
  workflow_run:
    workflows: ["CodeQL SAST Analysis", "SCA with Snyk", "DAST with OWASP ZAP", "Docker Build and Container Security Scan"]
    types:
      - completed

jobs:
  import-to-defectdojo:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    steps:
    - name: Download CodeQL SARIF
      uses: actions/download-artifact@v4
      with:
        name: codeql-sarif-results
        path: reports/codeql/
        run-id: ${{ github.event.workflow_run.id }}
        github-token: ${{ secrets.GITHUB_TOKEN }}

    - name: Download Snyk Report
      uses: actions/download-artifact@v4
      with:
        name: snyk-security-report
        path: reports/snyk/
        run-id: ${{ github.event.workflow_run.id }}
        github-token: ${{ secrets.GITHUB_TOKEN }}

    - name: Download ZAP Reports
      uses: actions/download-artifact@v4
      with:
        name: zap-dast-reports
        path: reports/zap/
        run-id: ${{ github.event.workflow_run.id }}
        github-token: ${{ secrets.GITHUB_TOKEN }}

    - name: Download Trivy Reports
      uses: actions/download-artifact@v4
      with:
        name: trivy-security-reports
        path: reports/trivy/
        run-id: ${{ github.event.workflow_run.id }}
        github-token: ${{ secrets.GITHUB_TOKEN }}

    - name: Import CodeQL to Defect Dojo
      run: |
        curl -X POST "${{ secrets.DEFECTDOJO_URL }}/api/v2/import-scan/" \
          -H "Authorization: Token ${{ secrets.DEFECTDOJO_TOKEN }}" \
          -F "scan_type=SARIF" \
          -F "product_name=Juice Shop DevSecOps" \
          -F "engagement_name=CI/CD $(date +%Y-%m-%d)" \
          -F "file=@reports/codeql/*.sarif" \
          -F "active=true" \
          -F "verified=false"

    - name: Import Snyk to Defect Dojo
      run: |
        curl -X POST "${{ secrets.DEFECTDOJO_URL }}/api/v2/import-scan/" \
          -H "Authorization: Token ${{ secrets.DEFECTDOJO_TOKEN }}" \
          -F "scan_type=Snyk Scan" \
          -F "product_name=Juice Shop DevSecOps" \
          -F "engagement_name=CI/CD $(date +%Y-%m-%d)" \
          -F "file=@reports/snyk/snyk-report.json" \
          -F "active=true" \
          -F "verified=false"

    - name: Import ZAP to Defect Dojo
      run: |
        curl -X POST "${{ secrets.DEFECTDOJO_URL }}/api/v2/import-scan/" \
          -H "Authorization: Token ${{ secrets.DEFECTDOJO_TOKEN }}" \
          -F "scan_type=ZAP Scan" \
          -F "product_name=Juice Shop DevSecOps" \
          -F "engagement_name=CI/CD $(date +%Y-%m-%d)" \
          -F "file=@reports/zap/zap-report.json" \
          -F "active=true" \
          -F "verified=false"

    - name: Import Trivy to Defect Dojo
      run: |
        curl -X POST "${{ secrets.DEFECTDOJO_URL }}/api/v2/import-scan/" \
          -H "Authorization: Token ${{ secrets.DEFECTDOJO_TOKEN }}" \
          -F "scan_type=Trivy Scan" \
          -F "product_name=Juice Shop DevSecOps" \
          -F "engagement_name=CI/CD $(date +%Y-%m-%d)" \
          -F "file=@reports/trivy/trivy-results.json" \
          -F "active=true" \
          -F "verified=false"
```

## Step 6: Analyze and Prioritize Vulnerabilities

1. In Defect Dojo, go to **Findings** → **All Findings**
2. Filter by:
   - **Product**: Juice Shop DevSecOps
   - **Severity**: Critical, High
   - **Active**: True

3. For each vulnerability:
   - Review the description and affected components
   - Assign to team members
   - Set risk acceptance or remediation plan
   - Track remediation progress

## Step 7: Generate Reports

Defect Dojo provides various reports:
- **Executive Summary**: High-level overview for management
- **Detailed Report**: Technical details for developers
- **Metrics Dashboard**: Trends over time

Navigate to **Reports** → **Generate Report** and select your product.

## Supported Scan Types in Defect Dojo

| Tool | Defect Dojo Scan Type | Format |
|------|----------------------|---------|
| CodeQL | SARIF | .sarif |
| Snyk | Snyk Scan | .json |
| OWASP ZAP | ZAP Scan | .json, .xml |
| Trivy | Trivy Scan | .json, .sarif |

## Best Practices

1. **Regular Imports**: Import scan results after every pipeline run
2. **Engagement Tracking**: Create new engagements for each sprint/release
3. **Risk Acceptance**: Document accepted risks with justification
4. **SLA Tracking**: Set SLAs for critical and high vulnerabilities
5. **Remediation Verification**: Re-scan after fixes to verify remediation

## Troubleshooting

### Import Fails
- Verify API token is valid
- Check file format matches scan type
- Ensure product exists in Defect Dojo

### Duplicate Findings
- Enable deduplication in Defect Dojo settings
- Use consistent engagement names

### Missing Findings
- Verify scan completed successfully
- Check artifact download in GitHub Actions
- Review Defect Dojo logs for import errors

## Additional Resources

- [Defect Dojo Documentation](https://documentation.defectdojo.com/)
- [API Documentation](https://demo.defectdojo.org/api/v2/doc/)
- [GitHub Actions Integration](https://github.com/DefectDojo/django-DefectDojo/blob/master/readme-docs/INTEGRATION.md)
