# Accessing secrets in Azure Key Vault from ASP.NET Core

## Introduction


## Create an ASP.NET Core 2.0 application with SQL Server LocalDB

### 1. Create ASP.NET Core project

In Visual Studio 2017, create a new **ASP.NET Core Web Application** project.

In the **New ASP.NET Core Web Application** dialog, select **ASP.NET Core 2.0** from the dropdowns and then select **Web Application**.

![New ASP.NET Core Web Application Dialog](media/new-aspnet-core-app-dialog-marked-up.png)

Change authentication to **Individual User Accounts**. Then select **Store user accounts in-app**.

![Change Authentication Dialog](media/change-authentication-dialog-marked-up.png)

A project with individual authentication will be created. Open **appsettings.json** and see that the `DefaultConnection` connection string is set to a SQL Server LocalDB instance. Because the LocalDB connection string is not considered a secret, it is acceptable to store it in appsettings.json and commit it to source control.

### 2. Enable automatic database migrations

In **Startup.cs**, add an `ApplicationDbContext` parameter to the `Configure()` method. Then add a line to automatically apply database migrations on application startup.

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ApplicationDbContext dbContext)
{
    dbContext.Database.Migrate();
    
    // ...
}
```

>  Note that in a real-world application, you may want to apply migrations manually.

### 3. Test the application

Run the application and register a new user.

![Register a new user](media/register-user-local-marked-up.png)

The user is created in the SQL Server LocalDB instance.


## Publish the application to Azure

### 1. Publish the application from Visual Studio

In **Solution Explorer**, right-click on the project and select **Publish** to open the publish dialog.

Select **Azure App Service** and **Create New**.

![Publish window](media/publish-window-marked-up.png)

Click **Publish** to open the **Create App Service** dialog.

Log in to your Azure account and fill out the form:

* **App Name** - Enter a unique app name.
* **Subscription** - Select the subscription to deploy the application to.
* **Resource Group** - Click **New** and enter a unique resource group name.
* **App Service Plan** - Click **New** and create an App Service Plan with a unique name. Select a location (**East US**) and a size (**Free**).

![Create App Service - Hosting](media/create-app-service.png)

Select the **Services** tab and add a SQL Database resource.

![Create App Service - Services](media/create-app-service-services-marked-up.png)

Select a new database server. Ensure the connection string name is `DefaultConnection`.

Create the three resources: App Service Plan, SQL Server, and SQL Database.

![Create App Service](media/create-app-service-services-completed-marked-up.png)

### 2. Test the application

When the resources are provisioned in Azure and the application is published, the published application will open in a browser window.

Register a new user. A new user is created in the Azure SQL Database.


## Store the Azure SQL Database connection string in Key Vault

### 1. Enable Managed Service Identity

In the Azure portal, open the Web App that you created. Select **Managed service identity** and enable it.

![Enable Managed Service Identity](media/managed-service-identity-marked-up.png)

### 2. Create a Key Vault

In the Azure portal, create a new Key Vault in the same location and resource group as the Web App and SQL Database.

![Create Key Vault](media/create-key-vault.png)

In the **Create Key Vault** window, select **Access policies**.

By default, your user already has access to manage items in the Key Vault.

Add a new access policy to give the Web App **Get** and **List** secrets permission to the Key Vault.

![Add permissions](media/add-key-vault-permissions-marked-up.png)

Create the Key Vault.

### 3. Use Key Vault to store connection string secret

Open the Web App in the Azure portal and select **Application settings**. Copy the value of the connection string named `DefaultConnection`.

![Copy connection string](media/copy-connection-string.png)

Open the Key Vault in the Azure portal.

Select **Secrets** and add a new **Manual** secret named `ConnectionStrings--DefaultConnection`. Use the value copied from the application settings.

![Create secret](media/create-secret.png)

Create the secret.

Copy the Key Vault's DNS name from its overview page.

Open the Web App's application settings again in the Azure portal:

* Add an application setting named `KEYVAULT_ENDPOINT` and paste in the Key Vault's DNS name.
* Delete the `DefaultConnection` connection string.

In Visual Studio, right-click on the project and select **Manage NuGet Packages**. Search for the `Microsoft.Azure.Services.AppAuthentication` package (include prerelease) and install it. 

Open **Program.cs** and modify the `BuildWebHost()` method to also read configuration from Key Vault:

```csharp
using Microsoft.Azure.KeyVault;
using Microsoft.Azure.Services.AppAuthentication;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Configuration.AzureKeyVault;

// ...

public static IWebHost BuildWebHost(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((ctx, builder) =>
        {
            var keyVaultEndpoint = Environment.GetEnvironmentVariable("KEYVAULT_ENDPOINT");
            if (!string.IsNullOrEmpty(keyVaultEndpoint))
            {
                var azureServiceTokenProvider = new AzureServiceTokenProvider();
                var keyVaultClient = new KeyVaultClient(
                    new KeyVaultClient.AuthenticationCallback(
                        azureServiceTokenProvider.KeyVaultTokenCallback));
                builder.AddAzureKeyVault(
                    keyVaultEndpoint, keyVaultClient, new DefaultKeyVaultSecretManager());
            }
        })
        .UseStartup<Startup>()
        .Build();
```

Publish the application to Azure. The application should still be functional.

## Develop locally using Key Vault