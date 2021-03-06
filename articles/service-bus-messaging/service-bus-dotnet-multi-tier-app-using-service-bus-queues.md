---
title: .NET multi-tier application using Azure Service Bus queues | Microsoft Docs
description: A .NET tutorial that helps you develop a multi-tier app in Azure that uses Service Bus queues to communicate between tiers.
services: service-bus-messaging
documentationcenter: .net
author: sethmanheim
manager: timlt
editor: ''

ms.assetid: 1b8608ca-aa5a-4700-b400-54d65b02615c
ms.service: service-bus-messaging
ms.workload: tbd
ms.tgt_pltfrm: na
ms.devlang: dotnet
ms.topic: get-started-article
ms.date: 01/10/2017
ms.author: sethm

---
# .NET multi-tier application using Azure Service Bus queues
## Introduction
Developing for Microsoft Azure is easy using Visual Studio and the free Azure SDK for .NET. This tutorial walks you through the steps to create an application that uses multiple Azure resources running in your local environment. The steps assume you have no prior experience using Azure.

You will learn the following:

* How to enable your computer for Azure development with a
  single download and install.
* How to use Visual Studio to develop for Azure.
* How to create a multi-tier application in Azure using web
  and worker roles.
* How to communicate between tiers using Service Bus queues.

[!INCLUDE [create-account-note](../../includes/create-account-note.md)]

In this tutorial you'll build and run the multi-tier application in an Azure cloud service. The front end is an ASP.NET MVC web role and the back end is a worker-role that uses a Service Bus queue. You can create the same multi-tier application with the front end as a web project, that is deployed to an Azure website instead of a cloud service. For instructions about what to do differently on an Azure website front end, see the [Next steps](#nextsteps) section. You can also try out the [.NET on-premises/cloud hybrid application](../service-bus-relay/service-bus-dotnet-hybrid-app-using-service-bus-relay.md) tutorial.

The following screen shot shows the completed application.

![][0]

## Scenario overview: inter-role communication
To submit an order for processing, the front-end UI component, running
in the web role, must interact with the middle tier logic running in
the worker role. This example uses Service Bus brokered messaging for
the communication between the tiers.

Using brokered messaging between the web and middle tiers decouples the
two components. In contrast to direct messaging (that is, TCP or HTTP),
the web tier does not connect to the middle tier directly; instead it
pushes units of work, as messages, into Service Bus, which reliably
retains them until the middle tier is ready to consume and process them.

Service Bus provides two entities to support brokered messaging:
queues and topics. With queues, each message sent to the queue is
consumed by a single receiver. Topics support the publish/subscribe
pattern in which each published message is made available to a subscription registered with the topic. Each subscription logically
maintains its own queue of messages. Subscriptions can also be
configured with filter rules that restrict the set of messages passed to
the subscription queue to those that match the filter. The following example uses
Service Bus queues.

![][1]

This communication mechanism has several advantages over direct
messaging:

* **Temporal decoupling.** With the asynchronous messaging pattern,
  producers and consumers need not be online at the same time. Service
  Bus reliably stores messages until the consuming party is ready to
  receive them. This enables the components of the distributed
  application to be disconnected, either voluntarily, for example, for
  maintenance, or due to a component crash, without impacting the
  system as a whole. Furthermore, the consuming application might only
  need to come online during certain times of the day.
* **Load leveling.** In many applications, system load varies over
  time, while the processing time required for each unit of work is
  typically constant. Intermediating message producers and consumers
  with a queue means that the consuming application (the worker) only needs to be provisioned to accommodate average load rather than peak
  load. The depth of the queue grows and contracts as the incoming
  load varies. This directly saves money in terms of the amount of
  infrastructure required to service the application load.
* **Load balancing.** As load increases, more worker processes can be
  added to read from the queue. Each message is processed by only one
  of the worker processes. Furthermore, this pull-based load balancing
  enables optimal use of the worker machines even if the
  worker machines differ in terms of processing power, as they will
  pull messages at their own maximum rate. This pattern is often
  termed the *competing consumer* pattern.
  
  ![][2]

The following sections discuss the code that implements this architecture.

## Create a namespace
The next step is to create a service namespace, and obtain a Shared Access Signature (SAS) key. A namespace provides an application boundary for
each application exposed through Service Bus. A SAS key is
generated by the system when a namespace is
created. The combination of namespace and SAS key
provides the credentials for Service Bus to authenticate access to an
application.

[!INCLUDE [service-bus-create-namespace-portal](../../includes/service-bus-create-namespace-portal.md)]

