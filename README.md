# githubcontainerdeploy

This is a sample project to demonstrate deploying a DotNet core application to Azure Container using GitHub CI workflow.

If you don't need the container you can visit this 
[post](https://dev.to/mkokabi/deploying-dotnet-core-web-app-to-azure-using-github-workflow-459d)

## Caveat
In the container deploying action we need the app-name but for creating the azure web app we need to chose a contanier image. (Chicken-Egg situation). The easy solution I chose is:
- Create a container registry. (this step is not relevant and Docker hub can be used)
- Workflow at the first stage, only build the code and create the docker image and push it to CR (container registry). Run it at least once.
- Create the azure web app and choose the CR and the image which has been created in above step.
- Now, enhance the workflow to deploy the image to the azure web app. 

## Step 1 - Creating a Azure Container registry. 
As mentioned before it's not neccessary to use Azure Container Registry and Docker hub or a any other CR can be used instead but here I have used AzureCR.

![Creating Docker registry](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image0.png?raw=true)

![Creating Docker registry](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image01.png?raw=true)

Get the username and password and the server name of CR. The server name which is needed later should be lowercase so get it from the Login server without the *azurecr.io*.
![Getting the registry credentials](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image02.png?raw=true)

## Step 2 - Creating a DotNet project

![Getting the registry credentials](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image3.png?raw=true)

## Step 3
Adding a Dockerfile to your project. 
If you use the Visual Studio to create your web project it would be created for you but if you use CLI you need to put your own. 
```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build
WORKDIR /src

COPY *.csproj ./GitHubWebApp/
RUN dotnet restore "GitHubWebApp/GitHubWebApp.csproj"

COPY . ./GitHubWebApp/
WORKDIR /src/GitHubWebApp
RUN dotnet publish -c Release -o publish

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim AS runtime
WORKDIR /app
EXPOSE 80
EXPOSE 443
COPY --from=build /src/GitHubWebApp/publish ./
ENTRYPOINT ["dotnet", "GitHubWebApp.dll"]
```

## Step 4
Pushing your code to the Github repostiroy
```PowerShell
git init
git remote add origin https://github.com/mkokabi/githubcontainerdeploy.git
git pull origin master
git add .
git commit -m "First commit including the DockerFile"
git push --set-upstream origin master
```

## Step 5
Adding security settings. In our workflow we need 4 settings. 
At this stage we should set 3 of them to be able to push our docker image to registry.
- REGISTRY_SERVERNAME
- REGISTRY_USERNAME
- REGISTRY_PASSWORD

*note:* the registry_servername should be in lower case. You can copy this from Login server and remove the *azurecr.io*.

Later we need to create the next one for deploying our application to the web app.

![Creating github security](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image5.png?raw=true)


## Step 6
Creating github workflow

![Creating github action](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image4.png?raw=true)

Select "Set up a workflow yourself"
Replace the steps with 
```YAML
    steps:
    - uses: actions/checkout@v2
      name: checkout
      
    - uses: actions/setup-dotnet@v1
      with:
        dotnet-version: '3.1.100' # SDK Version to use.
    - run: dotnet build --configuration Release

    - name: dotnet publish
      run: |
        dotnet publish -c Release -o ${{env.DOTNET_ROOT}}/GitHubWebApp 
    - uses: azure/docker-login@v1
      with:
        login-server: ${{ secrets.REGISTRY_SERVERNAME }}.azurecr.io
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - run: |
        docker build -t ${{ secrets.REGISTRY_SERVERNAME }}.azurecr.io/githubwebapp:${{ github.sha }} .
        docker push ${{ secrets.REGISTRY_SERVERNAME }}.azurecr.io/githubwebapp:${{ github.sha }} 
      
    ## This section should be commented as we haven't created the Azure web app yet
    # - uses: azure/login@v1
    #   with:
    #    creds: ${{ secrets.AZURE_CREDENTIALS }}
    # - uses: azure/webapps-container-deploy@v1
    #   with:
    #    app-name: 'GithubDotNetCoreContainer'
    #    images: '${{ secrets.REGISTRY_SERVERNAME }}.azurecr.io/githubwebapp:${{ github.sha }}'
    #    
    # - name: Azure logout
    #   run: |
    #     az logout
```

The build should be successfull.
![Build result](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image7.png?raw=true)

## Step 7
Creating an Azure web app. 

![Creating Azure Web app](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image1.png?raw=true)

Selecting the container we have created in step 6

![Selecting the container for web app](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image12.jpg?raw=true)

Test the web app.

![Testing web app](https://github.com/mkokabi/githubcontainerdeploy/blob/master/images/image6.png?raw=true)


## Step 8
Create the Role based Access to the web app using the following az. 
```
az ad sp create-for-rbac --name "{your-web-app}" --role contributor \
                            --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group} \
                            --sdk-auth
```

Remember to replace {your-web-app}, {subscription-id} and {resource-group} with your web app name, subscription id and the resource group of the web application.
This command will return a json. 

ref: https://github.com/Azure/login#configure-azure-credentials

## Step 9
Add the json returned in above step into another security in github security settings named *AZURE_CREDENTIALS*.


## Step 10
Uncomment the container deploy section now.
