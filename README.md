# GitHub Advanced Security Configuration as Code

![GitHub](https://img.shields.io/badge/GitHub-181717?logo=github&logoColor=white)
![Postman](https://img.shields.io/badge/Postman-FF6C37?logo=postman&logoColor=white)
![Configuration as Code](https://img.shields.io/badge/Configuration%20as%20Code-007ACC?logo=azuredevops&logoColor=white)

## Overview

This repository demonstrates how to leverage Newman and the [GitHub API](https://docs.github.com/en/rest?apiVersion=2022-11-28) to automate the configuration and testing of [GitHub Advanced Security](https://docs.github.com/en/enterprise-cloud@latest/get-started/learning-about-github/about-github-advanced-security) features, including Dependabot, CodeQL, and Secret Scanning. By treating security configurations as code, you can improve consistency, increase efficiency, and enhance collaboration. This repository is intended for security engineers, DevOps engineers, and developers who want to streamline their GitHub Advanced Security configurations. Explore the repository, contribute, and adapt it to your specific needs.

## Benefits

- **Consistency**: Ensure that security settings are applied uniformly across all repositories.
- **Automation**: Integrate with CI/CD pipelines to automate the application and verification of security configurations.
- **Version Control**: Track changes to security configurations over time using Git.
- **Collaboration**: Use pull requests and code reviews to manage changes to security settings.
- **Testability**: Validate configurations with automated tests to ensure they are applied correctly.


## How to Use locally

1. Clone this repository
2. cd into the repository
3. [Install newman](https://learning.postman.com/docs/collections/using-newman-cli/installing-running-newman)
4. Populate the `environment` file with the apropriate values. See examples in the `environments` directory.
5. Populate the repo_data.json file with the appropriate values. See examples in the `environments` directory as well. This file contains all the repos you want to configure.
For example, use the GitHub REST API to list all repositories in an organization and save the output to a file.

    <details>
    <summary>Example shell script to fetch repos</summary>

    ```bash
    # fetch_repos.sh
    #!/bin/bash
    PROD_CONFIG_PATH="environments/prod"
    # Fetch repos and create iteration data file
    curl -L \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $SECURITY_CONFIG_TOKEN" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    "https://api.github.com/orgs/tsvi-solutions/repos?per_page=100" | \
    jq '[.[] | select(.archived == false) | {repo_name: .name}]' > ${PROD_CONFIG_PATH}/repo_data.json
    ```

    </details>

6. Run newman

    <details>
    <summary>Example shell script to run newman</summary>

    ```bash
    #!/bin/bash

    PROD_ENV_CONFIG_PATH="environments/prod"
    PROD_CONFIG_PATH="configs/combined-features"
    # Fetch repos and create iteration data file
    ./fetch_repos.sh

    # Create timestamped results directory
    timestamp=$(date +"%Y%m%d_%H%M%S")
    results_dir="results_$timestamp"
    mkdir -p $results_dir

    # Check for rollback flag
    if [ "$1" == "-r" ]; then
    collection="${PROD_CONFIG_PATH}/security-settings_disable.json"
    else
    collection="${PROD_CONFIG_PATH}/security-settings.json"
    fi

    # Run newman with the selected collection
    newman run $collection \
    -e ${PROD_ENV_CONFIG_PATH}/tsvi-solutions-env.json \
    --env-var "GITHUB_TOKEN=$GITHUB_TOKEN" \
    -r cli,htmlextra,json \
    --reporter-htmlextra-export $results_dir/report.html \
    --reporter-json-export $results_dir/report.json \
    --iteration-count $(jq length ${PROD_ENV_CONFIG_PATH}/repo_data.json) \
    -d ${PROD_ENV_CONFIG_PATH}/repo_data.json

    # Output results directory
    echo "Results saved in $results_dir"
    ```

    </details>


In this example, the script above will run the configuration in path `configs/combined-features/security-settings.json` will be applied to all the repos in the `environments/prod/repo_data.json` file. The results will be saved in a timestamped directory in the root of the repository.

## Example Output

When running the Newman collection, you'll see detailed output showing the configuration of security settings across your repositories:

```console
GitHub Security Settings

Iteration 1/2

❏ 1. Secret Scanning Settings
↳ Enable Secret Scanning
  ┌
  │ 'Configuring Secret Scanning for: killercats_website'
  └
  PATCH https://api.github.com/repos/tsvi-solutions/killercats_website [200 OK, 19.6kB, 505ms]
  ✓  Status code is 200
  ✓  Response time is less than 1000ms
  ✓  Secret scanning is enabled

┌─────────────────────────┬─────────────────────┬────────────────────┐
│                         │            executed │             failed │
├─────────────────────────┼─────────────────────┼────────────────────┤
│              iterations │                   2 │                  0 │
│                requests │                  12 │                  0 │
│            test-scripts │                   8 │                  0 │
│      prerequest-scripts │                  12 │                  0 │
│              assertions │                  26 │                  0 │
├─────────────────────────┴─────────────────────┴────────────────────┤
│ total run duration: 3.5s                                           │
│ total data received: 74.09kB (approx)                             │
│ average response time: 276ms [min: 143ms, max: 505ms, s.d.: 106ms]│
└────────────────────────────────────────────────────────────────────┘
```

## How to use in GitHub Actions
1. There is already a workflow that runs on a schedule in `.github/workflows/run_sec_settings.yml`. 
> **_NOTE_** You can modify the schedule to run at your desired frequency.

2. Workflow will run on changes/pull requests against the main branch. Ideally, the workflow will be run new configuration changes on PR, conduct a code review, confirm the changes are working as expected and applied to the repositories accodingly, and then merge the changes to main.

3. The action `matt-ball/newman-action` is used to run newman and excute the configuration changes.

## Repository Structure

This repository is organized into the following directories and files:

### GitHub Actions Workflows (`.github/workflows/`)

This directory contains GitHub Actions workflows for automating various tasks.  Specifically, it includes:

*   `run_sec_settings.yml`: Workflow for running security settings using Newman.

### Configuration Files (`configs/`)

This directory houses configuration files for enabling and disabling various GitHub Advanced Security features.  It's further organized into subdirectories for each feature (please see the [GitHub REST API documentation](https://docs.github.com/en/rest?apiVersion=2022-11-28) for more information on the available configuration options):

*   `bulk-enable/`: Configurations for enabling and resetting all features.
*   `code-scanning/`: Configurations specific to Code Scanning.
*   `secret-scanning/`: Configurations specific to Secret Scanning.
*   `dependabot-sca/`: Configurations specific to Dependabot.
*   `push-protection-test.json`: Configuration for testing push protection.

### Newman Environment Files (`environments/`)

This directory contains environment files used by Newman:

*   `environment.json`: Environment variables for Newman collections.
*   `repo_data.json`: Repository data used for iterations.

### Results Storage (`results/`)

This directory is dedicated to storing output results generated by the scripts.
