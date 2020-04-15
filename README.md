Agility Through Continuous Delivery
====================================

bob comment

Git the Code!
--------------

### Clone the Repo ###

The base code can be found at https://github.com/sketchdev/devopsmw-express-hello.

There are two suggested ways of copying this code and using it on your own.  You can either 1) use GitHub and fork the repository, or you can 2) use AWS Code Commit to clone the code into.  This demo will proceed using GitHub.  If you are interested in the Code Commit option, see the README-AWS-CodeCommit.md document for instructions on that path.

#### Using GitHub ####
If you want to use GitHub instead of AWS Code Commit, you may.  Just fork the repo link to your own GitHub account.  See [GitHub's instructions](https://help.github.com/en/articles/fork-a-repo#fork-an-example-repository) on how to fork a repo.

Once the code is forked to your own GitHub repository, clone the code to your local workstation.

```shell
git clone https://github.com/<YOUR_USERNAME>/devopsmw-express-hello.git
```

#### Using AWS Code Commit ###
This demo focuses on saving your code in and running the pipeline against GitHub.  If you are reading this README yet using Code Commit, your instructions may vary slightly during the setup of the build pipeline.  Please refere to README-AWS-CodeCommit.md in this project instead.


Quick Tangent - Some Pre-Game Setup
------------------------------------

There are two AWS CloudFormation templates included with this code.  One sets up two servers to run the application -- one QA server, and one Production server.  The other template sets up permissions for these servers to be able to download the code bundle being deployed.  Because CloudFormation is outside the scope of this demo, we will not dive into the details of what all is happening.  But, because this minimal setup is required to be in place before proceeding with this DevOps demo, we'll keep the launching of these two templates to a minimum.

From the AWS Console, navigate to the CloudFormation service and then:

  1. Click the "Create Stack" button
  2. Choose upload template
  3. Click the upload button
  4. Browse to the aws-setup folder within this code project
     1. The first time through launching the templates, select the cloudformation-iam.yaml template
     2. The 2nd time through, select the cloudformation-environment.yaml template
  5. Click "Next"
  6. Name the stack
     1. The first time through, name it "DevopsDemoIam"
     2. The 2nd time through, name it "DevopsDemoEnv"
  7. Click "Next"
  8. Click "Next"
  9. Click "Create stack"
     **NOTE:** for the "-iam.yaml" template, you'll first need to check the "I acknowledge that AWS CloudFormation might create IAM resources with custom names" box
  10. Repeat the above steps for the 2nd template


CodeBuild Setup
----------------

I'll give you two guesses what Code*BUILD* is supposed to do.  For purposes of this demo, we're going to use CodeBuild to run `npm install` for our Node app and run unit tests.

Within the AWS Console, navigate to the CodeBuild service to view and configure this feature.  From within this area:

  1. Click the "Create build project" button
  2. Set up Project Configuration
     1. Give the build step a name
  3. Set up Source
     1. Select GitHub as the provider
     2. Choose your Repository
        1. Leave the "Connect using OAuth" selected
        2. Click the "Connect to GitHub" button
        3. A new window pops open. Sign in to GitHub using your GitHub credentials
        4. Once successfully authenticated, you will be taken back to the CodeBuild Setup screen.
        5. Provide your forked repository URL
  4. Set up Environment
     1. Use a Managed image
     2. Select Ubuntu as the Operating system
     3. Select Node.js as the Runtime
     4. Select the latest Runtime version (10.14.1 at the time of this writing)
     5. Leave Image version as-is
     6. Let the wizard create a New service role for you
     7. Provide a name for the new service role
  5. Set up Buildspec
     1. Leave the "Use a buildspec file" option selected.  This will require that our code has a buildspec.yml file located at the root level our our source.  We'll add that later.
  6. Leave the Artifacts section alone
  7. Leave the Logs section alone
  8. Click the "Create build project" button


CodeDeploy Setup
-----------------

### Create Application ###

Once logged in to the AWS Console, navigate to the CodeDeploy service to begin to set up this feature.  After finding your way there, the first step is to create an Application entity to be deployed.  From the "Applications" menu:

  1. Click the "Create application" button
  2. Give the application a name to identify it (i.e. DevopsDemo)
  3. Select "EC2/On-premises" as the compute platform
  4. Click the "Create application" button

### Create Deployment Groups ###

The next step is to set up your deployment group. This defines what servers are tied to the application you created in the previous section, and it maintains the settings of how your app gets deployed (rolling deploy, all-at-once, blue-green, rollbacks, etc.).

After creating the application in the previous step, you should already be on the "Create deployment group" section of the setup.  But, if you aren't, navigate to the Deploy > Applications menu on the left-nav, and then select the application you previously created.

  1. Click the "Create deployment group" button
  2. Give the deployment group a name with a "-qa" suffix
  3. Select the "DevopsMWDemoCodeDeployServiceRole" in the service role dropdown
  4. Leave deployment type as "In-place"
  5. Environment configuration:
     1. Choose "Amazon EC2 instances"
     2. Tag group 1:
        1. Key: "AppName"
        2. Value: "DevopsDemo"
     3. Click the "Add tag group" button
     4. Tag group 2:
        1. Key: "Environment"
        2. Value: "QA"
  6. Leave deployment settings as "CodeDeployDefault.AllAtOnce"
  7. Uncheck the "Enable load balancing" option as we are not using load balancers for this demo.
     (This is a really neat feature, though, that you should use in production deploys: [CodeDeploy Load Balancers](https://docs.aws.amazon.com/codedeploy/latest/userguide/integrations-aws-elastic-load-balancing.html) )
  8. Click the "Create deployment group" button

You should have successfully created a *QA* Deployment Group for this application.  Let's make a Deployment Group for *Production* real fast while we're here by repeating the steps above.  First, you need to get back to the DevopsDemo application by clicking on the "DevopsDemo" link in the "Developer Tools > CodeDeploy > Applications > DevopsDemo" breadcrumbs at the top of the page.

  1. Click "Create deployment group" again
  2. Give the deployment group a name with a "-production" suffix this time
  3. Select the "DevopsMWDemoCodeDeployServiceRole" in the service role dropdown
  4. Environment configuration:
     1. Choose "Amazon EC2 instances"
     2. Tag group 1 (same as before):
        1. Key: "AppName"
        2. Value: "DevopsDemo"
     3. Click the "Add tag group" button
     4. Tag group 2:
        1. Key: "Environment"
        2. Value: "Production"
  6. Uncheck the "Enable load balancing" option
  7. Click the "Create deployment group" button


CodePipeline Setup
-------------------

Ah, the glue that binds committing code changes to deployments.  That's what we'll be doing in this section.  To begin, open the CodePipeline service within the AWS Console.

### Main Configuration ###

  1. Click the "Create pipeline"
  2. Give the pipeline a name
  3. Let the wizard create a "New service role" for you
     **Note:** the role it creates is _FAR_ too open for any real production environment. You can either modify the one it creates, or you can create your own by referencing the [AWS docs](https://docs.aws.amazon.com/codepipeline/latest/userguide/how-to-custom-role.html).  For purposes of this demo, however, the default one it creates are fine.
     1. Tweak the service role name if you want
     2. Leave the box checked to "Allow AWS CodePipeline to create a service role..."
  4. Ensure the artifact store is set to the "Default location"
  5. Click "Next"

### Tie Source Code to the Pipeline ###

This is the step where you tell CodePipeline where to get the application source code from.  In our case, we need to specify GitHub as the provider.  The wizard is pretty good at getting the pipeline authenticated with GitHub, so begin by clicking the "Connect to GitHub" button.

  1. After clicking the "Connect to GitHub" button, a new window will pop open
     1. You should still be logged in / authorized, so all that's left is to confirm permissions being granted by GitHub to AWS Code Pipeline
     2. Approve by clicking "Authorize aws-codesuite"
     3. You should be returned back to the pipeline setup page
  2. Select the repository you forked at the beginning of this demo
  3. Select the "master" branch
  4. Leave the "GitHub webhooks" option selected
  5. Click "Next"

### Build Stage ###

This step tells the pipeline how we want our code compiled.  We'll point it to the CodeBuild setup we configured earlier.

  1. Select "AWS CodeBuild" for the build provider
  2. Make sure the selected region is the region in which you set up the CodeBuild project
  3. Select the CodeBuild project you configured earlier
  4. Click "Next"

### Tie the Deployment to the Pipeline ###

In this last phase of the pipeline setup, we'll tell CodePipeline about the Deployment Groups we set up earlier (one for QA and one for Production, remember?).

  1. Select "AWS CodeDeploy" for the provider
  2. Select the region you've been doing all this setup in
  3. Select the application you created earlier during CodeDeploy Setup (i.e. "DevopsDemo")
  4. For deployment group, choose the "XXXX-qa" group you created. We'll worry about the production one later.
  5. Click "Next"
  6. Review if you want.  I prefer skipping that and proceeding straight to clicking the "Create pipeline" button.

Once the pipeline is created, it will attempt to run.  But we aren't quite prepared for that just yet, so don't worry when it fails the first time 'round.


The Code - Enable AWS Integrations
-----------------------------------

We need to do two things to our code now to make this all work.  We have to add the buildspec.yml file so CodeBuild knows how to gather dependencies and run tests.  We also need to add the deployment scripts so CodeDeploy knows how to get our code to the right place on our servers.

### Buildspec.yml ###

For full details of the buildspec.yml syntax, please refer to the official [AWS documentation](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html).  For the purposes of this demo, we've already included a version to suit our needs in a separate code branch.

To import the buildspec.yml from the code branch, execute the following command from the root of the git repo you cloned near the beginning of this demo:

```shell
git merge origin/aws-codebuild
git commit -am "CodeBuild files added"
```

This simply adds the buildspec.yml file to our codebase so AWS can process it correctly.

### CodeDeploy Files ###

AWS CodeDeploy requires an appspec.yml file to orchestrate the code deployment -- what files go where, what permissions they should have, how to start/stop the app, etc.  For full details of appspec.yml, please refer to the official [AWS documentation](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure.html).

To import the CodeDeploy files pre-prepared for this demo, execute the following command from the root of your git repo:

```shell
git merge origin/aws-codedeploy
git commit -am "CodeDeploy files added"
git push
```

Your application should now have an appspec.yml file with two supporting script files in a `/scripts` directory.


CodePipeline Setup - Continued
-------------------------------

The code should be in good shape for automatic deploys to the QA environment.  The next step in this demo is to set up manual approvals in the pipeline before promoting the code to the production environment.

Once again, open the CodePipeline service within the AWS Console.

  1. In this list of created pipelines, click on the one you already created for this demo
  2. Click the "Edit" button for the pipeline
  3. Scroll to the bottom and click the "+ Add stage" button below the current Deploy stage
     1. Give the stage a name (i.e. "Approval")
     2. Click "Add stage"
  4. Click the "+ Add action group" button
     1. Give the action a name (i.e. "UAT-Approval")
     2. Select "Manual approval" for the action provider
     3. Leave the following fields empty for the "Configure the approval request" section. This step of setting up notifications may actually be quite useful but is left to the reader to do outside of the demo.  For assistance on that, read the [SNS Create Topic](https://docs.aws.amazon.com/sns/latest/api/API_CreateTopic.html) and [Manage Approval](https://docs.aws.amazon.com/codepipeline/latest/userguide/approvals.html) documentation.
     4. Click the "Save" button

At this point, the pipeline will wait for manual approval before proceeding to the next step, which will be deploying to production.  Let's add the production deployment step while we're still here before saving the pipeline changes.

  1. Click the "+ Add stage" button below the Approval stage we just finished setting up
     1. Give the stage a name (i.e. "Prod Deploy")
     2. Click "Add stage"
  2. Click the "+ Add action group" button
     1. Give the action a name (i.e. "Prod Deploy")
     2. Select "AWS CodeDeploy" for the action provider
     3. Select the region you've been doing all this setup in
     4. Select "BuildArtifact" as the Input artifact
     5. Select the application name you created in CodeDeploy earlier during CodeDeploy Setup (i.e. "DevopsDemo")
     6. For deployment group, choose the "XXXX-production" group you created
     7. Click "Save"
  3. Save your overall pipeline changes by clicking the "Save" button near the top
  4. Click the "Save" button again to confirm you really do want to save



Appendix
---------


### AWS Cleanup ###

  1. Services > CodePipeline
     1. Select your pipeline
     2. Click the "Delete pipeline" button
     3. Confirm
  2. Services > CodeBuild
     1. Select your build project
     2. Click the "Delete" button
     3. Confirm
  3. Services > CloudFormation
     1. Select the 2nd stack you created that set up the QA & Prod servers
     2. Choose Actions > Delete stack
     3. Confirm
     4. Once complete, select the 1st stack you created that set up permissions and roles
     5. Choose Actions > Delete stack
  4. Services > S3
     1. In the "Search for buckets" field, type "pipeline"
        **IF** you aren't already using CodePipeline for something else, this is the bucket that CodePipeline created for this demo project, and it is safe to delete it.  It is up to you to ensure your demo code is the only thing in there before deleting the bucket and all of its contents.
     2. Click the checkbox next to the pipeline bucket you wish to delete and click the "Delete" button
     3. Confirm
  5. Services > CloudWatch > Logs
     1. Find the log group for "/aws/codebuild/<YOUR_CODE_BUILD_NAME>"
     2. Click the circle selector (radio button) for that row and click Actions > Delete log group
     3. Confirm
  6. Services > IAM
     1. Roles
        1. Search for roles beginning with your initials that were created for purposes of this demo
        2. Click the checkbox next to each of those roles
        3. Click the "Delete role" button
        4. Confirm
     2. Policies
        1. Search for policies containing your initials that were created for purposes of this demo (i.e. AWSCodePipelineServiceRole)
        2. Click the radio button for the role
        3. Policy actions > Delete
        4. Confirm


### Resources ###

[Notification on Failed Builds](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-build-notifications.html)
[AWS Pipeline Walkthrough](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-simple-codecommit.html)
[CodeDeploy for On-premise Servers](https://docs.aws.amazon.com/codedeploy/latest/userguide/instances-on-premises.html)
[On-premise Deploy Walkthrough](https://docs.aws.amazon.com/codedeploy/latest/userguide/tutorials-on-premises-instance.html)
