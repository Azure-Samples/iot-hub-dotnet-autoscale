---
services: iot-hub
platforms: dotnet
author: stevebus
---

# Auto-scale your Azure IoT Hub

IoT Hub is scaled and priced based on an allowed number of messages per day across all devices connected to that IoT Hub. If you exceed the allowed message threshold for your chosen tier and number of units, IoT Hub will begin rejecting new messages. To date, there is no built-in mechanism for automatically scaling an IoT Hub to the next level of capacity if you approach or exceed that threshold.

The sample solution outlined in this article provides the ability to monitor an IoT Hub for the case where the current message count has exceeded a set threshold (for example, 90% of the allowed messages) and, in that case, to automatically scale the IoT Hub up to the next unit of capacity.

## Solution Overview

The solution is implemented as a set of Azure Functions leveraging the [Azure Durable Functions Framework](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-overview).   This framework, and specifically the [Stateful Singleton](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-counter) pattern within the framework, allows us to easily:

* Automatically kick off the solution in the event of an Azure App Service restart or cycle
* Schedule regular periodic execution of the function
* Ensure that one, and only one, instance of the function can be executing at one time.  This allows us to not have to worry about the complexities of race conditions that might occur if two instances of the function run at the same time.

The solution consists of three Azure Functions, each one playing a specific part in the Azure Durable Functions framework

* __**IotHubScaleInit**__ – this function is executed on a regular timer.  This function checks to see if an instance of the Orchestrator function is running and, if not, starts one.  In essence, it’s used to “kick off” the solution and make sure it’s always running.  For the sample, it is set for once per hour, but can be set to any period.
* __**IotHubScaleOrchestrator**__ – this function implements the “Orchestrator” for the solution.  It’s role in the pattern is to manage the execution of the worker function (asynchronously, and safely), and to, once it’s done, re-schedule itself for execution after a specific number of minutes.
* __**IotHubScaleWorker**__ – this is the function that performs the actions of checking to see if the IoTHub needs to be scales and, if so, scaling it.

## Prerequisites
There are several prerequisites required for development and deployment of the solution.  You need to perform the following steps to prepare:

