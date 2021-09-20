# Veracode manual for GitHub

## Introduction

This manual incorporate few option Veracode customers can use to integrate Veracode Scanning solutions with their own workflow processes. 

From the previous sentence we can see that all our discussed concepts and work will be around and within GitHub workflow and unavoidably GitHub Actions.

## The basic

The base of our integration is achievable via script running within the workflows. These script will look very similar to what you would run locally on your PC to do any of the listed scan types. 



### Upload and Scan 
#### Script
A basic script using a wrapper is documented __[at Veracode help center](https://help.veracode.com/r/r_uploadandscan)__ and looks as follow:
`java -jar vosp-api-wrapper-java<version>.jar -action uploadandscan -vid <Veracode API ID> -vkey <Veracode API key> -appname myapp -createprofile true -teams myteam -criticality VeryHigh -sandboxname sandboxA -createsandbox true -version <unique version> -scantimeout 30 -selectedpreviously true -filepath /workspace/myapp.jar`

within a GitHub workflow we will need to download the latest version and run the above script.
<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download Veracode Wrapper
        working-directory: ./dls/
        run: |
          javawrapperversion=$(curl https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/maven-metadata.xml | grep latest |  cut -d '>' -f 2 | cut -d '<' -f 1)
          echo "javawrapperversion: $javawrapperversion"
          curl -sS -o VeracodeJavaAPI.jar "https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/$javawrapperversion/vosp-api-wrappers-java-$javawrapperversion.jar"
      - name: Run Upload and Scan
        continue-on-error: true
        run: |
          java -jar VeracodeJavaAPI.jar -action uploadandscan -vid ${{secrets.VERACODE_ID}} -vkey ${{secrets.VERACODE_KEY}} -appname ${{ github.repository }} -createprofile true -teams teamA -criticality VeryHigh -sandboxname sandboxA -createsandbox true -version ${{ github.run_id }} -scantimeout 30 -selectedpreviously true -filepath /workspace/app.zip
```
</p>
</details>


#### GitHub Action
An easier way to incorporate the upload and scan into a workflow is using Veracode supported __[upload-and-scan GitHub action](https://github.com/marketplace/actions/veracode-upload-and-scan)__

To authoring a workflow with the official action we will use the documentation in the action page
<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: Veracode Upload And Scan
        uses: veracode/veracode-uploadandscan-action@0.2.1
        with:
          # appname
          appname: ${{ github.repository }}
          # createprofile
          createprofile: true
          # filepath
          filepath: result.zip #Path to dlls/src/jars/wars....
          # version
          version: ${{ github.run_id }}
          # vid
          vid: ${{ secrets.VERACODE_ID }}
          # vkey
          vkey: ${{ secrets.VERACODE_KEY }}
          # true or false
          createsandbox: true
          # name of the sandbox
          sandboxname: sandboxA
          # wait X minutes for the scan to complete
          scantimeout: 0
          # business criticality - policy selection
          criticality: "High"
```
</p>
</details>

#### Align sandbox name with branch name (Optional)
As you noticed in the above examples, the sandbox name for the scan was a fixed name. A fixed sandbox name is not maintainable as we probably want to scan different changes/versions in a different sandboxes.

An alternative to that is to have the sandbox name as the Git branch name. In order to achieve this we can modify the workflow definition to include a step prior to the scan to save the branch name as an attribute and use it when we submit for scan.

<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  generate-sandbox-name:
    runs-on: [ self-hosted, generic ]
    outputs:
      sandbox-name: ${{ steps.set-sandbox-name.outputs.sandbox-name }}
    steps:
      # Creates the sandbox(logical release descriptive status of current branch)
      - id: set-sandbox-name
        name: set-sandbox-name
        run: |
          echo ${{ github.head_ref }}
          branchName="${{ github.head_ref }}"
          if [[ -z "$branchName" ]]; then
            branchName="${{ github.ref }}"
          fi
          
          if [[ $branchName == *"master"* ]]; then
          echo "::set-output name=sandbox-name::Master"
          elif [[ $branchName == *"main"* ]]; then
          echo "::set-output name=sandbox-name::Main"
          elif [[ $branchName == *"elease/"* ]]; then
          echo "::set-output name=sandbox-name::$branchName"
          else
          echo "::set-output name=sandbox-name::Development"
          fi        
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: Veracode Upload And Scan
        uses: veracode/veracode-uploadandscan-action@0.2.1
        with:
          # appname
          appname: ${{ github.repository }}
          # createprofile
          createprofile: true
          # filepath
          filepath: result.zip #Path to dlls/src/jars/wars....
          # version
          version: ${{ github.run_id }}
          # vid
          vid: ${{ secrets.VERACODE_ID }}
          # vkey
          vkey: ${{ secrets.VERACODE_KEY }}
          # true or false
          createsandbox: true
          # name of the sandbox
          sandboxname: "${{needs.generate-sandbox-name.outputs.sandbox-name}}"
          # wait X minutes for the scan to complete
          scantimeout: 0
          # business criticality - policy selection
          criticality: "High"
```
</p>
</details>
<br/>

> :bulb: - The above example uses multiple jobs definition which I will further elaborate at the [Flow Control section](#flow-control)

### Pipeline Scan
#### Script
The basic script for Pipeline documented __[at the Veracode help center](https://help.veracode.com/r/Run_a_Pipeline_Scan_from_the_Command_Line)__ and in its most basic form it looks as follow.

`java -jar pipeline-scan.jar --file <file.zip>`

A more advanced and context aware options are documented __[here](https://help.veracode.com/r/r_pipeline_scan_commands)__. 

within a Github workflow the same scan script will be put as a step right after downloading the Pipeline Scan code itself
<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Pipeline Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Run Pipeline Scanner
        continue-on-error: true
        run: java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "result.zip" -jo true -so true  
```
</p>      
</details> 
<br/>

> :point_right: - it is recommended to include the application name and if possible the sandbox or branch name in the pipeline scan attribute to help with the platform reports. [-p \<project name/repository name\>] [-r \<project ref/branch name\>]

### Agent-Based SCA
#### Script
The basic script for Agent-Based SCA is documented at the Veracode help center and in its most basic form on a PC it looks as follow.

~~java -jar pipeline-scan.jar --file <file.zip>~~

~~A more advanced and context aware options are documented here.~~

For Agent-Based SCA (security scanning of 3<sup>rd</sup> party components) solution we have a very simple script which can easily put in a workflow.



<details>
<summary>See example</summary>
<p>

```yaml
name: SCA on change in dependencies definition

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
    paths:
      - 'package-lock.json'
  pull_request:
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # The workflow consist of a single job to quickly scan dependencies
  SCA_Scan:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Check out repo 
        uses: actions/checkout@v2

      # run quick scan on the project
      - name: SCA Scan
        env: 
          SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_API_TOKEN }}
        run: curl -sSL https://download.sourceclear.com/ci.sh | sh -s scan --quick
```
</p>      
</details> 
<br/>

> :bulb: The above example script used a very specific trigger for changes in TypeScript/JavaScript packaging file __`package-lock.json`__. Different programming language may result in different trigger __`on`__ scan attributes

## Import Findings

Import the findings using different techniques

- Upload and Scan / Pipeline Scan as Issues
  - [Action](https://github.com/marketplace/actions/veracode-scan-results-to-github-issues)

- Pipeline Scan as GitHub Security issues
  - [action](https://github.com/marketplace/actions/veracode-static-analysis-pipeline-scan-and-sarif-import)

- Import Pipeline Findings as Pull Request message
  - see [basic example](https://github.com/Lerer/veracode-pipeline-PR-comment). A more advance example can be done by further manipulating the text output

## Flow Control

Mainly used as GitHub build-in characteristic

- Branch protection (for Pull Request)
  - See [Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) in GitHub for more advance features
- Workflow Checks
  - Separate to Jobs or completely different workflows  
- Jobs and Steps condition
  - Make sure a step/job is running only if previous step/job succeed
- Trigger
  - You can define very specific triggers to run partial scan for specific things - see example for SCA on JS      

## Scaling in an Organization

### Share templated workflows
Instead of rewriting everything for every BU/Team/repo use shared workflow by creating `.github` repository in the account

- See [Shared workflows](https://docs.github.com/en/actions/learn-github-actions/sharing-workflows-with-your-organization) documentation

In addition, you can think on shared Secret - but keep in mind Pipeline Scan throughput to not exceed 6 scans/min

### Options for Pipeline scan baseline
Pipeline scan provides the ability to use baseline acting as the "approved mitigations" or accepted risk condition which instruct the Pipeline Scan to only highlight finding other than the ones in the baseline.
