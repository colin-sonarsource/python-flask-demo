# Storyboardssss

The goal of this demo is to show the analysis of a Python application in SonarCloud.
We want to showcase how to apply the "Clean As You Code" methodology in practice.

We start with a Flask application that represents a legacy project which we want to analyze.
This Flask application contains a "main" branch with the existing code. 
It also contains an "add-feature" branch that represents a new feature we want to develop for the application.

The full set-up can either be done as part of the demo (takes about 15 minutes), or beforehand.
A branch "enable-ci-analysis" is available to move from Automatic Analysis to a CI-based analysis, with import of code coverage information.

When fully set-up, the concept of PR Quality Gate on new code can be shown as well as its independence from the main code issues.
The application features basic, yet varied, issue types that can be detected by SonarCloud. In the PR, we have:
* A simple bug with no secondary location (raising a non-exception object)
* A bug with a secondary location on another file (calling a function with the wrong number of arguments)
* A classic taint analysis vulnerability (SQL injection)
* A reflected XSS (also taint analysis)
* A "bad practice" code smell (a bare except clause)
* A code smell that is actually a bug (inconsistency between type hint and usages) - SonarCloud tends to be conservative when raising issues
* A stylistic code smell (nested if statements that could be simplified) - good candidate to illustrate custom quality profiles (disabling the rule)

Additionally, we have security hotspots on the main branch:
* A disabled by default CSRF protection on the flask application
* A slow regular expression, vulnerable to catastrophic backtracking

When setting up CI-based analysis, import of code coverage will be done by default (in the enable-ci-analysis branch).
Flake8 is also running in the CI by default, its issues can be imported as well (we also support common linters like pylint, bandit or mypy).

If you want to demo SonarLint, you can also clone this project to show the issues in SonarLint. The injection vulnerabilities will not be displayed there. Some of the issues have quick fixes for them.
Connected mode can also be shown by simply following the tutorial in the IDE, which allow to synchronize silenced issues/custom quality profiles/etc...

## Running the webapp

Python 3 and flask need to be installed in the environment. You can run the following command to install the required dependencies:

```pip install -r requirements.txt```

- Initialize the database with `python init_db.py` (optional: a `database.db` file is already committed in the repository)
- `cd pokedex` and then simply run the webapp with `flask run`

Running the web application is entirely optional for the demo, it can be used to make the application more visual and to show some of the bugs/vulnerabilities in practice.
# Setup instructions

We're going to set up a SonarCloud analysis on this project. We'll visualise issues on the main branch and on pull requests and see how PRs get decorated automatically.

We'll then set up a CI-based analysis and import code coverage information into the SonarCloud UI.

Useful link: https://docs.sonarcloud.io/

## Getting started

- Fork this repository, with all existing branches (by default, only the main branch is forked).
- A basic workflow which will act as our CI already exists in `.github/workflows/python-app.yml`. It is disabled by default. Go to `Actions` and enable GitHub Actions to activate it.
- Go to `Pull requests->New pull request` and open a pull request from the `add-feature` branch to the `main` branch of your fork. Be careful that, by default, the PR targets the upstream repository.
- The GitHub Action should run and succeed.


## First analysis on SonarCloud

We'll see how to enable SonarCloud analysis without making any changes to our CI pipeline.

- Go to https://sonarcloud.io/sessions/new and sign up using your GitHub account.
- Create a new organization under your name and give SonarCloud permission to see the forked repository. 
- Go to `Analyze new project` and select the forked repository.

The first analysis should execute on the main branch first, then on the pull request. 
The pull request should be decorated with the analysis result.

## Adding code coverage to the analysis result

By default, source code is analyzed automatically by SonarCloud. 
As it is a static analysis tool, it does not execute tests and is not able to compute code coverage by itself.
You'll need to generate code coverage information and run the analysis in your CI to be able to import it.

