//
// Copyright (c) 2019 Intel Corporation
//
// SPDX-License-Identifier: Apache-2.0
//
// Kata Containers Jenkins CI pipeline.
// Written in Jenkins Declarative pipeline style.
//
// Prereqs:
//  In order to use the 'readJSON' and 'readYaml' functions, this pipeline
//  requires the 'pipeline-utility-steps' plugin to be installed.
//
//  If you wish to use the 'rebuild' plugin with this pipeline, you may
//  need to enable passing 'null' env vars through to rebuilds by executing
//  the following in your Jenkins master script console:
//  System.setProperty("hudson.model.ParametersAction.keepUndefinedParameters", "true")
//
//  In order to curl from the github API and not run into rate limiting issues,
//  you should lodge your github access token in your Jenkins master keystore
//  and match the ID with the name defined in this script in the 'credentials()' call.

// A stage Setting 'SKIPTESTS' to 'true' will skip later steps of the pipeline.
// Used to short-circuit *pass* the CI. e.g., if the script finds a
// github label of 'skip-ci', it will short-circuit return success to
// enable a CI fastpath for PRs that are known not to need a full CI test.
def SKIPTESTS
// Make a note of when we did do the check, so we can warn when we cannot
// do it (probalby due to lack of set env vars)
def DID_SKIPTEST = 'false'
// The github label we look for. If found, we skip the main body of the
// CI checks.
def SKIPLABEL='skip-ci'
// The matrix of repos/distros/tests we load from the test matrix YAML file
def testmatrix_file='Jenkins.yaml'
// Do not 'def' this map, or it will be out of scope for the test running functions.
testmatrix=[:]