1. Install the [latest version of Visual Studio](https://www.visualstudio.com/downloads/) (version 15.3 or greater). Include the Azure Development tools in your setup options.
2. Because the solution runs unattended (i.e. no authenticated user) in Azure, yet accesses Azure resources, we need to run it under the authentication of an authorized “application” in Azure (conceptually, the equivalent of a ‘service account’).   To that end, we need to create and authorize an Application in Azure Active Directory for our solution.   The below instructions assume you have sufficient permission in Azure to add a new application to Azure Active Directory, and to authorize that application to access your IoT Hub.  If you do not have those permissions, you’ll need to contact your Azure Administrator to do them for you.   To create and Application and authorize it, you’ll need to perform the following steps:
    1. Go to the Azure Portal (http://portal.azure.com).  On the left-hand Nav bar, choose Azure Active Directory  (you may have to click on “more services” or search for it)
    2. Make sure the directory shown at the top of the blade is the one in which you want to run your application  (the one you use to log into the portal and manage your IoT Hub)
    3. On the Azure Active Directory blade, choose “App registrations”
    4. Click on “New application registration.”  On the “Create” blade
        * Enter a name for your application (which must be unique within the AAD instance).
        * For application type, leave the default of “Web app / API”
        * For Sign-on URL, enter any validly formed URL (i.e. http://fakeurl.com).  We won’t use this URL, as this is not a ‘real’ application
    5. Once created, navigate to your new application (you may have to search for it).  One the main blade for your application, copy the Application ID and hang onto it, as we’ll use it l
    6. Click on “All settings” and then click “Keys”. Under the Passwords section, we need to create a new application password
        * Under Key description, enter a descriptive name for your key
        * Under expiration, enter your desired expiration timeframe (just remember it, if the password expires, the solution will fail to authenticate and stop working)
        * Click “Save” --   DO NOT CLOSE THE BLADE YET
        * After you click Save, the password “Value” should have been generated.  Copy and save this value somewhere safe.  You’ll need it later and you *cannot retrieve it once you leave this blade*.    (if you happen to lose it, you can return to this screen and create another password).  Close the Keys blade
    7. Now we need to give that application permission to our IoT Hub.
        * Navigate in the Azure Portal to your chosen IoT Hub. In settings, click on “Access Control (IAM)”.  Click the “Add” button
            * Under Role, choose Contributor
            * Under “Assign Access to”, leave the default of “Azure AD user, group, or application”
            * Under “Select”, search for your application you created earlier (by name)
            * Select your application from the search results and choose Save
        * You should now see that the application has access permissions to your IoT Hub
    8. To authenticate our function, we also need our subscript and tenant Ids.  You’ll need them both later
        * To get your subscription id, in the Azure Portal, on the left-hand nav bar choose “subscriptions”, find the subscription that contains your IoT Hub, and copy the Id
        * Getting the TenantId is a little trickier.  A quick web search will show you command line and powershell ways to do it.  However, from the Azure Portal, you can click on the “help” icon (the “?” in the upper right) and choose Show Diagnostics.  This will download a JSON file.  In that JSON file, you can do a search for “TenantId” and find it.  Save it for later.
    9. If you do not already have an existing Azure Function App in which to run the solution, create one per the instructions [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-function-app-portal) (choose Windows for the OS).

## Solution Development

1. Once you have downloaded or cloned the solution, open it in Visual Studio 2017
2. Fill in your specific Azure and IoT Hub parameters near the top of the IotHubScale class

```CSharp
        static string ApplicationId = "<your application id>";
        static string SubscriptionId = "<your subscription id>";
        static string TenantId = "<your tenant id>";
        static string ApplicationPassword = "<your application password>";
        static string ResourceGroupName = "<resource group containing your IoT Hub –short name>";
        static string IotHubName = "<short name of your IoT Hub>";
```

3. Note that the IotHubName should be provided WITHOUT the “.azure-devices.net”
4. If desired, adjust the
    * Job Frequency - how often the function checks to see if it needs to scale the IoT Hub
    * Threshold – the percentage of the IoT Hub message quota at which you want to scale
5. Build the solution.

## Solution Deployment

1. If you want to test the functions locally before deployment, you can [test and debug](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs#testing-functions) the functions on your development machine.
2. Once satisfied with the solution, you can publish the functions just as any other Azure function per [this](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs#publish-to-azure) guidance
3. Once published and deployed, you can navigate to your Azure Function App in the portal and confirm execution by expanding the functions and looking at the “Monitor” tab to view the logs.  Keep in mind, that the orchestrator and working functions will not execute until the timer causes the init function to execute (at the top of the next hour).   If you want to go ahead and kick off, you can run the Init function manually via the portal.

## Additional Considerations
Some additional considerations about the solution

* For simplicity of this sample, we have hard coded the various names IDs needed to identify our IoT Hub, and to authenticate to it for our management client.  For a production solution, you should look at leveraging a more secure alternative for storing these “secrets.” One option is to leverage Azure KeyVault as described [here](https://blogs.msdn.microsoft.com/dotnet/2016/10/03/storing-and-using-secrets-in-azure/). Alternately, you could store the values in Application Settings in your Function App, but ensure you have the right access control set up over the App Service to ensure only authorized users can retrieve the information
* Keep in mind that you cannot scale (auto or not) a “Free” tier IoT Hub.  As written, the sample will fail to do so.
* Just to keep the sample simple, we did not implement any notifications.  It would be useful to be notified when your hub has auto-scaled.  You can easy add notifications via [SMS/TXT (Twilio)](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-twilio) or [email (SendGrid)](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-sendgrid) bindings for the worker function
* Again, for simplicity, the sample implementation simply calculates the next ‘unit’ increment within the current Sku and cannot be used to scale between Sku’s.  You can, of course, change the logic to implement different and more complex scaling rules. For example, you could optimize around cost (once you reach ten ‘S1’ units, it’s financially cheaper to then move up to a single unit of ‘S2’.) Alternately, you could optimize around IoT Hub Throttling Limits, where continuing to add ‘S1’ units can achieve greater [throttling limits](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-quotas-throttling) than prematurely moving to ‘S2’ units.  The scaling logic for your IoT Hub may be different from hub to hub and could be based on several variables such as throttling limits, costs, and preference.
* Another consideration when scaling.  You should consider adjusting the “threshold” as you move up in Skus, as the percentages at each Sku are bigger.  For example, 10% of one unit of ‘S1’ is 40k messages, where 10% of one unit of ‘S2’ is 600k messages, and 10% of a single S3 unit is a whopping 30 million messages.  Similar scale happens even with just increasing units within a Sku.  You could implement a scheme, in GetSkuUnitThreshold, that automatically adjusts the threshold as the Sku increases (for sample, 90% when ‘S1’, and 99% when ‘S3’)
* Also keep in mind that auto-scaling ‘S3’ units should be done with due consideration, as the “wallet impact” increases more significantly as the Sku goes to higher levels

## Scaling down
For brevity, we did not implement scale-down functionality in this sample.  However, the framework used here could easily be adapted to support that scenario.   To support scale-down, you would

* Modify or replace the GetUnitSkuThreshold function to determine the threshold at which you want to scale down.   The calculation here would be a little more complicated than scaling up.  One simple example implementation would be the calculate the “threshold” based on the next lowest “unit” below the current one.  If the current message count is less than that threshold, you can safely scale down.  For example

```CSharp
        public static long GetSkuUnitThreshold(string sku, long units, int percent)
        {
          long multiplier = 0;

          if(currentUnits == 1)   // cannot scale down from 1 unit
              return -1;

            switch (sku)
            {
                case "S1":
                    multiplier = 400000;
                    break;
                case "S2":
                    multiplier = 6000000;
                    break;
                case "S3":
                    multiplier = 300000000;
                    break;
            }
            return (long)(multiplier * (units-1) * percent) / 100;
  }
```
* Since this function can now return an error condition, you’ll need to check for it in the calling code, log, and exit.
* Modify or replace the GetScaleUpTarget function to instead (or additionally) determine the target Sku/Unit to scale down to.  The implementation in this case would be simpler, if we stick with the simplicity of not auto-scaling between tiers.

```CSharp
        public static long GetScaleDownTarget(string currentSku, long currentUnits)
        {
		
            if(currentUnits >= 2)   // lowest you can scale is 1 unit
		return currentUnits--;
            else
              return -1;
}
```

* The check below (from the original code), would need to have the ‘if’ condition reversed

```CSharp
            if (currentMessageCount < messageLimit)
            {...}
```

* Finally, update comments and the various log text to reflect the new functionality.
