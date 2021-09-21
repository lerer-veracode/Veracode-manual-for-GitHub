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
The basic script for Agent-Based SCA CLI (Command line interface) is documented [at the Veracode help center](https://help.veracode.com/r/Using_the_Veracode_SCA_Command_Line_Agent)

The basic form on a PC using cURL is usually the simplest way to be use in a CI workflow.

`curl -sSL https://download.sourceclear.com/install | sh`

As all of our integrations, more advanced scan options are documented across our [help center](https://help.veracode.com/r/c_sc_agent_usage)

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

With our different Static scanning, we now have the ability to import the findings using different techniques

### Visualize Pipeline Findings as Pull Request comment
The basic form of 'import' is not really an import but more of a surfacing the result. When the Pipeline Scan run within a workflow, all messages are kept inside the workflow. However, when we are using the scan as part of a Pull Request, we may want to copy the Scan output as a comment in the Pull Request main `Conversation` tab.

to do that we can utilize build-in GitHub scripting functionality. (This is not a __Veracode__ function)

See __[basic example](https://github.com/Lerer/veracode-pipeline-PR-comment)__

<details>
<summary>Another inline example</summary>
<p>

```yaml
name: Veracode SAST Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  static_pipeline_scan:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    
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
        run: java -Dpipeline.debug=true -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "result.zip" -jo true -so true
      - id: get-comment-body
        if: ${{ github.head_ref != '' }}
        run: |
          body=$(cat results.txt)
          body="${body//$'====================\n--------------------------'/'====================<details>'}"
          body="${body//$'--------------------------\nF'/'</details><details><summary>F'}"
          body="${body//$'.\n--------------------------'/'.</summary>'}"                             
          body="${body//$'==========================\nFA'/'</details>==========================<br>FA'}"
          body="${body//$'\n'/'<br>'}"
          echo ::set-output name=body::$body
      - name: Add comment to PR
        if: ${{ github.head_ref != '' }}
        uses: peter-evans/create-or-update-comment@v1
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: ${{ steps.get-comment-body.outputs.body }}
          reactions: rocket
```
</p>
</details>
</br>
 
> :bulb: A different output can be done by further/differently manipulating the text output

### Pipeline Scan as GitHub Security issues
For customers with Enterprise Accounts with license for 'GitHub Advanced Security', we can import our Pipeline Scan result to the dedicated `Security` issues.

That option is available via our official __[action](https://github.com/marketplace/actions/veracode-static-analysis-pipeline-scan-and-sarif-import)__


<details>
<summary>See example</summary>
<p>

```yaml
name: Pipeline Scan with creation of GitHub Security issues

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
  pull_request:
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # The workflow consist of a single job to quickly scan dependencies
  Pipeline_to_Security_Issues:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
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
      - name: Convert pipeline scan output to SARIF format
        id: convert Pipeline results to SARIF format 
        uses: Veracode/veracode-pipeline-scan-results-to-sarif@v0.1.2
        with:
          pipeline-results-json: results.json
          output-results-sarif: veracode-results.sarif
          source-base-path-1: "^com/veracode:src/main/java/com/veracode"
          source-base-path-2: "^WEB-INF:src/main/webapp/WEB-INF"
          finding-rule-level: "3:1:0"

      - name: upload SARIF file to repository
        uses: github/codeql-action/upload-sarif@v1
        with: # Path to SARIF file relative to the root of the repository
          sarif_file: veracode-results.sarif
```
</p>      
</details> 


### Upload and Scan / Pipeline Scan as Issues
Lastly, for organization who don't have the license for 'GitHub Advanced Security' or simply don't want to use the `Security` issues, they can import the finding as any other issue in the `Issues` section. 

That is achieve by an __[Action](https://github.com/marketplace/actions/veracode-scan-results-to-github-issues)__.

<details>
<summary>Screenshots</summary>
<p>
<p align="center">
  <img src="/media/img/issues-action-pipeline.png" width="600px" alt="Pipeline scan output as GitHub Issue"/>
</p>
<br/>
<p align="center">
  <img src="/media/img/issues-action-policy.png" width="600px" alt="Policy scan output as GitHub Issue"/>
</p>
</p>
</details>

## Flow Control

This section is mostly around utilizing GitHub build-in functionality to get a desire process 

### Branch protection (for Pull Request)
__[Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches)__ can help answer a (repeated) question - How can we block a Pull Request for failed scan.

The answer is in a __['Require Status Checks Before Merging' section](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches#require-status-checks-before-merging)__ in the above manual.

For those with the right GitHub permissions, here are the __[Step-by-Step instructions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule)__ for setting up the Branch Protection Rules

If we enable branch protection, any failed `job` within a workflow can prevent pull request from being approved  :arrow_right: We then need to make sure a failed Scan is also failing the job in the workflow:exclamation:

### Splitting into Jobs - Adding Checks
- Separate to Jobs or completely different workflows

### Workflow, Jobs and Steps condition and triggers
  - Make sure a step/job is running only if previous step/job succeed    

  - You can define very specific triggers to run partial scan for specific things - see example for SCA on JS      

## Scaling in an Organization

### Share templated workflows
Instead of rewriting everything for every BU/Team/repo use shared workflow by creating `.github` repository in the account

- See [Shared workflows](https://docs.github.com/en/actions/learn-github-actions/sharing-workflows-with-your-organization) documentation

In addition, you can think on shared Secret - but keep in mind Pipeline Scan throughput to not exceed 6 scans/min

### Auto Pull request for vulnerable dependencies after merge into Main/Master
Right after a pull request is approved, the target branch will issue a `push` event (as new code is merged). This is an opportunity to run a scan to auto create Pull request based on the Vulnerable Methods found in a scan.

__[Veracode SCA Auto pull request](https://help.veracode.com/r/t_configure_auto_pr)__ only apply to [supported languages](https://help.veracode.com/r/Understanding_Automatic_Pull_Request_Support): Java, Python, Ruby, JavaScript, Objective-C, and PHP 

Make sure to read the instructions as it involves [creation of Github token](https://help.veracode.com/r/t_configure_pr_github) 

<details>
<summary>See example</summary>
<p>

```yaml
name: Auto Pull Request on SCA findings

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
    branches: [ master,main ]
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # The workflow consist of a single job to quickly scan dependencies
  SCA_Scan for PR:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]

    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Check out repo 
        uses: actions/checkout@v2

      # run quick scan on the project
      - name: SCA Scan
        env: 
          SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_API_TOKEN }}
          SRCCLR_SCM_TOKEN: ${{ secrets.SRCCLR_GITHUB_TOKEN }}
          SRCCLR_SCM_TYPE: GITHUB
          # PR on: methods
          SRCCLR_PR_ON: low
          SRCCLR_NO_BREAKING_UPDATES: true
          SRCCLR_IGNORE_CLOSED_PRS: true
          EXTRA_ARGS: '--update-advisor --pull-request --unmatched'
        run: |
          git config --global user.email "${{ secrets.USER_EMAIL }}"
          git config --global user.name "${{ secrets.USER_NAME }}"
          curl -sSL https://download.sourceclear.com/ci.sh | sh -s -- scan $EXTRA_ARGS
```

<br/>
<p align="center">
  <img src="/media/img/SCA-auto-PR.png" width="700px" alt="Policy scan output as GitHub Issue"/>
</p>

</p>      
</details> 
<br/>

### Options for Pipeline scan baseline
Pipeline scan provides the ability to use baseline acting as the "approved mitigations" or accepted risk condition which instruct the Pipeline Scan to only highlight finding other than the ones in the baseline.

#### Commit Baseline into a Repository

#### Save Baseline as an Artifact

### Shared Downloaded Policies for Pipeline Scan