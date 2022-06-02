# learn_shinytest2_LisaBranch

This project is a copy from: 
 - https://github.com/sbhagerty/learn_shinytest2
 
With references, useful details, and pulling in documentation from: 
 - https://github.com/rstudio/shinytest2
 - https://docs.google.com/presentation/d/1PQ_xZ4MGqB_edc26ty3a97eCM55gwKgPsJ1h1mpEjWA/edit#slide=id.g12d9053b0ec_0_44
 - https://rstudio.github.io/renv/articles/renv.html
 - https://github.com/colearendt/shinytest-example 
 
The goal of this example is to walk users through setting up a testing and automated publishing pipeline (continuous integration/continuous deployment) using github actions. To that end we can break this down into three separate chunks that will be put together at the end: 

1. Reproduceability
   - Using [git](https://happygitwithr.com/), [usethis](https://usethis.r-lib.org/index.html), and [reproduceable environments](https://environments.rstudio.com/) using [renv](https://rstudio.github.io/renv/articles/renv.html). 

2. Testing 
   - Using [shinytest2](https://rstudio.github.io/shinytest2/) based on the [testthat](https://testthat.r-lib.org/) workflow. For examples the [R Packages](https://r-pkgs.org/tests.html) documentation on testing might be useful.  

3. Automation
   - Using [github actions](https://docs.github.com/en/actions) and various community built action scripts to simplify the process such as [the actions written by the r-lib team](https://github.com/r-lib/actions). 

## Trevor's run through: Reproduceability

This example is mimicking a workflow where a developer is using [renv](https://rstudio.github.io/renv/articles/renv.html) however the project isn't currently using git. We'll be walking through the steps of setting up the provided project files on the [Workbench server provided by RStudio SolEng](https://colorado.rstudio.com/), loading the developer provided environment, and setting up [git](https://happygitwithr.com/) change control using [usethis](https://usethis.r-lib.org/index.html). 

1. Download the zip file. After creating a new RStudio session and creating a new project upload the files using the IDE. 

2. Set up our environment by running 'renv::restore()' in the console and then reload the session 

3. Initialize git with 'usethis::use_git()'

4. In order to create a branch so we can do work on it we need to commit the changes we've made. Use the RStudio IDE for committing and pushing changes (include everything except the renv folder and .rprofile file)

5. Create the Personal Access Token by running `usethis::create_github_token()`. Running this will pop up another chrome window for using the git web interface. 

6. Cache the credential with `gitcreds::gitcreds_set()`

7. Create the repo with `usethis::use_github()`

So we have now taken this project where we uploaded a zip onto workbench and we have now set it up in git so we can take advantage of automated workflows. 


## Trevor's run through: Testing

Now let's get set up for testing. We can either develop tests interactively or can programmatically create tests and run them manually or through automation (which is what we will be doing below). Tests are stored in the [`./tests/testthat/`](./tests/testthat/) folder. 

Dependencies: 

1. Install Shinytest dependencies with `shinytest::installDependencies()`.
 
2. Install the dev version of pak to resolve the map_mold dependency error (see: https://github.com/r-lib/pak/issues/298) with 'install.packages("pak", repos = "https://r-lib.github.io/p/pak/dev/")'.
 
3. Load the installed `library(pak)`.
 
4. Install `install.packages("shinyvalidate")`.
 
Creating and running tests manually: 

1. Load `library(shinytest2)`
 
2. To create a new test run `record_test()` or you can programmatically edit the tests in the [`./tests/testthat/`](./tests/testthat/) folder.  
 
3. Interact with your app, setting inputs and recording expected outputs. Save test and exit. 
 
4. Run test with `shinytest2::test_app()`.

Tip: The shinytest package commands include testApp() - don't do this! This is antiquated. 

## Trevor's run through: Github Actions

Github actions are a new capability of using triggers during the git workflow (such as on committing a project, pushing a project, or on PR) kicking off a series of steps defined in a recipe yaml file. 

Tip: In the file explorer click on the wheel and select "show hidden files"

### First goal: Publish to connect server using a github action 

Let's setup and run our first Github Actions workflow - automated running of a test the deployment to the Connect server. This yaml recipe file lives at  [`.github/workflows/connect-publish.yaml`](./.github/workflows/connect-publish.yaml)

Setting up for publishing to Connect: 

1. Create an API key on the Connect server you will later be deploying to and in GitHub Actions on your repo, set the `CONNECT_API_KEY` secret
   - Go to the repository on GitHub
   - Navigate to Settings > Secrets
   - Create a "New Repository Secret" with an API key

2. Create the manifest document by running in the console: `rsconnect::writeManifest()`. This document defines what will be included in the deployment to the Connect server when called later using the automation we are setting up. 

Creating the automation yaml recipe: 

1. Open [`.github/workflows/connect-publish.yaml`](./.github/workflows/connect-publish.yaml) - we'll use this as a starting point and modify it so we can get things working. 

2. Change branch name to match the one you have (In this example from 'main' to 'master')

3. Add the correct Connect URL 

4. Access type is changed according to what Connect has set (leave as acl)

5. Set the dir. The info included in this block are all separate examples. When done using this naming convention this field sets the name of the published document as well as the vanity url. (Future work: figure out how to set name separate from url)

6. Update the runs-on section as needed for the hosted virtual environments / operating systems you want it to test on

7. Verify that your recipe yaml now looks something like: 

```
name: connect-publish
on:
  push:
    branches: [master]

jobs:
  connect-publish:
    name: connect-publish
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - name: Publish Connect content
        uses: rstudio/actions/connect-publish@main
        with:
          url: https://colorado.rstudio.com/rsc
          api-key: ${{ secrets.CONNECT_API_KEY }}
          access-type: acl
          dir: |
            .:/shiny-workshop-test 
            
```

9. Test the automation by committing and pushing the changes. Github will see the action and will automatically run the defined recipe and on push will try to publish the app to the Connect server specified. 

10. Open a browser window with your git repo and go to actions -> workflows. Watch real time it's progress. 

### Second goal: Publish to connect server after testing using a github action 

Let's now set up an additional step and bring all the above parts together by adding testing to our workflow. In this example we are assuming that we would want testing to be kicked off on two conditions; (1) whenever changes are pushed to the repo and (2) if a PR is requested. In addition we want publishing to the Connect server to only happen when tests are successful. To that end we will be adding a second recipe yaml called [`.github/workflows/test-actions.yaml`](./.github/workflows/test-actions.yaml) that will be called by [`.github/workflows/connect-publish.yaml`](./.github/workflows/connect-publish.yaml) that will be setup to run when called (using the workflow-call parameter) and when certain triggers are met (on PR). 

- If the tests succeed, the workflow run will pass

- If the tests differ or fail, the workflow run will fail

- To review the results directly, you have to download the build artifacts

- Alternatively, since tests are failing, it means that something about the
application has changed. Tests may need to be updated (or code fixed) to address
the changes. To do this, you can run the tests locally or address any code
changes necessary

Make changes to the main recipe yaml [`.github/workflows/test-actions.yaml`](./.github/workflows/test-actions.yaml): 

1. By default github actions run in parallel. We need to add `needs: [test-app]`. 

2. Make changes to add the [`.github/workflows/test-actions.yaml`](./.github/workflows/test-actions.yaml)  recipe yaml to our main recipe.

3. Verify that after making changes the yaml now looks something like: 

```
name: connect-publish
on:
  push:
    branches: [master]

jobs:
  test-app:
    uses: ./.github/workflows/test-actions.yaml
  connect-publish:
    name: connect-publish
    needs: [test-app]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Publish Connect content
        uses: rstudio/actions/connect-publish@main
        with:
          url: https://colorado.rstudio.com/rsc/
          api-key: ${{ secrets.CONNECT_API_KEY }}
          access-type: acl
          dir: |
            .:/shiny-workshop-test-prod
```

Create the [`.github/workflows/connect-publish.yaml`](./.github/workflows/connect-publish.yaml) recipe: 

What this recipe does is:

1. Checkout the code 

2. Setup pandoc 

3. Setup the R version using the RStudio public rspm 

4. Remove the .Rprofile file (so it can't conflict with renv)

5. Use r-lib actions to set up the environment 

6. Run the tests using shinytest2

7. (optional) There's an option to run tests across multiple OS's and R versions using a "matrix" parameter. This example has that removed. 
 
8. Verify that yYour test yaml looks something like: 

```
on:
  pull_request:
    branches: [master]
  workflow_call:
  
name: test-app

jobs:
  test-app:
    name: test-app
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - uses: r-lib/actions/setup-pandoc@v2

      - uses: r-lib/actions/setup-r@v2
        with:
          r-version: release
          use-public-rspm: true

      # Connect does not like `renv`'s `./.Rprofile`
      # Removing from deployment as Connect listens to the `./manifest.json` file
      - name: Remove `.Rprofile`
        shell: bash
        run: |
          rm .Rprofile

      - uses: r-lib/actions/setup-renv@v2 # use our renv.lock

      - name: Test app
        uses: rstudio/shinytest2/actions/test-app@main
```

Tip: The packages being pulled in are cached. First time running an action will take longer but next time should be faster. 


## Debugging

Upcoming



