# Rails Project CI/CD with Capistrano

This README provides a comprehensive guide on setting up Continuous Integration (CI) and Continuous Deployment (CD) for a Rails project using GitHub Actions and Capistrano. The CI/CD process is configured for two environments: UAT and Production, with specific branch triggers for each.

## Overview

- **CI Phase**: Triggered on PR open, synchronize, or reopen events on `uat` and `production` branches.
- **CD Phase**: Executed only when PRs are merged into either the `uat` or `production` branches.

## Continuous Integration (CI) Workflow

The CI workflow is designed to run tests and lint the codebase whenever a new PR is opened, synchronized, or reopened on the `uat` or `production` branches.

### CI Workflow Steps:

### 1. CI-Test Job
   - Runs the test suite against the latest codebase.
   - Triggered on PR actions without merge.
   - Utilizes PostgreSQL service for database-related tests.
   - Includes Ruby setup, dependency installation, and test execution.

#### CI-Test Workflow YAML
```yml
name: Rails CI Workflow

on:
  pull_request:
    branches:
      - uat
      - production
    types: [opened, synchronize, reopened]
  workflow_dispatch:
jobs:
  CI-Test:
    if: github.event.pull_request.merged == false && (github.event.action == 'opened' || github.event.action == 'synchronize' || github.event.action == 'reopened')
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:11-alpine
        ports:
          - "5432:5432"
        env:
          POSTGRES_DB: rails_test
          POSTGRES_USER: rails
          POSTGRES_PASSWORD: password
    env:
      RAILS_ENV: test
      DATABASE_URL: "postgres://rails:password@localhost:5432/rails_test"
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ env.RUBY_VERSION }}
      - name: Clear Bundler cache
        run: |
          rm -rf $GITHUB_WORKSPACE/vendor/bundle
      - name: Install Bundler
        run: gem install bundler
      - name: Install Dependencies
        run: |
           bundle install
      - name: Install Ruby and gems
        uses: ruby/setup-ruby@55283cc23133118229fd3f97f9336ee23a179fcf # v1.146.0
        with:
          bundler-cache: true
      - name: Set up database schema
        run: bin/rails db:schema:load
      - name: Install Rspec
        run: gem install rspec
      - name: Run tests
        run: bin/rake
```
   
### 2. CI-Lint Job
   - Performs code linting to maintain code quality.
   - Depends on the successful completion of the CI-Test job.
   - Includes Ruby setup, dependency installation, and linting execution.

#### CI-Lint Workflow YAML  
- Addd this secction in the bottom of the Test workflow file
  
```yml
CI-Lint:
    needs: CI-Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ env.RUBY_VERSION }}
          
      - name: Install Ruby and gems
        uses: ruby/setup-ruby@55283cc23133118229fd3f97f9336ee23a179fcf # v1.146.0
        with:
          bundler-cache: true
      - name: Security audit dependencies
        run: bin/bundler-audit --update
      - name: Install Bundler
        run: gem install bundler
      - name: Install Dependencies
        run: bundle install
      - name: Install Brakeman
        run: gem install brakeman rubocop rubocop-rails rubocop-rspec rubocop-rails_config
      - name: Security audit application code
        run: brakeman --no-exit-on-warn 
      - name: Lint Ruby files
        run: rubocop -A
```

## Continuous Deployment (CD) Workflow:

The CD workflow utilizes Capistrano for deployment to the UAT or Production environments. It's triggered when PRs are merged into the respective branches.

### CD Workflow Steps

### 1. CD Job:
   - Automates the deployment process.
   - Checks out the repository.
   - Sets up Ruby and installs necessary gems.
   - Configures SSH for server access.
   - Determines the branch (either `uat` or `production`) and executes the appropriate Capistrano deployment command.

### CI-Lint Workflow YAML  

```yml
name: Rails CD Workflow

on:
  pull_request:
    branches:
      - uat
      - production
    types:
      - closed
  workflow_dispatch:
jobs:
  CD:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    env:
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      SERVER_USER: ${{ secrets.SERVER_USER }}
      SERVER_IP: ${{ secrets.SERVER_IP }}


    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ env.RUBY_VERSION }}

      - name: Install Ruby and gems
        uses: ruby/setup-ruby@55283cc23133118229fd3f97f9336ee23a179fcf # v1.146.0
        with:
          bundler-cache: true

      - name: Install Bundler
        run: gem install bundler
      - name: Install Dependencies
        run: |
           bundle install

      - name: Install Capistrano
        run: |
          gem install capistrano
          
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          printf "%s" "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/QaServer.pem
          chmod 700 ~/.ssh
          chmod 600 ~/.ssh/QaServer.pem

        env:
          SERVER_USER: ${{ secrets.SERVER_USER }}
          SERVER_IP: ${{ secrets.SERVER_IP }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Determine branch and deploy
        run: |
          if [[ $GITHUB_REF == 'refs/heads/production' ]]; then
            bundle exec cap prod deploy
          elif [[ $GITHUB_REF == 'refs/heads/uat' ]]; then
            bundle exec cap uat deploy
          else
            echo "No deployment for branch $GITHUB_REF"
          fi
        env:
          SERVER_USER: ${{ secrets.SERVER_USER }}
          SERVER_IP: ${{ secrets.SERVER_IP }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

## Getting Started

To implement these workflows in your project:

1. **Set up GitHub Actions**:
   - Add the CI and CD workflow files in your `.github/workflows` directory.
   - Name the CI workflow file as `rails-ci.yml`.
   - Name the CD workflow file as `rails-cd.yml`.

2. **Configure Capistrano**:
   - Ensure Capistrano is set up in your Rails project.
   - Configure deployment scripts for `uat` and `production` environments.

3. **Configure Repository Secrets**:
   - Add necessary secrets like `SSH_PRIVATE_KEY`, `SERVER_USER`, and `SERVER_IP` in your GitHub repository settings.

4. **Branch Setup**:
   - Create and maintain separate branches for `uat` and `production`.

5. **Test the Workflows**:
   - Create a PR against `uat` or `production` branches to test the CI workflow.
   - Merge the PR to trigger the CD workflow and observe the deployment process.

This setup ensures a robust CI/CD pipeline, facilitating efficient testing and reliable deployments to different environments using Capistrano.
