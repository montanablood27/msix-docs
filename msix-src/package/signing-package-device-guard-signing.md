---
description: This article describes how to sign an MSIX package with Device Guard signing, which enables enterprises to guarantee that apps come from a trusted source.
title: Sign an MSIX package with Device Guard signing
ms.date: 08/27/2020
ms.topic: article
keywords: windows 10, uwp, msix
ms.localizationpriority: medium
---

# Sign an MSIX package with Device Guard signing

> [!IMPORTANT]
> We are introducing a new version of the Device Guard Signing Service (DGSS) to be more automation friendly. The new version of the service (DGSS v2) will be available for consumption starting mid-September 2020, and you will have until the end of December 2020 to transition to DGSS v2. At the end of December 2020, the existing web-based mechanisms for the current version of the DGSS service will be retired and will no longer be available for use. Please make plans to migrate to the new version of the service between September and December 2020.
>
> Following are the major changes we are making to the service: 
> - The method for consuming the service will change to a more automation-friendly method based on PowerShell cmdlets. These cmdlets will be available as a NuGet download.
> - In order to achieve desired isolation, you will be required to get a new CI policy from DGSS v2 (and optionally sign it). 
> -	DGSS v2 will not have support for downloading leaf certificates used to sign your files (however, the root certificate will still be available to download).  Note that the certificate used to sign a file can be easily extracted from the signed file itself.  As a result, after DGSS v1 is retired at the end of December 2020, you will no longer be able to download the leaf certificates used to sign your files.
>
> The following functionality will be available via these PowerShell cmdlets:
> - Get a CI policy
> - Sign a CI policy
> - Sign a catalog 
> - Download root cert
> - Download history of your signing operations 
>
> We will share detailed instructions and NuGet location before mid-September 2020. For any questions, please contact us at DGSSMigration@microsoft.com for more information on migration.  


[Device Guard signing](/microsoft-store/device-guard-signing-portal) is a Device Guard feature that is available in the Microsoft Store for Business and Education. It enables enterprises to guarantee that every app comes from a trusted source. Starting in Windows 10 Insider Preview Build 18945, you can use SignTool in the Windows SDK to sign your MSIX apps with Device Guard signing. This feature support enables you to easily incorporate Device Guard signing into the MSIX package building and signing workflow.

Device Guard signing requires permissions in the Microsoft Store for Business and uses Azure Active Directory (AD) authentication. To sign an MSIX package with Device Guard signing, follow these steps.

1. If you haven't done so already, [sign up for Microsoft Store for Business or Microsoft Store for Education](/microsoft-store/sign-up-microsoft-store-for-business).
    > [!NOTE]
    > You only need to use this portal to configure permissions for Device Guard signing.