## Create a web role
In this section, you build the front end of your application. First, you
create the pages that your application displays.
After that, add code that submits items to a Service Bus
queue and displays status information about the queue.

### Create the project
1. Using administrator privileges, start Microsoft Visual
   Studio. To start Visual Studio with administrator privileges, right-click the **Visual Studio** program icon, and then click **Run as administrator**. The Azure compute emulator,
   discussed later in this article, requires that Visual Studio be
   started with administrator privileges.
   
   In Visual Studio, on the **File** menu, click **New**, and then
   click **Project**.
2. From **Installed Templates**, under **Visual C#**, click **Cloud** and
   then click **Azure Cloud Service**. Name the project
   **MultiTierApp**. Then click **OK**.
   
   ![][9]
3. From **.NET Framework 4.5** roles, double-click **ASP.NET Web
   Role**.
   
   ![][10]
4. Hover over **WebRole1** under **Azure Cloud Service solution**, click
   the pencil icon, and rename the web role to **FrontendWebRole**. Then click **OK**. (Make sure you enter "Frontend" with a lower-case 'e,' not "FrontEnd".)
   
   ![][11]
5. From the **New ASP.NET Project** dialog box, in the **Select a template** list, click **MVC**.
   
   ![][12]
6. Still in the **New ASP.NET Project** dialog box, click the **Change Authentication** button. In the **Change Authentication** dialog box, click **No Authentication**, and then click **OK**. For this tutorial, you're deploying an app that doesn't need a user login.
   
    ![][16]
7. Back in the **New ASP.NET Project** dialog box, click **OK** to create the project.
8. In **Solution Explorer**, in the **FrontendWebRole** project, right-click **References**, then click
   **Manage NuGet Packages**.
9. Click the **Browse** tab, then search for `Microsoft Azure Service Bus`. Click **Install**, and accept the terms of use.
   
   ![][13]
   
   Note that the required client assemblies are now referenced and some new code files have been added.
10. In **Solution Explorer**, right-click **Models** and click **Add**,
    then click **Class**. In the **Name** box, type the name
    **OnlineOrder.cs**. Then click **Add**.

### Write the code for your web role
In this section, you create the various pages that your application displays.

1. In the OnlineOrder.cs file in Visual Studio, replace the
   existing namespace definition with the following code:
   
   ```csharp
   namespace FrontendWebRole.Models
   {
       public class OnlineOrder
       {
           public string Customer { get; set; }
           public string Product { get; set; }
       }
   }
   ```
2. In **Solution Explorer**, double-click
   **Controllers\HomeController.cs**. Add the following **using**
   statements at the top of the file to include the namespaces for the
   model you just created, as well as Service Bus.
   
   ```csharp
   using FrontendWebRole.Models;
   using Microsoft.ServiceBus.Messaging;
   using Microsoft.ServiceBus;
   ```
3. Also in the HomeController.cs file in Visual Studio, replace the
   existing namespace definition with the following code. This code
   contains methods for handling the submission of items to the queue.
   
   ```csharp
   namespace FrontendWebRole.Controllers
   {
       public class HomeController : Controller
       {
           public ActionResult Index()
           {
               // Simply redirect to Submit, since Submit will serve as the
               // front page of this application.
               return RedirectToAction("Submit");
           }
   
           public ActionResult About()
           {
               return View();
           }
   
           // GET: /Home/Submit.
           // Controller method for a view you will create for the submission
           // form.
           public ActionResult Submit()
           {
               // Will put code for displaying queue message count here.
   
               return View();
           }
   
           // POST: /Home/Submit.
           // Controller method for handling submissions from the submission
           // form.
           [HttpPost]
           // Attribute to help prevent cross-site scripting attacks and
           // cross-site request forgery.  
           [ValidateAntiForgeryToken]
           public ActionResult Submit(OnlineOrder order)
           {
               if (ModelState.IsValid)
               {
                   // Will put code for submitting to queue here.
   
                   return RedirectToAction("Submit");
               }
               else
               {
                   return View(order);
               }
           }
       }
   }
   ```
4. On the **Build** menu, click **Build Solution** to test the accuracy of your work so far.
5. Now, create the view for the `Submit()` method you
   created earlier. Right-click within the `Submit()` method (the overload of `Submit()` that takes no parameters), and then choose
   **Add View**.
   
   ![][14]
6. A dialog box appears for creating the view. In the **Template** list, choose **Create**. In the **Model class** list, click the **OnlineOrder** class.
   
   ![][15]
7. Click **Add**.
8. Now, change the displayed name of your application. In **Solution Explorer**, double-click the
   **Views\Shared\\_Layout.cshtml** file to open it in the Visual
   Studio editor.
