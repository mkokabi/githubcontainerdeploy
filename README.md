# githubcontainerdeploy

This is a sample project to demonstrate deploying a DotNet core application to Azure Container using GitHub CI workflow.

### step 1
Creating a Azure Container registry. 
It's not neccessary to use Azure Container Registry and Docker hub or a local registry can be used as well but here I have used AzureCR.

image0

image01

image02

### step 2
Creating a DotNet project

image3

### step 3
Adding a Dockerfile to your project. 
If you use the Visual Studio to create your web project it would be created but if you use CLI you need to put your own
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

### step 4
Pushing your code to Github
```PowerShell
git init
git remote add origin https://github.com/mkokabi/githubcontainerdeploy.git
git pull origin master
git add .
git commit -m "First commit including the DockerFile"
git push --set-upstream origin master
```

### step 5
Creating github workflow