pipeline {
  // Set the node to 'none' at the global level, so we can then allocate
  // specific nodes to specific steps later on. This allows us to run any
  // fast lightweight steps on the master (to save spinning up a container
  // or VM node), and also to use specific nodes for specific tests, such
  // as distro or arch specific tests.
  agent none

  // Set up some required environment vars
  environment {
    // Ideally we might think we want to set the GOPATH here, but this env
    // section is 'outside' of any node/agent space, so how would you get
    // a value for WORKSPACE or HOME that matches the agent you end up on?
    // So, we have to use some withEnv later on once we are on an agent to
    // set some env things up.

    // Kata related variables:
    CI="true"
    // repo paths that require a GOPATH prepend etc. get done later on once
    // we are inside the 'withEnv' sections.
    // We need the tests repo to run the CI scripts
    tests_repo="github.com/kata-containers/tests"
    tests_repo_dir="src/${tests_repo}"
    // We need the runtime repo to get the latest kata versions.yaml file to ensure
    // we install and test with the required tool versions.
    runtime_repo="github.com/kata-containers/runtime"
    runtime_repo_dir="src/${runtime_repo}"
    // We need the CI repo to get access to the Jenkins.yaml test matrix file.
    ci_repo_name="kata-containers/ci"
    ci_repo="github.com/${ci_repo_name}"
    ci_repo_dir="src/${ci_repo}"
    repo_under_test="${ghprbGhRepository}"
    repo_under_test_repo="github.com/${repo_under_test}"
    repo_under_test_dir="src/${repo_under_test_repo}"

    // This has to be a string, so cannot be placed in a global var :-(
    GITHUB_API_TOKEN = credentials('cc70853d-7fac-4976-be2d-093d7d366fb1')
  }

  stages {
    // DEBUG, FIXME, show the env, as that is always helpful during debug
    stage('Show env') {
      agent { label "master" }
      steps {
          script {
          echo "Showing our inner env:"
          printEnv()
          echo "Showing our shell env:"
          sh "env"
        }
      }
    }

    // Clean up any previous workspace - otherwise we can run into some
    // file clashes, like on 'git clone'.
    stage('Clean workspace') {
      agent { label "master" }
      steps {
        cleanWs()
      }
    }

    // Do some prechecks. Check that we really do need to run the CI tests on this
    // PR.
    stage('Prechecks') {
      // Run these checks on the Master node. This saves us having to spin up a
      // node container or VM, so saves us time and resource.
      agent { label "master" }
      when {
        // If none of these are set, then we can't do the curl to get the list of
        // github labels. Currently we then fall through, warn we have not done the
        // check, and then run the CI. We *could* skip the CI in this situation if
        // we wanted to...
        allOf {
          expression { GITHUB_API_TOKEN != null }
          expression { env.ghprbGhRepository != null }
          expression { env.ghprbPullId != null }
        }
      }
      steps {
        script {
          DID_SKIPTEST='true'
          // Curl the github labels for this PR (the ghprb plugin does not supply them directly),
          // and then check if we have the 'fast track the CI' label set.
          json=sh(script: "/usr/bin/curl -H \"Authorization: Basic ${GITHUB_API_TOKEN}\" https://api.github.com/repos/${env.ghprbGhRepository}/issues/${env.ghprbPullId}/labels", returnStdout: true).trim()

          // Convert the labels JSON to a list of maps, one map per label
          labs = readJSON text: "${json}"
          // Check if we have the 'skip the CI' label applied
          labs.each { labmap ->
            if ( labmap['name'] == SKIPLABEL ) {
              echo "Found CI skip label, skipping further tests"
              SKIPTESTS='true'
            }
          }
          if ( SKIPTESTS != 'true' ) {
            echo "CI fastpath not set, running full pipeline"
          }
        }
      }
    }

    // Warn when we failed to do the precheck. This is not idea, as it will appear
    // as an extra stage. it would be great if the above 'when' check had an 'else'
    // clause we could use, but afaict, it does not.
    stage('CheckPrecheck') {
      agent { label "master" }
      when {
        expression { DID_SKIPTEST == 'false' }
      }
      steps {
        echo "Warning, skipped CI skip check"
      }
    }

    stage('checkout PR') {
      agent { label "master" }
      steps {
        withEnv(["GOPATH=${env.WORKSPACE}/go"]) {
          checkout_pr()
        }
      }
    }

    // We will always need some of these to run the static checks.
    // We need to git clone the:
    // - tests repo so we can use some of its CI scripts
    // - rutime repo to get the versions.yaml file
    // - ci repo to get the Jenkins test matrix YAML
    // FIXME - ideally make this all encoded into the YAML file, including
    // if a stage is skip-able or not.
    stage('Setup test repo environment') {
      agent { label "master" }
      steps {
        withEnv(["GOPATH=${env.WORKSPACE}/go"]) {
          // || true on checkouts, in case we already pulled the repo as it could be the
          // one we are testing....
          sh '''
            git clone "https://${tests_repo}.git" "${GOPATH}/${tests_repo_dir}" || true
            git clone "https://${runtime_repo}.git" "${GOPATH}/${runtime_repo_dir}" || true
            git clone "https://${ci_repo}.git" "${GOPATH}/${ci_repo_dir}" || true
          '''
        }
      }
    }

    // Load up our test matrix data from the YAML file so we can tell
    // which tests are to be run on which distro/jobs
    stage('Load test matrix data') {
      agent { label "master" }
      steps {
// FIXME - if we are testing the CI repo, then we need to pick the config file up from the
// Jenkins checked out PR (root of WORKSPACE), and not the golang checked out
// ci repo. Code that up when we have access to the ghprbGhRepository var...
// FIXME
// // By default we run the tests using the git checkout out test repo
// testcode_rootdir = env.ci_repo_dir
// if ( env.ghprbGhRepository != null ) {
//   // but, if this PR is from the tests repo, use the Jenkins checked out
//   // source/branch of that instead, in case the PR is updating the test code.
//   if ( env.ghprbGhRepository == env.ci_repo_name ) {
//     testcode_rootdir = ""
//   }
// }
        //dir("${testcode_rootdir}") {
          script {
            sh "pwd"
            sh "ls"
            testmatrix = readYaml file: "${testmatrix_file}"
            echo "testmatrix looks like ${testmatrix}"
         // }
        }
      }
    }

    stage('Assess phases') {
      agent { label "master" }
      steps {
        script {
          phases = testmatrix['phases']

          phases.each {
            echo "Would process phase ${it} now"
          }
        }
      }
    }

    // Always run the static checks, even if we have a 'skip-ci' label.
    // Later we could add a 'really-skip-ci' label if we needed etc. to
    // not even run the static checks??
    // If the PR does not pass the static checks,
    // then the rest of the CI checks will be skipped by Jenkins.
    // FIXME - ideally make this all encoded into the YAML file, including
    // if a stage is skip-able or not.
    // FIXME - annoyingly, each repo looks like it might do its own static
    // checks slightly differently, if you look in the existing .travis.yaml
    // files. We'll just have to cater for that later..
    stage('Static checks') {
      agent { label "master" }
      steps {
        withEnv(["target_branch=${ghprbTargetBranch}",
          "GOPATH=${env.WORKSPACE}/go",
          "PATH+=/usr/local/go/bin:${GOPATH}/bin"]) {
          dir("${GOPATH}/${repo_under_test_dir}") {
            script {
              echo "In static check"
              // start with just the basic static check
              sh "env"
              sh "pwd"
              sh "ls"
              sh "sudo apt update -y -qq"
              sh "sudo apt install -y -qq curl git"
              sh "(cd ${GOPATH}/${tests_repo_dir}; .ci/install_go.sh -p -f)"
              sh "go version"
              sh ".ci/static-checks.sh"
              //sh "make"
              //sh "make test"
              //sh "make install"
            }
          }
        }
      }
    }

    // Run the CI on the primary distro (note, that does not mean this is our
    // favoured distro, it just means we needed to pick one as the primary
    // smoke test). If the primary distro does not pass, the rest of the distro
    // tests will be skipped by Jenkins.
    stage('PrimaryDistros') {
      when {
        expression { SKIPTESTS != 'true' }
      }
      steps {
        script {
          echo "Am running ${STAGE_NAME}"
          jobsmap = distroMap("primary")
          echo "jobsmap looks like ${jobsmap}"
          echo "kick it off in parallel"
          // run parallel, in case at some stage we decide more than one distro
          // or job needs to be done for the primary stage.
          parallel jobsmap
        }
      }
    }

    // Now find and run the secondary stages in parallel.
    stage('SecondaryDistros') {
      when {
        expression { SKIPTESTS != 'true' }
      }
      steps {
        script {
          echo "Am running ${STAGE_NAME}"
          jobsmap = distroMap("secondary")
          echo "jobsmap looks like ${jobsmap}"
          echo "kick it off in parallel"
          parallel jobsmap
        }
      }
    }
  }
}