9. Replace all occurrences of **My ASP.NET Application** with
   **LITWARE'S Products**.
10. Remove the **Home**, **About**, and **Contact** links. Delete the highlighted code:
    
    ![][28]
11. Finally, modify the submission page to include some information about
    the queue. In **Solution Explorer**, double-click the
    **Views\Home\Submit.cshtml** file to open it in the Visual Studio
    editor. Add the following line after `<h2>Submit</h2>`. For now,
    the `ViewBag.MessageCount` is empty. You will populate it later.
    
    ```html
    <p>Current number of orders in queue waiting to be processed: @ViewBag.MessageCount</p>
    ```
12. You now have implemented your UI. You can press **F5** to run your
    application and confirm that it looks as expected.
    
    ![][17]

### Write the code for submitting items to a Service Bus queue
Now, add code for submitting items to a queue. First, you
create a class that contains your Service Bus queue connection
information. Then, initialize your connection from
Global.aspx.cs. Finally, update the submission code you
created earlier in HomeController.cs to actually submit items to a
Service Bus queue.

1. In **Solution Explorer**, right-click **FrontendWebRole** (right-click the project, not the role). Click **Add**, and then click **Class**.
2. Name the class **QueueConnector.cs**. Click **Add** to create the class.
3. Now, add code that encapsulates the connection information and initializes the connection to a Service Bus queue. Replace the entire contents of QueueConnector.cs with the following code, and enter values for `your Service Bus namespace` (your namespace name) and `yourKey`, which is the **primary key** you previously obtained from the Azure portal.
   
   ```csharp
   using System;
   using System.Collections.Generic;
   using System.Linq;
   using System.Web;
   using Microsoft.ServiceBus.Messaging;
   using Microsoft.ServiceBus;
   
   namespace FrontendWebRole
   {
       public static class QueueConnector
       {
           // Thread-safe. Recommended that you cache rather than recreating it
           // on every request.
           public static QueueClient OrdersQueueClient;
   
           // Obtain these values from the portal.
           public const string Namespace = "your Service Bus namespace";
   
           // The name of your queue.
           public const string QueueName = "OrdersQueue";
   
           public static NamespaceManager CreateNamespaceManager()
           {
               // Create the namespace manager which gives you access to
               // management operations.
               var uri = ServiceBusEnvironment.CreateServiceUri(
                   "sb", Namespace, String.Empty);
               var tP = TokenProvider.CreateSharedAccessSignatureTokenProvider(
                   "RootManageSharedAccessKey", "yourKey");
               return new NamespaceManager(uri, tP);
           }
   
           public static void Initialize()
           {
               // Using Http to be friendly with outbound firewalls.
               ServiceBusEnvironment.SystemConnectivity.Mode =
                   ConnectivityMode.Http;
   
               // Create the namespace manager which gives you access to
               // management operations.
               var namespaceManager = CreateNamespaceManager();
   
               // Create the queue if it does not exist already.
               if (!namespaceManager.QueueExists(QueueName))
               {
                   namespaceManager.CreateQueue(QueueName);
               }
   
               // Get a client to the queue.
               var messagingFactory = MessagingFactory.Create(
                   namespaceManager.Address,
                   namespaceManager.Settings.TokenProvider);
               OrdersQueueClient = messagingFactory.CreateQueueClient(
                   "OrdersQueue");
           }
       }
   }
   ```
4. Now, ensure that your **Initialize** method gets called. In **Solution Explorer**, double-click **Global.asax\Global.asax.cs**.
5. Add the following line of code at the end of the **Application_Start** method.
   
   ```csharp
   FrontendWebRole.QueueConnector.Initialize();
   ```
6. Finally, update the web code you created earlier, to
   submit items to the queue. In **Solution Explorer**,
   double-click **Controllers\HomeController.cs**.
7. Update the `Submit()` method (the overload that takes no parameters) as follows to get the message count
   for the queue.
   
   ```csharp
   public ActionResult Submit()
   {
       // Get a NamespaceManager which allows you to perform management and
       // diagnostic operations on your Service Bus queues.
       var namespaceManager = QueueConnector.CreateNamespaceManager();
   
       // Get the queue, and obtain the message count.
       var queue = namespaceManager.GetQueue(QueueConnector.QueueName);
       ViewBag.MessageCount = queue.MessageCount;
   
       return View();
   }
   ```
8. Update the `Submit(OnlineOrder order)` method (the overload that takes one parameter) as follows to submit
   order information to the queue.
   
   ```csharp
   public ActionResult Submit(OnlineOrder order)
   {
       if (ModelState.IsValid)
       {
           // Create a message from the order.
           var message = new BrokeredMessage(order);
   
           // Submit the order.
           QueueConnector.OrdersQueueClient.Send(message);
           return RedirectToAction("Submit");
       }
       else
       {
           return View(order);
       }
   }
   ```
