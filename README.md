# Deploying to Azure Web Apps with DevOps and ChatOps

The following walkthrough will help you set up a demo to showcase:

1. Integration between Microsoft Teams and GitHub
1. CI/CD with GitHub Actions
1. Environment protection with GitHub Environments
1. Azure Web App for Containers
1. Azure Bicep for resource deployment

Read more about it on my [blog](https://www.cloudamentals.com/posts/github-environments/)

## Pre-requisites

You will need:

1. An Azure subscription (free trial is fine)
1. A GitHub account
1. Command line with az cli installed, or use [Azure Cloud Shell](http://shell.azure.com/)
1. Microsoft Teams

## Fork the source code

1. Make sure you are logged into GitHub, then navigate to https://github.com/CharleneMcKeown/Tailwind and fork the repo. This will create a copy of the repo in your GitHub account. 
1. Clone the newly created repo from your account (optional - we will be making small code changes that can be done in the browser)

## Setup Teams

Create two new connectors for your target Microsoft Teams channel.

1. Incoming Webhook - see [here](https://techcommunity.microsoft.com/t5/microsoft-365-pnp-blog/how-to-configure-and-use-incoming-webhooks-in-microsoft-teams/ba-p/2051118) for instructions. Make sure you save the webhook uri to your text editor. 
2. GitHub Connector - see [here](https://github.com/integrations/microsoft-teams) for instructions. 


## Generate secrets

In order to deploy and access Azure resources, we need an identity that has the right permissions.

1. In your terminal or Cloud Shell, run the following command:

    ```
    az ad sp create for rbac
    ```

    The output will look something like this:
    ```
        {
        "appId": "",
        "displayName": "",
        "name": "",
        "password": "",
        "tenant": ""
        }
    ```
1. Copy and paste the output into a text editor.
1. We need to change this slightly. Update **appId** to **clientId**, **password** to **clientSecret**, **tenant** to **tenantId** and finally, add in a new key value pair for your **subscriptionId**. It should now look like the below: 

    ``` {
        "clientId": "",
        "displayName": "",
        "name": "",
        "clientSecret": "",
        "tenantId": "",
        "subscriptionId": ""
        }
    ```

1. Return to your terminal or Cloud Shell, and run the following command to get the object id of your newly created service principal. Copy it and paste it into your text editor for the next step. 
    ```
    az ad sp show --id <putYourclientIdHere>
    ```

## GitHub Secrets

We now need to do some configuration in GitHub. 

1. In the repo, navigate to Settings > Secrets. 
1. Add a new secret called AZURE_CREDENTIALS, and paste in the JSON object you modified earlier in your text editor. Save it.
1. Add another secret called TEAMS_WEBHOOK, and paste in the webhook you saved earlier. Save it. 

## GitHub Environments

1. In your repo, navigate to Settings > Environments.
1. Create a new environment called Staging. 
1. Create another new environment called Production. Make sure you add an approver (it can be yourself).
1. Lastly, you will most likely have to enable actions because this is a forked repo. Navigate to the **Actions** tab and do that:

    ![Enable Actions](/Documents/Images/enableactions.PNG)


## Update Source code

1. In your repo, navigate to **Deploy/main.bicep** and click **edit**.

    ![Code change](/Documents/Images/editobjectid.PNG)

1. On line 14, add in your objectId that you saved earlier on and create a new PR at the bottom instead of commiting to main:

    ![PR](/Documents/Images/pr.PNG)

    ![PR2](/Documents/Images/pr2.PNG)

1. This should trigger the GitHub workflow 'build.yml' to build the application. Check it out on the **Actions** tab. 

    ![Check](/Documents/Images/check.PNG)

    ![Action](/Documents/Images/action.PNG)

1. Once it is green, go ahead and merge the pull request.

    ![Merge](/Documents/Images/merge.PNG)

 1. This will trigger another pipeline to build the application and deploy it!

    ![Deployment](/Documents/Images/deployment.PNG)

So what's happening?

### Azure Bicep

Azure Bicep is used to declare what our resources should look like, including dependencies and configuration. Bicep is deploying the main.bicep file, which defines the following resources:

### Azure Web App with two slots - Production and Staging

The application is a sample front end web store - Tailwind Traders. We are utilising a pre-created backend that is managed by Microsoft, and the app itself will be hosted on a Web App for Containers (Linux) app service plan. 

The web app has two deployment slots - one for staging, where we can deploy and observe the change, and one for production. When we're happy with the change, we can initiate a swap slot, which is seamless to users of the website.

### Azure Container Registry

We need to package the application and build it into a container image using Docker - so we need somewhere to host it - Azure Container Registry! As we will be pushing to the registry the newly created container image, we obviously need somewhere to securely store the password to the registry once it has been created.

### Azure Key Vault to store secrets

Azure Key Vault is the place where we will put the secret - we can securely access it later during deployments.

## Checking Progress

Okay, after reading all that - your app has probably deployed! Navigate to the **Actions** tab in your repo. You should see a deployment either in progress or finished. Click on it. 

You should see something like this if it's still in progress:

![Actions](/Documents/Images/actions.PNG)

Feel free to click it and have a look at the logs. When it is green, you should have an approval waiting:

![Actions](/Documents/Images/approval.PNG)

Here you can see that the Staging environment actually has a URL - click on that, and you should see the Tailwind Traders website has been deployed. Go ahead and approve Production by clicking on **Review Deployments**.

Meanwhile, over in Microsoft Teams, you might have noticed some notifcations:

![Teams](/Documents/Images/teams.PNG)

The incoming webhook you created earlier is getting used by a marketplace GitHub Action in the pipeline:

```
- name: Microsoft Teams Deploy Card
    uses: toko-bifrost/ms-teams-deploy-card@3.1.2
    if: always()
    with:
    webhook-uri: ${{ secrets.TEAMS_WEBHOOK }}
    github-token: ${{ secrets.GH_TOKEN }} 
    environment: production
    card-layout-start: complete
    card-layout-exit: cozy
    show-on-exit: true
    view-status-action-text: View prod deployment status
    custom-actions: |
        - text: View Production Website
        url: "http://${{ needs.deployInfra.outputs.web }}.azurewebsites.net"
```

It uses the card feature in teams, and passes some useful information for us, like:

1. environment
1. status
1. website url to check

So it is easy to see the status of a deployment without having to go directly to GitHub. 

## Branch protection

Lastly, the point of this exercise to make sure that deployments are done safely. To ensure that no changes make it out to production, we need to set up some branch policies to protect main.

1. Click on Settings > Branches > Add Rule:
    ![Branch prorection](/Documents/Images/addrule.PNG)
1. Add rules as per the below:
    ![Branch protection](/Documents/Images/addrule2.PNG) and save.


That is really all you need. To demo this:

1. Create a new branch off main
1. Edit: **/Source/Tailwind.Traders.Web/ClientApp/src/assets/locales/translation.json** to a add a new free shipping price on line 5.
1. Open a Pull Request into main
1. Observe your Actions tab - you will see that the build.yml pipeline is building and testing the application. 
1. Once it passes, complete the merge.
1. Observe the deployment pipeline and Teams notifications as it moves through the environments.

A few notes..

This is demo code and not recommended for actual production scenarios. One reason is that we're running the Bicep deployment every time we push changes to the website. This is dangerous! We're doing it here to make it easy for you to create the Azure resources with as few steps as possible, and to show you how easy it is to grab outputs from one step, and pass them to steps in downstream jobs - as well as showing off the nifty capabilities of environments and urls.

If this was real life, the website would likely face downtime during the Bicep deployment. The nice thing about using deployment slot swaps to perform the change means that users won't be impacted at all - the site will remain available throughout. 

Another reason - you will notice that every time we push a new version of our application to the container registry, we give it the tag:latest. This is not best practice. Ideally you would link the image to the build, and use something like ${{ github.sha }} instead. Unfortunately, as we are running the Bicep step every time we deploy, this won't work. 

>Side Note: Bicep doesn't yet have the ability to ignore changes to certain fields like Terraform does with lifecycle_ignore.

Hope you found this useful!

![example workflow](https://github.com/charlenemckeown/tailwind/actions/workflows/dotnetcore.yml/badge.svg)
