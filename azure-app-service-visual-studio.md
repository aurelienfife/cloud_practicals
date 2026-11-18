# Publish a C# web app from Visual Studio

<em>Note from Aurélien: I took this original tutorial from Microsoft and used GPT to make a simplified version in markdown format. I successfully tested it.
The troubleshooting steps are not fully verified</em>

Create a one-page web app in **Visual Studio 2026 on Windows**, then publish it to a public HTTPS address on Azure App Service.

**Azure App Service is Platform as a Service (PaaS).** You write and publish the C# code. Microsoft provides the HTTPS address and manages the web server, operating system, .NET runtime and Azure platform APIs used to create and manage the service. You can deploy the app without configuring the server stack. [Microsoft Learn on OS and runtime management](https://learn.microsoft.com/en-us/azure/app-service/overview-patch-os-runtime)

## Before you start

- In Visual Studio Installer, make sure **ASP.NET and web development** is installed.
- Have an active Azure subscription that lets you create App Service resources.

## 1. Create the app

1. Open Visual Studio and choose **Create a new project**.
2. Search for **ASP.NET Core Empty** and choose the **C#** template.

   ![Visual Studio’s Create a new project dialog with the C# ASP.NET Core Empty template selected](azure%20screenshots/azure1.png)

3. Name the project `HelloAzure`. Select **.NET 10.0 (Long Term Support)** and choose **Create**.
4. Open `Program.cs` in Solution Explorer and replace its contents with this code.

```csharp
var app = WebApplication.CreateBuilder(args).Build();

app.MapGet("/", () => "Hello from my C# web app!");

app.Run();
```

5. Press **Ctrl+F5**. The browser should show **Hello from my C# web app!** at a `localhost` address. The app returns the same line of text on every visit.

   ![Browser displaying the app’s greeting at a localhost HTTPS address](azure%20screenshots/azure2.png)

## 2. Publish to Azure App Service

1. In **Solution Explorer**, right-click the **HelloAzure** project and choose **Publish**.
2. Select **Azure**, then **Azure App Service (Windows)**. Sign in if prompted and select your subscription. If you get stuck signing in, see [If MFA blocks publishing](#if-mfa-blocks-publishing).

   ![Visual Studio Publish dialog with Azure App Service (Windows) selected](azure%20screenshots/azure3.png)

3. Select **Create new** for the App Service. Accept the generated app name or enter your own unique name.
4. Create a resource group for the lab, such as `hello-azure-lab`. When creating the hosting plan, start with a region where you’ve already deployed successfully with the same Azure subscription. Select **Free (F1)** if available. If Azure rejects the region or F1 is missing, see [If Azure rejects the region](#if-azure-rejects-the-region).
5. Choose **Create**. When the new App Service appears in the list, select it and choose **Finish**. Close the setup dialog if it stays open, then click **Publish**.

   ![Visual Studio Publish dialog showing an App Service selected under its resource group](azure%20screenshots/azure4.png)

[Microsoft’s Visual Studio publishing walkthrough](https://learn.microsoft.com/en-us/azure/app-service/quickstart-dotnetcore?pivots=development-environment-vs)

## 3. Visit the live site

Open the HTTPS address shown in Visual Studio after publishing. You can also find it under **Default domain** on the App Service **Overview** page in the Azure portal. The address ends in `.azurewebsites.net`.

![Visual Studio Hosting panel with the live HTTPS address highlighted in the Site field](azure%20screenshots/azure5.png)

The browser should show **Hello from my C# web app!** at the Azure address. After publishing succeeds, it may take a few minutes for your app’s content to appear. If you see the default Azure App Service page, wait a few minutes and refresh. Once your message appears, stop the local app and refresh again. Azure is running the published copy on its managed .NET runtime.

## If MFA blocks publishing

1. Sign in to the [Azure portal](https://portal.azure.com/) with the account you use for the lab. Complete the MFA prompt and check that your lab subscription appears under **Subscriptions**. If you have multiple directories, switch to the one that contains the subscription.
2. If the portal works but Visual Studio keeps prompting or sign-in stalls, open **Tools → Options** in Visual Studio and search for **Accounts**. Set **Add and reauthenticate accounts using** to **System web browser**. Open **File → Account Settings** and add or reauthenticate the same account. Complete MFA in the browser, then retry **Publish**. [Microsoft’s Visual Studio MFA guide](https://learn.microsoft.com/en-us/visualstudio/ide/work-with-multi-factor-authentication?view=visualstudio)
3. If your work or school account asks you to register an MFA method (`AADSTS50079`), follow the setup prompt or open [Security info](https://mysignins.microsoft.com/security-info). If you’ve lost access to your verification method or see `AADSTS53003` (Conditional Access blocked the sign-in), contact your school or workplace IT team. Give them the error code, timestamp and correlation ID if shown. [Microsoft Entra sign-in troubleshooting](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-troubleshoot-sign-in-errors)

## If Azure rejects the region

1. Read the error from **Create** in Visual Studio. If it only shows a generic failure, open **Monitor → Activity log** in the Azure portal, select your lab subscription and look for the failed operation.
2. If the message says the location is blocked by policy (for example, `RequestDisallowedByPolicy`), check whether it lists allowed regions. If it doesn’t, ask your tutor or subscription administrator which regions your subscription permits. In Visual Studio, return to **Hosting Plan → New → Location** and choose an allowed region. [Azure policy error guide](https://learn.microsoft.com/en-us/azure/azure-resource-manager/troubleshooting/error-policy-requestdisallowedbypolicy)
3. If **Free (F1)** is missing, or Azure reports that the plan has a quota limit or no available capacity, try another allowed region with **Windows** and **Free (F1)** selected. If none works, send your tutor the region, plan tier and exact error before selecting a paid tier. [App Service quota and capacity guide](https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/troubleshoot-non-zone-redundant-quota-requests)

## After the lab

If you created a resource group just for this exercise and no longer need the live site, open that group in the Azure portal and choose **Delete resource group**. This removes the app and the other resources in the group.

[Original Microsoft Learn module](https://learn.microsoft.com/en-us/training/modules/host-a-web-app-with-azure-app-service/)
