# Authorize FunctionApp calls from Data Factory using Managed Identity

The intent of this repository is to provide a how-to guide on the configuration of a FunctionApp in order to be able to call it from Data Factory using its own Managed Identity.

## Create FunctionApp resource

First of all, we need to create a Function App which we are going to call from Data Factory.
Here we are going to use a C# function with an HTTP Trigger.
You have an excelent example [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-azure-function)

Here he have a series of pictures with the different steps to create the function we are going to use in this example:

**IMPORTANT**: Select Anonymous as Authorization Level, as we will be using AAD Authorization later on, we need to use neither function nor host keys

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img001.png)

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img002.png)

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img003_1.png)

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img003_2.png)

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img003_3.png)

## Configure the FunctionApp to use Active Directory Authentication

Once we have our function created, we need to configure our App to use Azure AD login. You can follow the example [here](https://docs.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad?toc=%2fazure%2fazure-functions%2ftoc.json) and continue to the next session, or continue reading.

We navigate to the resource, and under Platform features (old UI experience) or in the left pane (new UI experience), we select the *Authentication / Authorization* option:

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img004_1.png)

![CreateFunctionImage](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img004_2.png)

From there, we switch on App Service Authentication and select Azure Active Directory as Authentication. Select Express as the management mode and leave everything as is, annotating the name of the App, as you will need it later.

As the last step, to restrict app access only to users authenticated by Azure Active Directory, set **Action to take when request is not authenticated** to **Log in with Azure Active Directory**. When you set this functionality, your app requires all requests to be authenticated.

![ConfigureAppAADAuthentication](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img006.png)

## Create Data Factory resource and pipelines

Now we just have to create the Data Factory from which we are going to call the function.

![CreateDataFactory](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img012.png)

Once it is created, go to the resource, and in the Properties pane, get the Managed Identity properties:

![CreateDataFactory](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img013.png)

## Authorize Data Factory Managed Identity to FunctionApp SPN

Now we have to configure the Application that performs the authentication of the users who access the app (if you use the express mode, it has the same name as the function app resource), to perform authorization, in our case, allowing only the Data Factory Managed Identity to perform calls.

First, we have to configure the Application to only allow entities that have been assigned to it to be able to log in. For this:

1. Look for the application in ADD (if you use the express mode, has the same name as the function app resource). Look for App Registrations in the AAD, under **All Applications**:

   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img008.png)
2. In  the overview, select the managed application in local directory, that will redirect you to the entreprise application
   
   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img009.png)

   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img010.png)
3. Under properties, configure the enterprise application and set **User assignment required** to **Yes**
   
   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img015.png)

Now that only that the entities that have been assigned to the app can perform calls to the function app, we need to assign the Data Factory Managed Identity to the application.

There are 2 ways to accomplish this:

1. Assign an AAD Group which the MI will be member of
2. Assign only the MI

The only option achievable through the UI is option 1, which we will be using now. For the second part, refer to the Appendix 1 in this README (Use Custom AppRole to authorize single service principal).

In order to assign the AAD Group to the Application, we need to:

1. Create a group (or use an existing one)
   
   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img007.png)
2. Go to the entreprise application (following steps 1 and 2 from the previous section), and select the option **1.Assign users and groups**
   
   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img010.png)
3. Add the assignment, selecting the appropiate group
   
   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img011.png)
4. Go to the group, and add as a member the Managed Identity of the Data Factory (shares the same name as the resource)
   
   ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img018.png)

From this point forward, you can call the Function App from your Data Factory using MSI!

## Check the access

Go to the Data Factory Portal, and create a simple pipeline that has a Web Activy:

![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img014.png)

Configure the Web Activity settings as follows:

* URL: url of the function App (you can get it from the Function clicking the Get Function URL)
  
  ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img003_3.png)

* Method and Body: Depends on the function you just created. If you created the default HTTPTrigger function, you can use the ones provided in the image.
* *Advanced*: Select **MSI** as Authentication method, and under resource, indicate the **Application ID URI** of the Application. 
  
  ![a](https://github.com/jdocampo/Call-FunctionApp-using-ADF-MI/blob/master/images/img009_1.png)

  This also matches the base url of the Function App. For example, if your function url is: https://howto-dfafmi-fa.azurewebsites.net/api/HttpTrigger the resource is  https://howto-dfafmi-fa.azurewebsites.net (beware of not ending the URL in slash (/), or the authentication will fail)


## Appendix 1: Use Custom AppRole to authorize single service principal

First, kudos to [psignoret](https://github.com/psignoret) for this cmdlet in order to add an application role to an application.

We are going to use Powershell and AzureAD Module for this tasks.
If you don't the module yet, you can install it using:
```
Install-Module -Name AzureAD
```
After, what we need to do is:

1. Connect to the tenant using an admin account
2. Add an AppRole in which to add the Managed Identity of Data Factory
3. Obtain the Function App Application and its managed application (service principal) as well as the Data Factory managed application (Service principal)
4. Add the App Role Assignment we just created to the Data Factory managed application

Below you have the snippet in order to perform this task:

```
wget https://gist.githubusercontent.com/psignoret/45e2a5769ea78ae9991d1adef88f6637/raw/8c5be4f8c89f7316d8021e6dfc0d72ab3148714d/New-AzureADPSApplicationAppRole.ps1 -o New-AzureADPSApplicationAppRole.ps1

Connect-AzureAD -TenantId <your-tentant-id>

Get-AzureADApplication -Filter "DisplayName eq 'howto-dfafmi-fa'" | `
.\New-AzureADPSApplicationAppRole.ps1 `
-AllowedMemberTypes "Application" `
-DisplayName "ConsumerApps" `
-Description "Consumer apps have access to the Function App" `
-Value "ConsumerApps"

$FunctionApp=(Get-AzureADApplication -Filter "DisplayName eq 'howto-dfafmi-fa'")
$FASPN=Get-AzureADServicePrincipal -All $True -Filter "DisplayName eq 'howto-dfafmi-fa'"
$ADFSPN=Get-AzureADServicePrincipal -All $True -Filter "DisplayName eq 'howto-dffami-df'"

New-AzureADServiceAppRoleAssignment -ObjectId $ADFSPN.ObjectId -PrincipalId $ADFSPN.ObjectId -Id $FunctionApp.AppRoles[0].Id -ResourceId $FASPN.ObjectId
```