9. You can now run the application again. Each time you submit an
   order, the message count increases.
   
   ![][18]

## Create the worker role
You will now create the worker role that processes the order
submissions. This example uses the **Worker Role with Service Bus Queue** Visual Studio project template. You already obtained the required credentials from the portal.

1. Make sure you have connected Visual Studio to your Azure account.
2. In Visual Studio, in **Solution Explorer** right-click the
   **Roles** folder under the **MultiTierApp** project.
3. Click **Add**, and then click **New Worker Role Project**. The **Add New Role Project** dialog box appears.
   
   ![][26]
4. In the **Add New Role Project** dialog box, click **Worker Role with Service Bus Queue**.
   
   ![][23]
5. In the **Name** box, name the project **OrderProcessingRole**. Then click **Add**.
6. Copy the connection string that you obtained in step 9 of the "Create a Service Bus namespace" section to the clipboard.
7. In **Solution Explorer**, right-click the **OrderProcessingRole** you created in step 5 (make sure that you right-click **OrderProcessingRole** under **Roles**, and not the class). Then click **Properties**.
8. On the **Settings** tab of the **Properties** dialog box, click inside the **Value** box for **Microsoft.ServiceBus.ConnectionString**, and then paste the endpoint value you copied in step 6.
   
   ![][25]
9. Create an **OnlineOrder** class to represent the orders as you process them from the queue. You can reuse a class you have already created. In **Solution Explorer**, right-click the **OrderProcessingRole** class (right-click the class icon, not the role). Click **Add**, then click **Existing Item**.
10. Browse to the subfolder for **FrontendWebRole\Models**, and then double-click **OnlineOrder.cs** to add it to this project.
11. In **WorkerRole.cs**, change the value of the **QueueName** variable from `"ProcessingQueue"` to `"OrdersQueue"` as shown in the following code.
    
    ```csharp
    // The name of your queue.
    const string QueueName = "OrdersQueue";
    ```
12. Add the following using statement at the top of the WorkerRole.cs file.
    
    ```csharp
    using FrontendWebRole.Models;
    ```
13. In the `Run()` function, inside the `OnMessage()` call, replace the contents of the `try` clause with the following code.
    
    ```csharp
    Trace.WriteLine("Processing", receivedMessage.SequenceNumber.ToString());
    // View the message as an OnlineOrder.
    OnlineOrder order = receivedMessage.GetBody<OnlineOrder>();
    Trace.WriteLine(order.Customer + ": " + order.Product, "ProcessingMessage");
    receivedMessage.Complete();
    ```
14. You have completed the application. You can test the full
    application by right-clicking the MultiTierApp project in Solution Explorer,
    selecting **Set as Startup Project**, and then pressing F5. Note that the
    message count does not increment, because the worker role processes items
    from the queue and marks them as complete. You can see the trace output of
    your worker role by viewing the Azure Compute Emulator UI. You
    can do this by right-clicking the emulator icon in the notification
    area of your taskbar and selecting **Show Compute Emulator UI**.
    
    ![][19]
    
    ![][20]

## Next steps
To learn more about Service Bus, see the following resources:  

* [Azure Service Bus][sbmsdn]  
* [Service Bus service page][sbacom]  
* [How to Use Service Bus Queues][sbacomqhowto]  

To learn more about multi-tier scenarios, see:  

* [.NET Multi-Tier Application Using Storage Tables, Queues, and Blobs][mutitierstorage]  

[0]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-01.png
[1]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-100.png
[2]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-101.png
[9]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-10.png
[10]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-11.png
[11]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-02.png
[12]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-12.png
[13]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-13.png
[14]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-33.png
[15]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-34.png
[16]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-14.png
[17]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-36.png
[18]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-37.png

[19]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-38.png
[20]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-39.png
[23]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/SBWorkerRole1.png
[25]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/SBWorkerRoleProperties.png
[26]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/SBNewWorkerRole.png
[28]: ./media/service-bus-dotnet-multi-tier-app-using-service-bus-queues/getting-started-multi-tier-40.png

[sbmsdn]: http://msdn.microsoft.com/library/azure/ee732537.aspx  
[sbacom]: https://azure.microsoft.com/services/service-bus/  
[sbacomqhowto]: service-bus-dotnet-get-started-with-queues.md  
[mutitierstorage]: https://code.msdn.microsoft.com/Windows-Azure-Multi-Tier-eadceb36