**Note:** for simplicity, the branch `enable-ci-analysis` is already created in this repository with the required changes. From this branch, you only need to:
* Define a `SONAR_TOKEN` secret in your GitHub repository with a token created in SonarCloud (see [here](#enable-ci-based-analysis)).
* Replace the placeholders in the `sonar-project.properties` file with your project information.
* Merge the `enable-ci-analysis` in your main branch, then rebase the feature branch.

If you're using the `enable-ci-analysis` branch, you can skip the rest of this section.

### Generate coverage information
To generate coverage information, the `.github/workflow/python-app.yml` file should be updated. We'll also need to make sure file paths are set to be relative to avoid any issue when importing the report.

- Clone the repository and open it in your favorite IDE.
- At the root of the repository, create a `.coveragerc` file containing the following:
```
[run]
source = pokedex
branch = True
relative_files = True
```
- In the `.github/workflow/python-app.yml`, replace the `pytest` command with:
 
```pytest --cov --cov-report xml:cov.xml --cov-config=.coveragerc```


### Enable CI-based analysis
We'll then enable CI-based analysis using the [SonarCloud GitHub Action](https://github.com/marketplace/actions/sonarcloud-scan):

- Go to the overview of your project in SonarCloud.
- Under `Administration->Analysis Method`, turn Automatic Analysis off. 
- Under `GitHub Actions`, click `Follow the tutorial`.
- Create a `SONAR_TOKEN` in your GitHub repository settings then click `Continue`.
- To the question "What option best describes your build?", select `Other`.
- Update the `.github/workflow/python-app.yml` file to include the SonarCloud scan. For simplicity, the final file should look like this:

```
# This workflow will install Python dependencies, run tests and lint with a single version of Python
# For more information see: https://help.github.com/actions/language-and-framework-guides/using-python-with-github-actions

name: Python application

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

permissions:
  contents: read

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    - name: Set up Python 3.10
      uses: actions/setup-python@v3
      with:
        python-version: "3.10"
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install flake8
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
    - name: Lint with flake8
      run: |
        # stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
    - name: Test with pytest
      run: |
        pytest --cov --cov-report xml:cov.xml --cov-config=.coveragerc
    - name: SonarCloud Scan
      uses: SonarSource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

We still need to create the analysis configuration file:

- Create a `sonar-project.properties` file the root of the repository. You can copy and paste the following (replace the placeholders with your project and organization keys)

```
sonar.projectKey={{YOUR_PROJECT_KEY}}
sonar.organization={{YOUR_ORGANIZATION_KEY}}

sonar.sources=pokedex
sonar.tests=tests
sonar.python.coverage.reportPaths=cov.xml
```

Let's commit this on the main branch and push it by running:
`git add .` then `git commit -m "Add CI analysis and coverage"` and `git push`.

Let's also rebase our PR immediately by running: 
`git checkout add-feature`, `git rebase main` and `git push --force`.

A new analysis should have been triggered for the main branch as well as the pull request. When it's done, we should see the overall coverage for our project as well as the one for our PR.

## (Extra: import Flake8 reports into SonarCloud)

You're already using tools like Flake8 in your CI and want to visualize its report in the SonarCloud UI?

This is possible by redirecting flake8 output to a file: `flake8 --output-file=flake8report.txt` and then adding the property
`sonar.python.flake8.reportPaths=flake8report.txt` to your `sonar-project.properties` file. Note that the report will be displayed as-is and it will not be possible to silence issues from SonarCloud UI.


# SonarLint: Fix issues before they exist

In your IDE, you can install the SonarLint plugin to detect issues before even committing them.


## Synchronize issues between SonarCloud and SonarLint

By default, SonarLint analyses the currently opened file with its default configuration.
It means that if you are using a different quality profile on SonarCloud, decided to silence some issues, or have an older version of the analyzer than what is available on SonarCloud there may be discrepancies between the two tools.

To remedy to that, you can use SonarLint connected mode, which will retrieve your quality profile as well as the silenced issues from SonarCloud to offer you a consistent experience.

For more information about SonarLint and its connected mode, you can visit the [SonarLint website](https://docs.sonarcloud.io/improving/sonarlint/) as well as the [SonarCloud documentation](https://docs.sonarcloud.io/improving/sonarlint/).

# Final words

Thank you for following this workshop!

If you'd like to know more, feel free to visit [our website](https://sonarsource.com/) or our [community forum](https://community.sonarsource.com/). 
