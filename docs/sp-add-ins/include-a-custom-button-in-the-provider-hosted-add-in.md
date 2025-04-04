---
title: Include a custom button in the provider-hosted add-in
description: Create a custom list on the host website, add a custom button, request Read permissions, run the add-in, and test the button.
ms.date: 09/26/2023
ms.localizationpriority: medium
ms.service: sharepoint
---
# Include a custom button in the provider-hosted add-in

[!INCLUDE [sp-add-in-deprecation](../../includes/snippets/sp-add-in-deprecation.md)]

This is the third in a series of articles about the basics of developing provider-hosted SharePoint Add-ins. You should first be familiar with [SharePoint Add-ins](sharepoint-add-ins.md) and the previous articles in this series, which you can find at [Get started creating provider-hosted SharePoint Add-ins](get-started-creating-provider-hosted-sharepoint-add-ins.md#next-steps).

> [!NOTE]
> If you have been working through this series about provider-hosted add-ins, you have a Visual Studio solution that you can use to continue with this topic. You can also download the repository at [SharePoint_Provider-hosted_Add-Ins_Tutorials](https://github.com/OfficeDev/SharePoint_Provider-hosted_Add-ins_Tutorials) and open the BeforeRibbonButton.sln file.

A SharePoint Add-in can include custom actions, which is the SharePoint term for custom menu items or ribbon buttons. In this article, you'll learn how to create a custom button that synchronizes a SharePoint list with a remote database.

## Create a custom list on the host website

The custom button is going to be on the ribbon of a specific list that records the employees of the local store. In a later article in this series, you'll learn how to programmatically add a custom list to a host website, but for now you'll add one manually.

1. From the home page of the Fabrikam Hong Kong SAR Store, go to **Site Contents** > **Add an add-in** > **Custom List**.
1. In the **Adding Custom List** dialog, specify **Local Employees** as the name, and then select **Create**.
1. On the **Site Contents** page, open the **Local Employees** list.
1. On the **List** tab on the ribbon, select **List Settings**.
1. In the **Columns** section of the **List Settings** page, select the **Title** column.
1. In the **Edit Column** form, change the **Column name** from **Title** to **Name**, and then select **OK**.
1. On the **Settings** page, select **Create column**.
1. In the **Create Column** form, do the following:

   1. For **Column name**, enter **Added to Corporate DB**.
   1. Set **type** to **Yes/No (check box)**.
   1. Set **Default value** to **No**.
   1. Select **OK**. You are taken back to the **Settings** page.

1. Select **Site Contents** to open the **Site Contents** page. The tile for the new list is there. Open it.
1. Click **new item**, and on the create item form, enter a name, but do *not* select **Added to Corporate DB**. Select **Save**. The list should look similar to the following.

    *Figure 1. Local Employees list with a single item*

    ![The Local Employees list with a single item. Name is Brayden Sawtell. The value of the "Added to Corporate DB" column is No.](../images/a3371859-e42f-49ea-8f17-48d8a248b075.PNG)

## Add the custom button

In this section, you include markup in the add-in that deploys a button to the list's ribbon. When a user highlights an employee in the list and selects the button, the employee's name is added to the corporate database, and the **Added to Corporate DB** field for the employee switches from **No** to **Yes**.

1. If Visual Studio is open, you have to close it and reopen the Chain Store solution so that Visual Studio can discover your new list (run Visual Studio as an administrator).

   > [!NOTE]
   > The settings for Startup Projects in Visual Studio tend to revert to defaults whenever the solution is reopened. Always take these steps immediately after reopening the sample solution in this series of articles:
   >
   > 1. Right-click the solution node at the top of **Solution Explorer**, and then select **Set startup projects**.
   > 1. Ensure that all three projects are set to **Start** in the **Action** column.

1. Right-click the **ChainStore** project in **Solution Explorer**, and then select **Add** > **New Item**.
1. In the **Add New Item** dialog, select **Ribbon Custom Action**, name it **AddEmployeeToCorpDB**, and then select **Add**.
1. The dialog that opens asks three questions. Give the following answers:

   |                       **Question**                        | **Give this answer:** |
   | :-------------------------------------------------------- | :-------------------- |
   | **Where do you want to expose the custom action?**        | Host Web              |
   | **Where is the custom action scoped to?**                 | List Instance         |
   | **Which particular item is the custom action scoped to?** | Local Employees       |

1. Select **Next** and you get three more questions:

   |                    **Question**                    |                                                   **Give this answer:**                                                    |
   | :------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------- |
   | **Where is the control located?**                  | Ribbon.ListItem.Actions                                                                                                    |
   | **What is the label text for the button control?** | Add to Corporate DB                                                                                                        |
   | **Where does the button control navigate to?**     | ChainStoreWeb\Pages\EmployeeAdder.aspx<br/>(this is a page whose code-behind is going to add the employee to the database) |

1. Click  **Finish**.

   An elements.xml file that defines the custom action is added to the project and opened. For the most part, you can treat this file as a black box, and you won't need to make any changes in it until a later article in this series. For now, note only the following:

   - The  **Location** attribute of the **CommandUIDefinition** element has the value `Ribbon.ListItem.Actions.Controls_children`. The second part of this, `ListItem`, identifies the *tab* on the ribbon where the button will be placed (but that may not be the exact display name of the tab). The third part, `Actions`, is the name of the *section* of the ribbon where the button will be placed.
   - The  **CommandAction** attribute of the **CommandUIHandler** element begins with the placeholder `~remoteAppUrl`. This will be replaced with the URL of the remote web application when the button is deployed.
   - A few query parameters have been added to the **CommandAction** value with placeholder values in braces "{ }". These placeholders are resolved at runtime. Note that one of them is the ID of the list item that is selected by the user before she selects the custom button on the ribbon.

1. In the **ChainStoreWeb** project, open the **Pages/EmployeeAdder.aspx** file. Notice that it doesn't have any UI. The add-in is going to use this page as a kind of web service. This is possible because the ASP.NET **System.Web.UI.Page** class implements **System.Web.IHttpHandler** and because the **Page\_Load** event runs automatically when the page is requested.

1. Open the code-behind file **Pages/EmployeeAdder.aspx.cs**. The method that adds the employee to the remote database, `AddLocalEmployeeToCorpDB`, is already present. It uses the **SharePointContext** object to get the host web's URL, which the add-in uses as its tenant discriminator. The first thing the **Page_Load** method needs to do is initialize this object. The object is created and cached in the Session when the add-in's start page loads, so add the following code to the **Page_Load** method. (The **SharePointContext** object is defined in the SharePointContext.cs file that the Office Developer Tools for Visual Studio generates when the add-in solution is created.)

    ```csharp
    spContext = Session["SPContext"] as SharePointContext;
    ```

1. The `AddLocalEmployeeToCorpDB` method takes the employee's name as a parameter, so add the following line to the **Page_Load** method. You'll create the `GetLocalEmployeeName` method in a later step.

    ```csharp
    // Read from SharePoint
    string employeeName = GetLocalEmployeeName();
    ```

1. Under this line, add the call to the `AddLocalEmployeeToCorpDB` method.

    ```csharp
    // Write to remote database
    AddLocalEmployeeToCorpDB(employeeName);
    ```

1. Add a **using** statement to file for the namespace `Microsoft.SharePoint.Client`. (The Office Developer Tools for Visual Studio included the Microsoft.SharePoint.Client assembly in the **ChainStoreWeb** project when it was created.)
1. Now add the following method to the `EmployeeAdder` class. The SharePoint .NET Client-side Object Model (CSOM) is documented in detail elsewhere on MSDN, and we encourage you to explore it when you are finished with this series of articles. For this article, note that the **ListItem** class represents an item in a SharePoint list, and that the value of a field in the item can be referenced with "indexer" syntax. Also notice that the code refers to the field as **Title** even though you changed the field name to **Name**. This is because fields are always referred to in code by their *internal* name, not their display name. The internal name of a field is set when the field is created and can never change. You complete the `TODO1` in a later step.

    ```csharp
    private string GetLocalEmployeeName()
    {
      ListItem localEmployee;

      // TODO1: Initialize the localEmployee object by getting
      // the item from SharePoint.

      return localEmployee["Title"].ToString();
    }
    ```

1. Our code will need the list item's ID before it can retrieve it from SharePoint. Add the following declaration to the `EmployeeAdder` class just under the declaration for the `spContext` object.

    ```csharp
    private int listItemID;
    ```

1. Now add the following method to the `EmployeeAdder` class to get the list item's ID from the query parameter.

    ```csharp
    private int GetListItemIDFromQueryParameter()
    {
      int result;
      Int32.TryParse(Request.QueryString["SPListItemId"], out result);
      return result;
    }
    ```

1. To initialize the `listItemID` variable, add the following line to the **Page_Load** method just under the line that initializes the `spContext` variable.

    ```csharp
    listItemID = GetListItemIDFromQueryParameter();
    ```

1. In the `GetLocalEmployeeName`, replace the `TODO1` with the following code. For the time being, just treat this code as a black box while we concentrate on getting the custom button working. We'll learn more about this code in the next article in this series, which is all about the SharePoint client-side object model.

    ```csharp
    using (var clientContext = spContext.CreateUserClientContextForSPHost())
    {
      List localEmployeesList = clientContext.Web.Lists.GetByTitle("Local Employees");
      localEmployee = localEmployeesList.GetItemById(listItemID);
      clientContext.Load(localEmployee);
      clientContext.ExecuteQuery();
    }

    ```

   The entire method should now look like the following.

    ```csharp
    private string GetLocalEmployeeName()
    {
      ListItem localEmployee;

      using (var clientContext = spContext.CreateUserClientContextForSPHost())
      {
        List localEmployeesList = clientContext.Web.Lists.GetByTitle("Local Employees");
        selectedLocalEmployee = localEmployeesList.GetItemById(listItemID);
        clientContext.Load(selectedLocalEmployee);
        clientContext.ExecuteQuery();
      }
      return localEmployee["Title"].ToString();
    }
    ```

1. The EmployeeAdder page should not actually render, so add the following as the last line in the **Page_Load** method. This redirects the browser back to the list view page for the **Local Employees** list.

    ```csharp
    // Go back to the Local Employees page
    Response.Redirect(spContext.SPHostUrl.ToString() + "Lists/Local%20Employees/AllItems.aspx", true);
    ```

   The entire **Page_Load** method should now look like the following.

    ```csharp
    protected void Page_Load(object sender, EventArgs e)
    {
      spContext = Session["SPContext"] as SharePointContext;
      listItemID = GetListItemIDFromQueryParameter();

      // Read from SharePoint
      string employeeName = GetLocalEmployeeName();

      // Write to remote database
      AddLocalEmployeeToCorpDB(employeeName);

      // Go back to the preceding page
      Response.Redirect(spContext.SPHostUrl.ToString() + "Lists/Local%20Employees/AllItems.aspx", true);
    }
    ```

## Request permission to read the host web list

As you have seen, SharePoint prompts you to grant the add-in permissions to the host web when it is installed. You have been re-installing the add-in every time you select F5. So far, the add-in has only needed minimal permissions, but the `GetLocalEmployeeName` method requires permission to read the lists of the host website. The add-in uses its add-in manifest to tell SharePoint what permissions it needs. Follow these steps.

1. In **Solution Explorer**, open the AppManifest.xml file in the **ChainStore** project (the file is called AppManifest because add-ins used to be called "apps"). The manifest designer opens.
1. Open the **Permissions** tab, select the empty cell under the **Scope** column, and then select **List** from the drop-down.
1. In the **Permission** field, select **Read** from the drop-down.
1. Leave the **Properties** field empty and save the file. The **Permissions** tab should look similar to the following.

   *Figure 2. Permissions tab*

   ![The Permissions tab of the add-in manifest designer showing the Scope to List and the Permission to be Read.](../images/8dd2a25f-103a-42af-aa88-c8ec796315db.PNG)

## Run the add-in and test the button

1. Use the F5 key to deploy and run your add-in. Visual Studio hosts the remote web application in IIS Express and hosts the SQL database in SQL Express. It also makes a temporary installation of the add-in on your test SharePoint site and immediately runs the add-in. You are prompted to grant permissions to the add-in before its start page opens. This time the prompt has a drop-down where you select the list that the app needs to read as seen in the following screenshot.

    *Figure 3. SharePoint add-in permission prompt*

    ![The SharePoint add-in permission prompt with the list named Local Employees selected in a drop-down that is labeled "Let it read items in the list"](../images/84e8b42c-4800-4947-acbd-21c6f096f4ea.PNG)

1. Select **Local Employees** from the list, and then select **Trust it**.
1. When the add-in's start page opens, select **Back to Site** on the chrome control at the top.
1. From the website's home page, go to **Site Contents** > **Local Employees**. The list view page opens.
1. Add a few employees to the list. *Do not select the __Added to Corporate DB__ check box.*
1. On the ribbon, open the **Items** tab. In the **Actions** section of the tab is the custom button **Add to Corporate DB**.
1. Select an item in the list. The page and ribbon should look similar to the following.

   *Figure 4. Local Employees list*

   ![The Local Employees list. One item is highlighted. Above it is the ribbon, and a button named "Add To Corporate DB" is in the Actions section.](../images/797a5ceb-7291-4b62-8075-2bb6a1b8e8a1.PNG)

1. After selecting an item in the list, select **Add to Corporate DB**.
1. The page seems to reload because the **Page_Load** method of the EmployeeAdder page redirects back to it.
1. Use the browser's back button twice to go back to the add-in's start page.
1. Select **Show Employees**, and the list of employees will be populated with the employee that you added. It should look similar to the following:

   *Figure 5. Corporate employees list on the add-in start page*

   ![The corporate employees list on the add-in start page showing the same employee that was selected in the earlier step.](../images/4a300a4e-f479-4f63-b536-6315c5d9ba4d.PNG)

1. To end the debugging session, close the browser window or stop debugging in Visual Studio. Each time that you select F5, Visual Studio retracts the previous version of the add-in and installs the latest one.
1. You will work with this add-in and Visual Studio solution in other articles, and it's a good practice to retract the add-in one last time when you are done working with it for a while. Right-click the project in **Solution Explorer** and select **Retract**.

## Next steps

In the next article, we'll take a brief break from coding to [get a quick overview of the SharePoint client-side object model](get-a-quick-overview-of-the-sharepoint-object-model.md).
