Integrating Sigrid CI with Mendix QSM in Azure devops
==============================================

Please note: `QSM` is the brand name used by Mendix, in this manual we will use `Sigrid`.

## Prerequisites

- You are not using the default Mendix teamserver, but you are using your own Azure server for version control of your Mendix projects.
- You would like to trigger the Sigrid analysis from within your own pipeline.
- Your pipeline runners are able to pull this [public docker image](https://hub.docker.com/r/softwareimprovementgroup/mendixpreprocessor), the image is used to preprocess the Mendix code before uploading it to Sigrid.
- You have a [Sigrid](https://qsm.mendix.com) user account. 
- You have created an [authentication token using Sigrid](authentication-tokens.md).
- You have created a Personal access (PAT) token using [warden.mendix.com](https://warden.mendix.com)

## On-boarding your system to Sigrid

On-boarding is done automatically when you first run Sigrid CI Publish. As long as you have a valid token, and that token is authorized to on-board systems, you will receive the message *system has been on-boarded to Sigrid*. Subsequent runs will then be visible in both your CI environment and [Sigrid](https://qsm.mendix.com). 

## Configuration


### Create Azure pipeline configuration file

We will create a pipeline that consists of two jobs:

- One job 'SigridPublish-for-QSM' that will publish the main branch to [Sigrid](https://sigrid-says.com) after every commit.
- One job 'SigridCI-for-QSM' to provide feedback on pull requests, which can be used as input for code reviews.

#### Docker-based analysis

The recommended approach is to run Sigrid CI using the [Docker image](https://hub.docker.com/r/softwareimprovementgroup/mendixpreprocessor) published by SIG. In the root of your repository, create a file `azure-devops-pipeline.yaml` and add the following contents:

```
stages:
  - stage: Report
    jobs:
      - job: SigridPublish-for-QSM
        pool:
          vmImage: ubuntu-latest
        container: softwareimprovementgroup/mendixpreprocessor:latest
        continueOnError: true
        condition: "eq(variables['Build.SourceBranch'], 'refs/heads/main')"
        
        steps:
          - run: |
            /usr/local/bin/entrypoint.sh 
       
            env:
              SIGRID_CI_TOKEN: $(SIGRID_CI_TOKEN)
              MENDIX_TOKEN: $(MENDIX_TOKEN)
              SIGRID_CI_CUSTOMER: 'examplecustomername'
              SIGRID_CI_SYSTEM: 'examplesystemname'
              SIGRID_CI_PUBLISH: 'publish'
            continueOnError: true
          - publish: sigrid-ci-output
            artifact: sigrid-ci-output

      - job: SigridCI-for-QSM
        pool:
          vmImage: ubuntu-latest
        container: softwareimprovementgroup/mendixpreprocessor:latest
        continueOnError: true
        condition: "ne(variables['Build.SourceBranch'], 'refs/heads/main')"
        steps:
          - run: |
            /usr/local/bin/entrypoint.sh
            
            env:
              MENDIX_TOKEN: $(MENDIX_TOKEN)
              SIGRID_CI_CUSTOMER: 'examplecustomername'
              SIGRID_CI_SYSTEM: 'examplesystemname'
              SIGRID_CI_TARGET_QUALITY: '3.5'
              SIGRID_CI_TOKEN: $(SIGRID_CI_TOKEN)  
            continueOnError: true
          - publish: sigrid-ci-output
            artifact: sigrid-ci-output
```

Note the name of the branch, which is `main` in the example but might be different for your repository. In general, most older projects will use `master` as their main branch, while more recent projects will use `main`. 

Commit and push this file to the repository, so that Azure DevOps can use this configuration file for your pipeline. If you already have an existing pipeline configuration, simply add these steps to it.

**Security note:** This example downloads the containers directly from the internet. That might be acceptable for some projects, some projects might not allow this as part of their security policy.  




### How to Create your Azure DevOps pipeline

In Azure DevOps, access the section "Pipelines" from the main menu. In this example we assume you are using a YAML file to configure your pipeline:

<img src="images/azure-configurepipeline.png" width="500" />

Select the YAML file you created in the previous step:

<img src="images/azure-selectyaml.png" width="500" />

This will display the contents of the YAML file in the next screen. 

### Add both the Sigrid credential and the Mendix PAT to environment variables

Sigrid CI reads your credentials from 2 environment variables called `SIGRID_CI_TOKEN` and `MENDIX_TOKEN`. 
To add these to your Azure pipeline, follow these steps:

Click "Variables" in the top right corner. Create a secret named `SIGRID_CI_TOKEN` and use your [Sigrid authentication token](authentication-tokens.md) as the value. And repeat this for the `MENDIX_TOKEN` obtained from warden.mendix.com

<img src="images/azure-variables.png" width="500" />

From this point, Sigrid CI will run as part of the pipeline. When the pipeline is triggered depends on the configuration: by default it will run after every commit, but you can also trigger it periodically or run it manually.

<img src="images/azure-build-status.png" width="700" />

# Usage

To obtain feedback on your commit, click on the "Sigrid CI" step in the pipeline results screen shown above. 

<img src="images/azure-indicator.png" width="300" />

The check will succeed if the code quality meets the specified target, and will fail otherwise. In addition to the simple success/failure indicator, Sigrid CI provides multiple levels of feedback. The first and fastest type of feedback is directly produced in the CI output, as shown in the following screenshot:

<img src="images/azure-feedback.png" width="600" />

The output consists of the following:

- A list of refactoring candidates that were introduced in your merge request. This allows you to understand what quality issues you caused, which in turn allows you to fix them quickly. Note that quality is obviously important, but you are not expected to always fix every single issue. As long as you meet the target, it's fine.
- An overview of all ratings, compared against the system as a whole. This allows you to check if your changes improved the system, or accidentally made things worse.
- The final conclusion on whether your changes and merge request meet the quality target.

In addition to the textual output, Sigrid CI also generates a static HTML file that shows the results in a more graphical form. This is similar to test coverage tools, which also tend to produce a HTML report. You can access the HTML report from the "published" section in the build summary.

<img src="images/azure-artifacts.png" width="500" />

In the list of published artifacts, expand the "sigrid-ci-output" section and download the index.html file to view the report.

<img src="images/azure-artifact-download.png" width="600" />

The information in the HTML report is similar to the command line output, though it includes slightly more detail.

<img src="images/feedback-report.png" width="600" />

Finally, if you want to have more information on the system as a whole, you can also access [Sigrid](http://sigrid-says.com/), which gives you more information on the overall quality of the system, its architecture, and more.

## Contact and support

Feel free to contact [SIG's support department](mailto:support@softwareimprovementgroup.com) for any questions or issues you may have after reading this document, or when using Sigrid or Sigrid CI. Users in Europe can also contact us by phone at +31 20 314 0953.
