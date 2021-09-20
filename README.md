# Veracode manual for GitHub

## The basic

the basic steps of integration

- Upload and Scan 
  - Script
  - [Action](https://github.com/marketplace/actions/veracode-upload-and-scan)
- Pipeline Scan
  - Script
- Agent-Based SCA
  - Script

## Import Findings

Import the findings using different techniques

- Upload and Scan / Pipeline Scan as Issues
  - [Action](https://github.com/marketplace/actions/veracode-scan-results-to-github-issues)

- Pipeline Scan as GitHub Security issues
  - [action](https://github.com/marketplace/actions/veracode-static-analysis-pipeline-scan-and-sarif-import)

## Flaw Control

Mainly used as GitHub build-in charectaristic

- Branch protection
  - See [Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) in GitHub for more advance features
- Workflaw Checks
  - Seperate to Jobs or completely different workflaws  

