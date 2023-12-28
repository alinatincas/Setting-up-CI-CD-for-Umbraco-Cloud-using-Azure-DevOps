> [!Note]
> UPDATE!
> Umbraco has released a CI/CD feature on Umbraco Cloud, where you can use Azure DevOps or Github Actions with Umbraco Cloud for auto-deployments from v10+. For further information you can check the [Umbraco Cloud documentation](https://docs.umbraco.com/umbraco-cloud/set-up/project-settings/umbraco-cicd). There are 2 examples for each using Bash and Powershell scripts:
> - Details the setup of a CI/CD pipeline using Azure DevOps.
> - [Example using Powershell scripts](https://github.com/umbraco/UmbracoDocs/blob/main/umbraco-cloud/set-up/project-settings/umbraco-cicd/samplecicdpipeline/azure-devops-pwsh.md)
> - [Example using Bash scripts](https://github.com/umbraco/UmbracoDocs/blob/main/umbraco-cloud/set-up/project-settings/umbraco-cicd/samplecicdpipeline/azure-devops.md)
> - Details the setup of a CI/CD pipeline using GitHub Actions.
> - [Example using Powershell scripts](https://github.com/umbraco/UmbracoDocs/blob/main/umbraco-cloud/set-up/project-settings/umbraco-cicd/samplecicdpipeline/github-actions-pwsh.md)
> - [Example using Bash scripts](https://github.com/umbraco/UmbracoDocs/blob/main/umbraco-cloud/set-up/project-settings/umbraco-cicd/samplecicdpipeline/github-actions.md)

# How to deploy to Umbraco Cloud with AzureDevOps release pipelines for Umbraco CMS v9 and above
This a step by step guide on how you can deploy to Umbraco Cloud projects with AzureDevOps release pipelines. As a bonus, at the end of this article, you can also find a guide on how you can auto-deploy to Umbraco Cloud by using Gitlab external repository with Gitlab bidirectional mirroring.

![devops](https://user-images.githubusercontent.com/27504014/220066762-f3a207d8-84a5-42a6-a699-e0f53c7465f0.png)

This guide is based on the CodeGarden talk [“Team workflow for Umbraco Cloud and Azure DevOps”](https://www.youtube.com/watch?v=Ss0tlxOujB0) with Dave´s [github repository](https://github.com/dawoe/umbraco-cloud-devops), which contains the scripts used for the guide. 

**CREDITS**: 
- Dave Woestenborghs 
- Benji Peck from Thoughtquarter

**Authors**: [Alina-Magdalena Tincas](https://www.linkedin.com/in/tincasalina/) and [Muslim Al-Ali](https://www.linkedin.com/in/muslim-al-ali/)

**Editor**: [Rheannon Lefever](https://www.linkedin.com/in/rheannon-lefever/)

# Steps:

### Umbraco Cloud on Azure DevOps
**1.** Clone down both the Azure repository and the Umbraco Cloud environment
- Clone down the Umbraco Cloud environment by using https://docs.umbraco.com/umbraco-cloud/set-up/working-locally#cloning-an-umbraco-cloud-project 
- Clone down your Azure repository https://learn.microsoft.com/en-us/azure/devops/repos/git/create-new-repo?view=azure-devops#clone-the-repo-to-your-computer 

> *Note here: do not add a .gitignore file when creating the repository

**2.** Open both cloned folders side by side and move all the files and folders with exception of .git folder from the Cloud repository to the azure repository (replace the readme file if you get the notification)
![step2](https://user-images.githubusercontent.com/27504014/219951365-aca53260-5f10-4c01-b035-677aee815b87.PNG)

**3.** Push the changes to the Azure Repository

**4.** Confirm you have the changes on your Azure Devops Repository with the Cloud repository pushed up
![step4](https://user-images.githubusercontent.com/27504014/219951414-d9d283c1-2546-4829-977a-23af2c9579c1.PNG)

### Create a pipeline

Create a new Pipeline on your Azure Repository with “Azure Repos Git” with the created repository -> choose “Starter pipeline” in the configuration part and then **paste in the code from the file named [“azure-pipelines.yaml”](https://github.com/alinatincas/Setting-up-CI-CD-for-Umbraco-Cloud-using-Azure-DevOps/blob/main/azure-pipelines.yml)** and then save and run it to your main branch.

The above code contains all the build configurations for this specific pipeline. This just runs basic fundamentals .Net Core actions such as .Net restore, build and test. Then it copies the files to artifacts folder and creates an artifacts zip file and then it publishes that artifact. 

Once this pipeline is done publishing, now we need to create a release pipeline.

### Create a release pipeline
**1.** Go to “pipelines” -> “releases” -> click to create a new release pipeline with an “empty job” 
![releasePipeline](https://user-images.githubusercontent.com/27504014/219951633-294f00d7-2ea7-42b3-bbea-a11f78283288.png)

**2.** Then add an artifact with the source type of “build” for this project and pipeline that you have created. Keep the default version to latest and the same for the autogenerated source alias.

**3.** Click on the lightning icon “continuous deployment trigger” on the artifact you have created and enable the trigger to “create a new release every time a new build is available”. The idea behind this is that we want to push from local to a Azure Repository which then triggers a build and once that is finished we then want to trigger our release pipeline

**4.** Create a task on the stage/job you have created. In order to this, click on the lightning with a person icon “pre-deployment conditions” and here we need to make sure that we have the triggers defined which would then start the deployment to the stage which is “after release” is being built.

### Create tasks for your release pipelines
Now let´s create a task for our stage/job by clicking on the “task”. Here we can create the jobs we need to go through every time our release pipeline is triggered. 

**1.** The first task will be to extract the file task. So click on the “+” button on the Agent job to add a task to it and find the “extract files”. 
- Under it´s settings, we need to make some adjustments to the “archive file patterns” (this is the format of zip files we have in our build once it is finished) to allow **7z**. The reason for adding 7 zip is due to performance optimization and easier to handle than the regular zip format.
- Here we need to specify the destination folder to extract our files from the 7z folder. We want to extract this to a temporary storage within Azure DevOps to have a place to have our files with path ```$(System.DefaultWorkingDirectory)/$(solutionNamespace).Temp```

![task](https://user-images.githubusercontent.com/27504014/219952072-80bb80a6-3a2b-4a8c-9c4d-8a67f2d51080.png)

**2.** Let´s add one more task called “PowerShell”. This task creates a PowerShell script that clones down your Umbraco Cloud environment.
- Pick “inline” type, so we can write our own PowerShell commands.
-  Add in the **script code from the file named [“script1.txt”](https://github.com/alinatincas/Setting-up-CI-CD-for-Umbraco-Cloud-using-Azure-DevOps/blob/main/script1.txt)**.

Here we specify a folder name which will then be equal to our temporary storage in Azure DevOps, Then we assign our credentials from Cloud (logging and password) and clone URL from Cloud and a Deployment URL. Then we create a new directory with a folder name and clone down everything from our deployment url to that specific folder.

**3.** We will add a 3rd task called “Delete files”. This task will make sure that once we cloned down our cloud environment this will clean up the repository so we don´t have any files. 
- On the “display name” we need to specify the path of the temporary storage that we specified on the previous task ```$(System.DefaultWorkingDirectory)/$(solutionNamespace).Cloud```
- On the “Source folder” add the same temporary storage ```$(System.DefaultWorkingDirectory)/$(solutionNamespace).Cloud```
- On “contents” we want to delete everything besides the git folder, as we need to use the git folder to push up eventually to the Cloud Repository 

![task3](https://user-images.githubusercontent.com/27504014/219952195-85671f7c-1f3d-47cd-a3c6-26efaaa32920.png)

**4.** Add a 4th task called “Copy Files”. This will copy the files from our release artifact temporary folder to our Cloud temporary folder
- On the “display name” we need to specify the path to our Cloud temporary folder ```Copy Files to: $(System.DefaultWorkingDirectory)/$(solutionNamespace).Cloud```
- The  “source folder” is our temporary release folder ```$(System.DefaultWorkingDirectory)/$(solutionNamespace).Temp```
- In the “contents” we include everything besides the git folder (make sure that !.git/**/* is on new line as seen in the image from previous step) ```**/*!.git/**/*```
- The “target folder” is our Cloud temporary folder ```$(System.DefaultWorkingDirectory)/$(solutionNamespace).Cloud```
- In the “advanced” enable the “overwrite” option

**5.** Our last task is a “PowerShell” script with the “inline” type and **paste in the script code from the file named [“script2.txt”](https://github.com/alinatincas/Setting-up-CI-CD-for-Umbraco-Cloud-using-Azure-DevOps/blob/main/script2.txt)**.
Here we are going inside our temporary cloud folder, we do some git configurations to add an identity to the person that pushes to the cloud repository. Then we have some git commands to commit and force push those changes to our git Umbraco Cloud branch with a message of a release id from Azure DevOps so we can track everything through the release id.

- On the “advance” option enable the  “Show warnings as Azure DevOps warnings” 

**6.** Save the tasks 

![taskSave](https://user-images.githubusercontent.com/27504014/219952452-a478c117-373b-49b3-8222-52ea1da53170.png)

### Specify your environment variables
On our Powershell scripts we have mentioned some environment variables which we need to create and define. 

**1.** In the same release pipeline, click on “Variables” -> “Variable Groups” -> “Manage variable groups” -> Add a new “Variable Group”. 
- Give it a name and a description
- Add the following variables (make sure to add in your credentials and mail):
![EnvironmentVariables](https://user-images.githubusercontent.com/27504014/219952563-e22e0651-18d7-4a51-a4d2-6464a18b5de5.png)

**2.** Save the variable group

**3.** Now let´s go back to the “Variables” section -> “Link Variable Group” and pick the one you have just created, choose “release” and “link” it. Now it uses all the environment variables that you just created within that group into your pipeline. 

**4.** Save the linked variable group 

### Test your release pipeline
**1.** Locally, on your DevOps repository, pull the changes and then add a new change to your repository (example: a .txt file). Then you can push from your local branch up to the azure repo which then triggered the build pipeline which then triggers the release pipeline and then it eventually pushes to Umbraco Cloud. 

**2.** On the release pipeline where you can see the commits, if you click on “build” you can see the behind scene what happens when the tasks are run in regard to the build itself 
![buildSolution](https://user-images.githubusercontent.com/27504014/219952655-239f23f1-acfc-4e9b-9d16-f6bdd554a00a.PNG)

This is simple .NET Core features, where we restore, build, do some test etc. Then we copy the web source code to artifacts, we create the artifact zip file and then we publish that artifact so we use it in our release pipeline.

You can see the same if you head over to the stage of the release pipeline and click on “logs”

**3.** If you go to your Umbraco Cloud environment history, there you will see the change.

# The end

Thank you for reading and following this guide. 

> If you have any feedback or encounter issues, please use this [link](https://github.com/alinatincas/Setting-up-CI-CD-for-Umbraco-Cloud-using-Azure-DevOps/issues) to create an issue

**Bonus guide**:
If you would like to know how you can auto-deploy to Umbraco Cloud projects with Gitlab bidirectional mirroring instead of Azure Devops release pipelines, feel free to check the following guide: https://skrift.io/issues/using-gitlab-bidirectional-mirroring-azure-devops-release-pipelines-to-auto-deploy-into-umbraco-cloud/ 
