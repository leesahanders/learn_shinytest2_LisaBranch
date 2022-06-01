# learn_shinytest2_LisaBranch

This project is a copy from: 
 - https://github.com/sbhagerty/learn_shinytest2
 
With references/useful details at: 
 - https://github.com/rstudio/shinytest2
 - https://docs.google.com/presentation/d/1PQ_xZ4MGqB_edc26ty3a97eCM55gwKgPsJ1h1mpEjWA/edit#slide=id.g12d9053b0ec_0_44
 - https://rstudio.github.io/renv/articles/renv.html
 - https://github.com/colearendt/shinytest-example 
 
Make sure that the environment is restored with 
`renv::restore()`

Follow the instructions in: 
 - https://github.com/colearendt/shinytest-example 
 

## Trevor's run through: Reproduceability

 - Set up our environment and then reload the session with 'renv::restore()'

 - Let's start our git setup by initializing it with 'usethis::use_git()'

 - Branch won't exist until we have committed it - let's use the RStudio IDE to commit (select everything except renv and .rprofile) 

 - Let's create the Personal Access Token. Running this will pop up another chrome window in git for using the UI with `usethis::create_github_token()`

 - Let's cache the credential with `gitcreds::gitcreds_set()`

 - Now let's create our repo with `usethis::use_github()`

So we have now taken this project where we uploaded a zip onto workbench and we have now set it up in git so we can take advantage of automated workflows. 


## Trevor's run through: Testing

Let's run our first test - we can do this interactively! 

 - Install Shinytest dependencies with `shinytest::installDependencies()`
 
 - Install the dev version of pak to resolve the map_mold dependency error (see: https://github.com/r-lib/pak/issues/298) with 'install.packages("pak", repos = "https://r-lib.github.io/p/pak/dev/")'
 
 - Load the installed `library(pak)`
 
 - Install `install.packages("shinyvalidate")`
 
 - Load `library(shinytest2)`
 
 - To create a new test run `record_test()`
 
 - Interact with your app, setting inputs and recording expected outputs.
 
 - Save test and exit. 
 
 - Make changes and run test with `test_app()`
 
Uses the testthat format -> looks in tests folder and runs the testthat.R 

To run the recording that you've already created (or created programmatically): 
`shinytest2::test_app()`

Pro tip: shinytest runs like testApp() - don't do this! This is antiquated. 

## Trevor's run through: Github Actions

Let's run our first Github Actions - automated running of a test the deployment to the Connect server. 

 - Pro tip: In the file explorer click on the wheel and select "show hidden files"

### First goal: Publish to connect server using a github action 

 1. Open .github/workflows/connect-publish.yaml - we'll use this as a starting point and modify it so we can get things working. 

1. Create an API key on the Connect server you will later be deploying to. Add that API key to the repository through the web github interface through Settings > Secrets, create a "New Repository Secret" with an API key. 

2. Change branch name to match the one you have (from main to master but only for the top one)

3. Add the correct Connect URL 

4. Access type is changed according to what Connect has set (leave as acl)

5. Let's set the dir. The info included are all separate examples. When done using this naming it will also set the name as well as the vanity url. (Future work: figure out how to set name separate from url)

6. Let's create the manifest by running in the console: `rsconnect::writeManifest()`

7. Update the runs-on section as needed for the hosted virtual environments / operating systems you want it to test on

At the end the yaml should look something like: 

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

Now when we go to commit it and push it github will see the action and will automatically run it. 

Open a browser window with your git repo and go to actions -> workflows. Check out how it ran!

### Second goal: Publish to connect server after testing using a github action 

We are going to be adding another job. We are adding the test-app job. We now have two steps - 1. test-app 2. connect-publish. 

By default github actions run in parallel. We need to add `needs: [test-app]`

At the end your main yaml should look something like: 

```
name: connect-publish
on:
  push:
    branches: [master]

jobs:
  connect-publish:
    name: connect-publish
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Publish Connect content
        uses: rstudio/actions/connect-publish@main
        with:
          url: https://colorado.rstudio.com/rsc
          api-key: ${{ secrets.CONNECT_API_KEY }}
          access-type: acl
          dir: |
            .:/shiny-workshop-test-2
```

You'll create the test-actions.yaml file which will be run in two situations

What this does is 

1. Checkout the code 

2. Setup pandoc 

3. Setup the R version using the RStudio public rspm 

4. Remove the .Rprofile file (so it can't conflict with renv)

5. Use r-lib actions to set up the environment 

6. Finally we get to run the test using shinytest2

 - The matrix is used for running on multiple OS's and R versions 
 
 - workflow-call is used so you can have separate trigger so that test-actions file will run independently under some condition but also will run if called from your main yaml file. 

 - The packages being pulled in are cache'd. First time running an action will take longer but next time should be faster. 
 
Your test yaml should look something like: 

```
on:
 # push:
  #  branches: [main]
  pull_request:
    branches: [master]
  workflow_call:
  
name: test-app

jobs:
  test-app:
    runs-on: ${{ matrix.config.os }}

    name: Test app ${{ matrix.config.os }} (${{ matrix.config.r }})

    strategy:
      fail-fast: false
      matrix:
        config:
          - {os: ubuntu-latest, r: release}

    env:
      GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
      R_KEEP_PKG_SOURCE: yes

    steps:
      - uses: actions/checkout@v2

      - uses: r-lib/actions/setup-pandoc@v2

      - uses: r-lib/actions/setup-r@v2
        with:
          r-version: ${{ matrix.config.r }}
          http-user-agent: ${{ matrix.config.http-user-agent }}
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