2. In the Microsoft Store for Business (or or Microsoft Store for Education), assign yourself a role with permissions necessary to perform Device Guard signing.
3. Register your app in the [Azure portal](https://portal.azure.com/) with the proper settings so that you can use Azure AD authentication with the Microsoft Store for Business.
4. Get an Azure AD access token in JSON format.
5. Run SignTool to sign your MSIX package with Device Guard signing, and pass the Azure AD access token you obtained in the previous step.

The following sections describes these steps in more detail.

## Configure permissions for Device Guard signing

To use Device Guard signing in the Microsoft Store for Business or Microsoft Store for Education, you need the **Device Guard signer** role. This is the least privilege role that has the ability to sign. Other roles such as **Global Administrator** and **Billing account owner** can also sign. 

 > [!NOTE]
 > Device Guard Signer role is used when you are signing as an app. Global Administrator and Billing Account Owner is used when you sign as a logged in person.

To confirm or reassign roles:

1. Sign in to the [Microsoft Store for Business](https://businessstore.microsoft.com/).
2. Select **Manage** and then select **Permissions**.
3. View **Roles**.

For more information, see [Roles and permissions in the Microsoft Store for Business and Education](/microsoft-store/roles-and-permissions-microsoft-store-for-business).

## Register your app in the Azure Portal

To register your app with the proper settings so that you can use Azure AD authentication with the Microsoft Store for Business:

1. Sign in to the [Azure portal](https://portal.azure.com/) and follow the instructions in [Quickstart: Register an application with the Microsoft identity platform](/azure/active-directory/develop/quickstart-register-app) to register the app that will use Device Guard signing.

    > [!NOTE]
    > Under **Redirect URI** section, we recommend you choose **Public client (mobile & desktop)**. Otherwise, if you choose **Web** for the app type, you will need to provide a [client secret](/azure/active-directory/develop/quickstart-configure-app-access-web-apis#add-credentials-to-your-web-application) when you obtain an Azure AD access token later in this process.

2. After you register your app, on the main page for your app in the Azure portal, click **API permissions**, under **APIs my organization uses** and add a permission for the **Windows Store for Business API**.

3. Next, select **Delegated permissions** and then select **user_impersonation**.

## Get an Azure AD access token

Next, obtain an Azure AD access token for your Azure AD app in JSON format. You can do this using a variety of programming and scripting languages. For more information about this process, see [Authorize access to Azure Active Directory web applications using the OAuth 2.0 code grant flow](/azure/active-directory/develop/v1-protocols-oauth-code). We recommend that you retrieve a [refresh token](/azure/active-directory/develop/v1-protocols-oauth-code#refreshing-the-access-tokens) along with the access token, because your access token will expire in one hour.

> [!NOTE]
> If you registered your app as a **Web** app in the Azure portal, you must provide a client secret when you request your token. For more information, see the previous section.

The following PowerShell example demonstrates how to request an access token.

```powershell
function GetToken()
{

    $c = Get-Credential -Credential $user
    
    $Credentials = New-Object System.Management.Automation.PSCredential -ArgumentList $c.UserName, $c.password
    $user = $Credentials.UserName
    $password = $Credentials.GetNetworkCredential().Password
    
    $tokenCache = "outfile.json"

    #replace <application-id> and <client_secret-id> with the Application ID from your Azure AD application registration
    $Body = @{
      'grant_type' = 'password'
      'client_id'= '<application-id>'
      'client_secret' = '<client_secret>'
      'resource' = 'https://onestore.microsoft.com'
      'username' = $user
      'password' = $password
    }

    $webpage = Invoke-WebRequest 'https://login.microsoftonline.com/common/oauth2/token' -Method 'POST'  -Body $Body -UseBasicParsing
    $webpage.Content | Out-File $tokenCache -Encoding ascii
}
```

> [!NOTE]
> We recommand that you save your JSON file for later use.

## Sign your package

After you have your Azure AD access token, you are ready to use SignTool to sign your package with Device Guard signing. For more information about using SignTool to sign packages, see [Sign an app package using SignTool](/windows/uwp/packaging/sign-app-package-using-signtool?context=%252fwindows%252fmsix%252frender#prerequisites).

The following command line example demonstrates how to sign a package with Device Guard signing.

```cmd
signtool sign /fd sha256 /dlib DgssLib.dll /dmdf <Azure AAD in .json format> /t <timestamp-service-url> <your .msix package>
```

> [!NOTE]
> * We recommend that you use one of the timestamp options when you sign your package. If you do not apply a [timestamp](signing-package-overview.md#timestamping), the signing will expire in one year and the app will need to be resigned.
> * Make sure that the publisher name in your package's manifest matches the certificate you are using to sign the package. With this feature, it will be your leaf certificate. For example, if leaf certificate is **CompanyName**, than the publisher name in the manifest must be **CN=CompanyName**. Otherwise, the signing operation will fail.
> * Only the SHA256 algorithm is supported.
> * When you sign your package with Device Guard signing, your package is not being sent over the Internet.

## Test

To test the Device Guard signing, download your certificate from the Microsoft Store for Business Portal.

1. Sign in to the [Microsoft Store for Business](https://businessstore.microsoft.com/).
2. Select **Manage** and then select **Settings**.
3. View **Devices**.
4. View **Download your organization's root certificate for use with Device Guard**
5. Click **Download** 

Install the root certificate to the **Trusted Root Certification Authorities** on your device . Install your newly signed app to verify that you have successfully signed your app with Device Guard signing.

## Common errors

Here are common errors you might encounter.

* 0x800700d: This common error means that the format of the Azure AD JSON file is invalid.
* You may need to accept the terms and conditions of Microsoft Store for Business before downloading the root certificate of Device Guard Signing. This can be done by acquiring a free app in the portal.