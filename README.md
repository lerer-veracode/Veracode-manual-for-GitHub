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

- Import Pipeline Findings as Pull Request message
  - see [basic example](https://github.com/Lerer/veracode-pipeline-PR-comment). A more advance example can be done by further manipulating the text output

## Flaw Control

Mainly used as GitHub build-in charectaristic

- Branch protection
  - See [Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) in GitHub for more advance features
- Workflaw Checks
  - Seperate to Jobs or completely different workflaws  
- Jobs and Steps condition
  - Make sure a step/job is running only if previous step/job succeed
- Trigger
  - You can define very specific triggers to run partial scan for specific things - see example for SCA on JS      

## Scaling in an Organization

Instead of rewriting everything for every BU/Team/repo use shared workflow by creating `.github` repository in the account

- See [Shared workflows](https://docs.github.com/en/actions/learn-github-actions/sharing-workflows-with-your-organization) documentation

In addition, you can think on shared Secret - but keep in mind Pipeline Scan throughput to not exceed 6 scans/min