// Generate a map of distro:jobfunc from the YAML data,
// of jobs that match 'type:ofType', and return it, so we can
// then parallel launch them.
// This is a little bit of 'magic' to allow us to parallel launch
// multiple stages dynamically generated from data.
def distroMap(ofType) {
  jobmap = [:]		// Store our map of name->funcs

  echo "Iterating the distro list of type ${ofType}"

  // Check here so we can mark a 'stage' as skipped
  if ( env.ghprbGhRepository == null ) {
    stage('skipping') {
      echo "Skipping tests, ghprbGhRepository variable not set"
      // This only returns out of this stage...
      return jobmap
    }
  }

  echo "testmatrix looks like ${testmatrix}"
  distromap = testmatrix[env.ghprbGhRepository]
  echo "distromap looks like ${distromap}"

  distromap.each { key, value ->
    echo " Check key:${key} of value:${value}"
    if ( value['type'] == ofType ) {
      echo "Getting func for distro:${key}"
      // store the result of the magic 'genStage()' function
      // into the map. The result of the genStage is something
      // that 'parallel' can launch.
      jobmap[key] = genStage(key)
    }
  }

  return jobmap
}

// A bit of a magic function, that generates and returns some
// 'code' that will then perform the correct job action, including
// setting up the stage name and allocating on the correct node.
def genStage(jobname) {
  echo "Generating func for distro:${jobname}"
  return {
    // ensure we don't get a null env.
    myEnv = distromap[jobname]['env'] ?: []

    withEnv(myEnv) {
      stage("stage: ${jobname}") {
        node(jobname) {
          script {
            echo "This is stage ${jobname}."
            // And then run the actual tests as defined in the YAML.
            testrunner(jobname, distromap[jobname])
          }
        }
      }
    }
  }
}

// Run our tests on the 'distroName' distro provided.
// Use that name to look in the testmatrix to get the test
// code to be run.
def testrunner(distroName, distroMap) {
  echo "Running tests for distro (${distroName})"

  stage(distroName) {
    // Check here so we can mark a 'stage' as skipped
    if ( env.ghprbGhRepository == null ) {
      stage('skipping') {
        echo "Skipping tests, ghprbGhRepository variable not set"
        // This only returns out of this stage...
        return
      }
    }
  
    echo "distroMap looks like ${distroMap}"
    // Extract the list of test commands to run for this distro
    testmap = distroMap['tests']
    echo "testmap looks like ${testmap}"
  
    testmap.each { key, value ->
      echo "test key:${key} value:${value}"
      echo " Run test:${key['name']} commands:'${key['commands']}'"

      // Setup our env
      // FIXME - would be quite nice if the env data were extracted
      // from the YAML as well.
      withEnv(["GOPATH=${env.WORKSPACE}/go"]) {
        stage(key['name']) {
          script {
            // Some groovy magic - ensures any 'single entry' command list is turned
            // into a collection, so we iterate the command itself, and not each individual
            // character in that command.
            commandcollection = [] + key['commands'] ?: [key['commands']]
            commandcollection.each {
              echo "  >>> extracted [${it}]"
              // Un-backslashed quotes (like quoted strings) upset the following
              // magic evaluate - so inject backslashes...
              it2 = it.replaceAll(/"/, /\\"/)
              echo "translated to [$it2]"
              // Some magic - expand the string we extracted to get all the ${xxx}
              // variables converted - so we can dump this out for debug. It can be
              // hard to debug otherwise as you don't naturally see the expanded string
              // that ends up in the shell.
              realcmd = evaluate(/"$it2"/)
              echo "  >>> actually executing [${realcmd}]"
              // And finally run the command!
              sh "${realcmd}"
            }
          }
        }
      }
    }
  }
}

def checkout_pr() {
  echo "Checking out the PR branch"
  // ||true as the repo may already exist (for instance, we pre-pull test, ci and runtime)
  sh '''
    git clone https://${repo_under_test_repo}.git ${GOPATH}/${repo_under_test_dir} || true
    cd ${GOPATH}/${repo_under_test_dir}
    git fetch origin refs/pull/${ghprbPullId}/head:${ghprbTargetBranch}
    git checkout ${ghprbTargetBranch}
    git branch
    git log --oneline -10
  '''
}

// Place this outside the CPS environment so we can access the getEnvironment() func.
@NonCPS
def printEnv() {
  env.getEnvironment().each { name, value -> println "Name: $name -> Value $value" }
}

