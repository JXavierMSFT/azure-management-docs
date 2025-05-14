---
title: Share Quote Across Subscriptions with Azure Quota Groups
description: Learn how to share quota across Azure subscriptions with Azure Quota Groups to reduce the number of quota transactions.
author: yaya-5
ms.author: yaalanis
ms.topic: how-to
ms.date: 04/23/2025
---

# Azure Quota Groups

Azure Quota Groups allow you to share quota among a group of subscriptions, reducing the number of quota transactions. This feature elevates the quota construct from a subscription level to a Quota Group Azure Resource Management (ARM) object, enabling customers to self-manage their procured quota within a group without needing approvals.

## Key benefits

- **Quota sharing across subscriptions**: Share procured quotas within a group of subscriptions.  
- **Self-service management**: Distribute or reallocate unused quota without Microsoft intervention.  
- **Fewer support requests**: Avoid filing support tickets when reallocating quota or managing new subscriptions.  
- **Group quota requests**: Request quota at the group level and allocate it across subscriptions as needed.  


## Supported scenarios  
The transfer of unused quota between subscription(s) will be done via Quota Group object created. At the moment of creating a Quota group object the group limit will be set to 0. Group limit must be explicitly updated by customer either through quota transfer from sub in group or by group quota increase request. When deploying resources the quota check at runtime is done against the subscription quota.  

- Deallocation: Transfer unused quota from your subscriptions to Group Quota. 
- Allocation: Transfer quota from group to target subscriptions. 
- Submit Quota Group increase request for a given region and Virtual Machine (VM) family. Once your request is approved, transfer quota from group to target subscriptions. 
- Quota Group increase requests are subject to the same checks as subscription quota increase requests. If capacity is low, then the request is rejected. 

## Prerequisites

Before you can use the Quota Group feature, you must:  
- Register the `Microsoft.Quota` and `Microsoft.Compute` resource provider on all relevant subscriptions that will be added to Quota Group(s). For more information, see [Registering the Microsoft Quota resource provider](https://learn.microsoft.com/en-us/rest/api/quota/#registering-the-microsoft-quota-resource-provider)
- A Management Group (MG) is needed to create a Quota Group. Your group will inherit quota write and or read permissions from the Management Group. Subscriptions belonging to another MG can be added to the Quota Group.
- Certain permissions are required to create Quota Groups and to add subscriptions. For more information on which roles to assign, see [#permissions].   

## Limitations

- Available only for Enterprise Agreement and Internal subscriptions. 
- Supports IaaS compute resources only.  
- Available in public cloud regions only.  
- Management Group deletion will result in the loss of access to the Quota Group limit. To clear out the group limit, allocate cores to subscriptions, delete subscriptions, then the Quota Group object before deletion of Management Group. In the even that the MG is deleted, access your Quota Group limit by recreating the MG with the same ID as before.

## Quota Group is an ARM object

Quota Group is a global ARM object created under a Management Group to logically group subscriptions for quota management. While it is tied to the Management Group for permissions, it doesn't auto-sync subscription membership. This means you have full flexibility to include subscriptions from different Management Groups. Quota Groups are: 

- Quota Groups are created at the Management Group scope.  
- Quota Groups are inherit permissions from its parent Management Group.
- Quota Groups are designed as an orthogonal grouping mechanism. They're independent of subscription placement in the Management Group hierarchy.
- Subscription lists are not auto-synced from Management Groups, giving you flexibility to organize quotas separately from policy or role management.

The following diagram shows and existing MG hierarchy set up with *subscription 1* and *subscription 2* being part of *Management Group A*, and *subscription 3* being part of *Management Group B*. In this example, the customer chose to create all quota groups under the single *Management Group A*. 
 
 :::image type="content" source="./media/quota-groups/sample-management-group-quota-group-hierarchy.png" alt-text="Diagram of Management Group hierarchy with sample Quota Groups created under Management Group."::: 

## Recommended group setup

A single Quota Group object manages quotas across multiple regions and VM families. Design your quota group structure with access control in mind. Access is inherited from the Management Group, so consider creating a tiered Management Group structure to ensure proper role assignments.  

Example hierarchy:  
- Management Group A owns Quota Groups 1 & 2.
- Management Group B owns Quota Group 3.
- Each quota group may be used to manage different applications, departments, and or regions.
- When performing operations such as quota transfers or increase requests, actions are scoped to specific regions and VM families.

 :::image type="content" source="./media/quota-groups/sample-recommended-quota-group-setup.png" alt-text="Diagram of Management Group hierarchy with multiple Quota Groups created under Management Group.":::

## Permissions

Certain permissions are required to create Quota Groups and to add subscriptions. For more information, see [Assign Azure roles using Azure CLI](/azure/role-based-access-control/role-assignments-cli) or [Assign Azure roles using the Azure portal](/azure/role-based-access-control/role-assignments-portal).
- Assign the *GroupQuota Request Operator* role on the Management Group where the Quota Group will be created.
- Assign the *Quota Request Operator* role on all participating subscriptions to the relevant users or applications managing quota operations.
 
## Quota Group APIs

This section covers the supported Quota Group operations via API and portal.
<!-- What is the purpose here? Please write at least a sentance to introduce this subsection. Elaborate, but keep it to the point. Write in an active voice speaking to the customer. -->

Use [Quota Group APIs](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/quota/resource-manager/Microsoft.Quota/stable/2025-03-01/groupquota.json) to:  
- Create or delete a Quota Group.
- Add or remove subscriptions from a Quota Group.
- Transfer or deallocate unused quota from subscriptions to a Quota Group. 
- Submit a Quota Group limit increase request.
- Submit a support ticket via portal if Quota Group limit request is rejected
- View Group limit

## SDK sample links

Use the below links to download the latest supported SDKs for Quota Group operations
<!-- What is the purpose here? Please write at least a sentance to introduce this subsection. -->

- [Go](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/quota/armquota@v1.1.0)
- [Python](https://pypi.org/project/azure-mgmt-quota/2.0.0/)
- [Java](https://central.sonatype.com/artifact/com.azure.resourcemanager/azure-resourcemanager-quota/1.1.0)
- [.NET](https://www.nuget.org/packages/Azure.ResourceManager.Quota/1.1.0#readme-body-tab)
- [JS](https://www.npmjs.com/package/@azure/arm-quota/v/1.1.0)

## Create a Quota Group
- Create a Quota Group object to be able to do quota transfers between subscriptions (step on how to add subcriptions is below) and submit Quota Group increase requests.
- Requires the *GroupQuota Request Operator* role on the Management Group used to create Quota group
<!-- Please write at least a sentance to introduce this subsection. -->

### [REST API](#tab/rest-1)
To create a Quota Group using the REST API, make a `PUT` request to the following endpoint:

```http
PUT https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}?api-version=2025-03-01 
```

Request body:
```json
{
  "properties": {
    "displayName": "allocationGroupTest"
  }
}
```
Example using `az rest`:
Sample response: 
```json
az rest --method put --url https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}?api-version=2025-03-01 --body '{
  "properties": {
    "displayName": "allocationGroupTest"
  }
}'
{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}",
  "name": "{groupquota}",
  "properties": {
    "provisioningState": "ACCEPTED"
  },
  "type": "Microsoft.Quota/groupQuotas"
}
```

### [Azure portal](#tab/portal-1)
Create a Quota Group through the Azure portal.

1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.
2. Under settings in left hand side select **Quota groups**.
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group
4. Select **Create** button to go to Create Quota Group view
5. In the **Basics** tab, fill out required fields such name of Quota group. Quota Group name cannot contain any special characters and or spaces.
6. Select **Management group**, if the default Management group is not the desired one, select **Change management group**, then select **Next** button.
7. In the **Subscription selection** tab select subscription(s) to be added to quota group. Subscription will be greyed out and unable to be selected if it already belongs to existing Quota Group, then  select **Next** button.
8. In **Review + create** tab review details of Quota group before creation. Under **Basics** name of Quota Group will be displayed, under **Group selection** the selected Management group and subscription(s) will be displayed
--- 

## Delete a Quota Group
To delete a Quota Group
- Create a Quota Group object to be able to do quota transfers between subscriptions (step on how to add subcriptions is below) and submit Quota Group increase requests.
- Requires the *GroupQuota Request Operator* role on the Management Group used to DELETE a Quota group
- All subscription(s) must be removed from Quota Group before deletion  
### [REST API](#tab/rest-2)

```http
DELETE https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupQuotaName}?api-version=2025-03-01
```

### [Azure portal](#tab/portal-2)

To delete a Quota Group through the Azure portal 
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.
2. Under settings in left hand side select **Quota groups**.
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group
4. Select the box next tto the quota group to be deleted
5. Select the **Delete**  button on the top
6. In the **Delete a quota group** view the Quota Group to be deleted will be shown, along with the list of **Dependent subscription to be removed from quota group**
7. Select the **Delete** button on the bottom, **Delete confirmation** notification will pop up notifying **Deleting this quota group is a permanent actino and cannot be undone**, by selecting delete you are giving permission for Azure to remove your subscription(s) if there are any from group and then group will be deleted
8. Notification on right hand side wil surface with **Your quota group has been deleted** once group has been deleted
--- 
<!-- The following is an example of the format we use to write out portal instructions. Be clear, concise, only bold or italicise actual names of blades and options, etc. Start instructions from the start of openning the portal.
1. Go to **Virtual machine scale sets**.
2. Select the **Create** button to go to the **Create a virtual machine scale set** view.
3. In the **Basics** tab, fill out the required fields. If the field isn't called out in the next sections, you can set the fields to what works best for your scale set.
4. Ensure that you select a region that instance mix is supported in.
5. Be sure **Orchestration mode** is set to **Flexible**.
6. In the **Size** section, click **Select up to 5 sizes** and the **Select a VM size** page appears.
7. Use the size picker to select up to five VM sizes. Once you select your VM sizes, click the **Select** button at the bottom of the page to return to the scale set Basics tab.
8. In the **Allocation strategy** field, select your allocation strategy.
9. Using the `Prioritized (preview)` allocation strategy, the **Rank size** section appears below the Allocation strategy section. Clicking on the bottom **Rank priority** brings up the prioritization blade, where you can adjust the priority of your VM sizes.
10. You can specify other properties in subsequent tabs, or you can go to **Review + create** and select the **Create** button at the bottom of the page to start your instance mix scale set deployment.
-->


<!-- Keep the 3 dashes above this line. That indicates the end of a tabbed section. Remove this note after portal steps are added. -->


## Add or remove subscriptions from a Quota Group
This section covers how to add subscription(s) after the Quota group is created. Subscription(s) will carry their existing quota and usage, this will not be manipulated at the moment of adding to group. Subscription quota remains separate from the group limit. 

<!-- Please write at least a sentance to introduce this subsection. -->
<!-- Consider breaking add and remove into their own seperate sections. -->

### [REST API](#tab/rest-3)
To add subscriptions from the Quota Group using the REST API, make a `PUT` request to the following endpoint:

```http
PUT https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/subscriptions/{subscriptionId}?api-version=2025-03-01
```

This section covers how to remove subscriptions from the Quota Group. At the moment of removal, subscription will have carry its existing quota and usage. The group limit will not be manipulated based on subscription removal.  

To remove subscriptions from the Quota Group using the REST API, make a `DELETE` request to the following endpoint:

```http
DELETE https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/subscriptions/{subscriptionId}?api-version=2025-03-01

```

### [Azure portal](#tab/portal-3)
Add or remove subscriptions from the Quota Group through the Azure portal. Subscriptions can only belong to one quota group at a time.

To add subscriptions from Quota Group through portal.
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.
2. Under settings in left hand side select **Quota groups**.
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group
4. Select **Edit** or **Add** button under **Subscriptions added** column
5. In the Edit subscriptions view the exisiting subscription(s) will be listed, select **Add subscription** button
6. In the Add Subscriptions view select the desired subscription and select **Save**. You can search for subscription in search box at the top of blade. Subscription(s) will be greyed out if they belong to existing group and indicate 'No' under **Available to add** column.
7. Notification should indicate that subscription(s) was successfully added and the Edit Subscriptions view will be updated with the added subscription(s)
   
To remove subscription from Quota Group through portal. 
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.
2. Under settings in left hand side select **Quota groups**.
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group
4. Select **Edit** or **Add** button under **Subscriptions added** column
5. In the Edit subscriptions view the exisiting subscription(s) will be listed, select trashcan icon for the subscription you'd like to remove
6. There will be prompt confirming whether you're sure about removing the selected subscription, select **remove**
7. Notification should indicate that subscription(s) was successfully removed and the Edit Subscriptions view will be updated with the latest list of subscription(s)

--- 
<!-- Keep the 3 dashes above this line. That indicates the end of a tabbed section. Remove this note after portal steps are added. -->
## Transfer quota within Quota Group
### [REST API](#tab/rest-4)
- Transfer unused quota from your subscription to a Quota Group or from a Quota Group to a subscription.
- Once your quota group is created and subscription(s) have been added, you can transfer quota between subscription(s) by deallocating/transfering quota from source subscription to group, then allocating/transferring quota from group to target subscription for a given region and VM family.
- To allocate or transfer quota from group to target subscription, update subID to target subscription, then set the limit property to the new desired subscription limit. If your current subscription quota is 10 and you want to transfer 10 cores from group to target subscription, set the new limit to 20.
- You can view quota allocation snapshot for subscription in Quota Group or view group limit to validate transfer and stamping of cores at group level
- To view your existing subscription usage for a given region, please use the [Compute Usages API](https://learn.microsoft.com/en-us/rest/api/compute/usage/list?view=rest-compute-2023-07-01&tabs=HTTP&tryIt=true&source=docs#code-try-0).

```http
PATCH https://management.azure.com/"providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/{location}?api-version=2025-03-01"
```
```json
{
  "properties": {
    "value": [{
        "properties": {
          "limit": 50,
          "resourceName": "standardddv4family"
        }
      }]
  }
}
```

Example using `az rest`: 
 I transfer 10 cores  of *standarddv4family* in centralus from subscription to group by setting limit to 50


```json
az rest –method patch –url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/{location}?api-version=2025-03-01" –body ‘{
  "properties": {
    "value": [
      {
        "properties": {
          "limit": 50,
          "resourceName": "standardddv4family"
        }
      }
    ]
  }
}’ –debug
```
### View quota allocation snapshot for subscription in Quota Group
- Limit = current subscription limit  
- Shareable quota = how many cores have been deallocated/transferred from sub to group  ‘-5’ = 5 cores were given from sub to group  

```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/{location}?api-version=2025-03-01
```

Example using `az rest`:
My new subscription limit is 50 cores for *standarddv4family* in centralus and my shareable quota is -10 because I gave 10 cores to my Quota group. 
```json
az rest --method get --url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/075216c4-f88b-4a82-b9f8-cdebf9cc097a/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/eastus?api-version=2025-03-01&\$filter=resourceName eq 'standardddv4family'" --debug
Response content

{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/075216c4-f88b-4a82-b9f8-cdebf9cc097a/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/eastus",
  "name": "eastus",
  "provisioningState": "Succeeded",
  "type": "Microsoft.Quota/groupQuotas/quotaAllocations",
  "value": [
    {
      "properties": {
        "limit": 50,
        "name": {
          "localizedValue": "standardddv4family",
          "value": "standardddv4family"
        },
        "resourceName": "standardddv4family",
        "shareableQuota": -10
      }
    }
  ]
}

```

<!-- Portal steps on how to do quota transfer -->
### [Azure portal](#tab/portal-4)
Transfer unused quota between subscriptions via the Quota Group object. Below steps are how to do quota transfer from ssource subscription to group and from group to target subscription.

Transfer from source subscription to group. 
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.  
2. Under settings in left hand side select **Quota groups**.  
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group  
4. Select **Quota group** from list of Quota Group(s)  
5. In the Quota Group resoruces view there will be the list of Quota group resources by region by Group quota (limit)  
6. Use the filters to select **Region** and or **VM Family**, you can also search for region and or VM family in the search bar  
7. Select the **Quota group resource** link to view the Quota Group resources details view for selected resource, view your **Group Quota** also known as group limit, **Total current usage / limit** also know as the sum of subscription(s) usage over sum of subscription quota. Lastly view list of subscription(s) in selected group -> Quota Group resource  
8. Select from the list of subscription(s) and select **Manage subscription quota** button  
9. Select **Return quota to family limit** option, the **Increase subscription quota** option will be greyed out if Group quota for selected   resource = 0  
10. In the **Distribute** blade you can view your **Group quota** also known as group limit and the **Manage subscription quota** drop down, ensure **Return quota to group quota** is selected if you want to transfer unused quota from source subscription to group.  
11. View the list of subscription(s) by **Current usage / limit** and **Return to group quota column**, input value of quota you'd like to transfer from source subscription to group. Output message **Your new quota limit will be** will indicate the new subscription quota if you complete transfer. You cannot insert value above the current subscription quota or above the value of (subsription quota - usage)  
12. Select **Next** button to view the **Review + Distribute** page. You can view your **New group quota** value if you submit quota transfer, you can also view list of subscription selected by **New usage / limit** with the value inputted in previous step. The **Returned to group quota** column indicates the quota being moved from subscription to group.  
13. Select **Submit** button to trigger quota transfer, notification **We are reviewing your request to adjust quota** on right hand side will surface. Quota transfer may take up to ~3 minutes to complete.  
14. Once completed notification **Your quota has been adjusted** with subscription name and new subscription limit value will surface on right hand side.  
15. Select the Quota group resource / VM family in breadcrumb **Home -> Quota |Quota group -> QuotaGroupName -> Quota group resource / VM family** to view updated Group quota  

Transfer from group to target subscripion
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.  
2. Under settings in left hand side select **Quota groups**.  
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group  
4. Select **Quota group** from list of Quota Group(s)  
5. In the Quota Group resoruces view there will be the list of Quota group resources by region by Group quota (limit)  
6. Use the filters to select **Region** and or **VM Family**, you can also search for region and or VM family in the search bar  
7. Select the **Quota group resource** link to view the Quota Group resources details view for selected resource, view your **Group Quota** also known as group limit, **Total current usage / limit** also know as the sum of subscription(s) usage over sum of subscription quota. Lastly view list of subscription(s) in selected group -> Quota Group resource
8. Select from the list of subscription(s) and select **Manage subscription quota** button
9. Select **Increase subscription quota** option, this will be geyed out if Group quota for selected resource = 0
10. In the **Distribute** blade you can view your **Group quota** also known as group limit and the **Manage subscription quota** drop down, ensure **Increase subscription quota** is selected if you want to transfer unused quota from group to target subscription
11. View the list of subscription(s) by **Current usage / limit** and **Distribute to subscription quota**, input value of quota you'd like to transfer from group to target subscription. Output message **Your new quota limit will be** will indicate the new subscription quota if you complete transfer. You cannot insert value above the group quota limit or 0.
12. Select **Next** button to view the **Review + Distribute** page. You can view your **New group quota** value if you submit quota transfer, you can also view list of subscription selected by **New usage / limit** with the value inputted in previous step. The **Distribute to subscription quota** column indicates the quota being moved from group to subscription. 
13. Select **Submit** button to trigger quota transfer, notification **We are reviewing your request to adjust quota** on right hand side will surface. Quota transfer may take up to ~3 minutes to complete.  
14. Once completed notification **Your quota has been adjusted** with subscription name and new subscription limit value will surface on right hand side.
15. Select the Quota group resource / VM family in breadcrumb **Home -> Quota |Quota group -> QuotaGroupName -> Quota group resource / VM family** to view updated Group quota  
--- 

## Submit Quota Group Limit increase request
One of the key benefits of Quota Group offering is the ability to submit Quota Group Limit increase requests rather than at the per subscription level. If your group limit request is approved you can then follow steps to allocate/transfer quota from group to target subscription(s) for a given region x VM family.  
- Require *GroupQuota Request Operator* role on the Management Group to submit Quota Group limit increase request.
- Customers can submit Quota Group limit increase requests for a region x VM family combination, and if approved, quota will be stamped on the specified Quota GroupID.  
- Quota Group Limit increase requests undergo the same checks as subscription level requests. Value should be absolute value of the new desired amount.   
- If Quota Group  Limit request is rejected then customer must submit support ticket via the self-serve Quota group request blade.  
- Support tickets for Quota Groups will be created based on a preselected subscriptionID within the group, the customer has the ability to edit the subID when updating request details.

### [REST API](#tab/rest-5)
```http
PATCH https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/{location}?api-version=2025-03-01
```
```json
{
  "properties": {
    "value": [
      {
        "properties": {
          "resourceName": "standardddv4family",
          "limit": 50,
          "comment": "comments"
        }
      }
    ]
  }
}
```
Example using `az rest`
- I submit PATCH Quota Group limit increase request of 50 cores for ***standardddv4family*** in centralus
- Use the groupQuotaOperationsStatus ID in my reponse header to validate the status of request in next section

```http
az rest --method patch --uri "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01" --body '{"properties":{"value":[{"properties":{"resourceName":"standardddv4family","limit":20,"comment":"comments"}}]}}' --verbose
```


```json
{
  "properties": {
    "value": [
      {
        "properties": {
          "resourceName": "standardddv4family",
          "limit": 50,
          "comment": "comments"
        }
      }
    ]
  }
}
```
Example response Quota Group increase request
```json
Status code: 202
Response header:
Location: https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaOperationsStatus/6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2?api-version=2025-03-01
Retry-After: 30 
Response Content
```

## GET groupQuotaLimit request status
Since groupQuotaLimit request is async operation, capture status of request using groupQuotaOperationsStatus ID from response header when submitting limit increase request
```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaRequests/{requestId}?api-version=2025-03-01
```


Example using `az rest`
I used the groupQuotaOperationsStatus ID from my PATCH Quota Group Limit increase request of 50 cores for ***standardddv4family*** in centralus succeeded
```http
az rest --method get --uri "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaRequests/6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2?api-version=2025-03-01"
```
Sample response of approved Quota Group increase request
```json
{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaRequests/6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2",
  "name": "6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2",
  "properties": {
    "provisioningState": "Succeeded",
    "requestProperties": {
      "requestSubmitTime": "2025-05-06T17:57:00.0001431+00:00"
    },
    "requestedResource": {
      "properties": {
        "limit": 20,
        "name": {
          "localizedValue": "STANDARDDDV4FAMILY",
          "value": "STANDARDDDV4FAMILY"
        },
        "provisioningState": "Succeeded",
        "region": "centralus"
      }
    }
  },
  "type": "Microsoft.Quota/groupQuotas/groupQuotaRequests"
}
```
### GET groupQuotaLimits  

Validate that the correct amount of cores were transferred from source subcription to group or that your group limit increase request was approved. Consider the below when interpreting the API response. 
- Available limit = how many  cores do I have at group level to distribute  
- Limit = how many cores have been explicitly requested and approved/stamped on your group via quota increase requests  
- Quota allocated = how many cores the sub has been allocated from group, ‘-‘ value indicates cores have been de-allocated from sub to group 

```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/{location}?api-version=2025-03-01&$filter=resourceName eq standarddv4family" -verbose
```
Example using `az rest`:
```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/{location}?api-version=2025-03-01&$filter=resourceName eq standarddv4family" -verbose
```

Example using `az rest`:
I do a GET group limit for my quota group in centralus.  
- For the resource standardddv4family my **availableLimit** = 50 cores which matches the amount of cores I requested and got approved at the group level
- The **Limit** = 40 because even though I submitted an increase for 50 I already had 10 cores at the group level from quota transfer example, and Azure only stamped an additional 40 cores 
- The **quotaAllocated** = -10 because I de-allocated /transfered 10 cores from source sub to group from previous section
```http
az rest --method get --url https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01&$filter=resourceName eq standardddv4family

```

Az rest sample reponse:

```json
az rest --method get --url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01&$filter=resourceName eq 'standardddv4family'"
{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/eastus",
  "name": "eastus",
  "properties": {
    "nextLink": "",
    "provisioningState": "Succeeded",
        "value": [
      {
        "properties": {
          "allocatedToSubscriptions": {
            "value": [
              {
                "quotaAllocated": -10,
                "subscriptionId": "226818a0-4fa2-4c2d-be7f-03b9b92ab3a2"
              }
            ]
          },
          "availableLimit": 50,
          "limit": 40,
          "name": {
            "localizedValue": "standardddv4family",
            "value": "standardddv4family"
          },
          "resourceName": "standardddv4family",
          "unit": "Count"
        }
      }
    ]
  },
  "type": "Microsoft.Quota/groupQuotas/groupQuotaLimits"
}

```
--- 

## Submit Quota Group Limit increase and file support ticket if request fails. 
The below covers how to submit Quota Group Limit increase via portal and file support ticket if request fails. 
- If Quota Group  Limit request is rejected via API or porta; then customer must submit support ticket via the self-serve Quota group request portal blade.  
- Support tickets for Quota Groups will be created based on a preselected subscriptionID within the group, the customer has the ability to edit the subID when updating request details. Even though ticket is created using subID, if approved the quota will be stamped at the group level. 
### [Azure portal](#tab/portal-5)  
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.
2. Under settings in left hand side select **Quota groups**.
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group
4. Select **Quota group** from list of Quota Group(s)
5. In the Quota Group resources view there will be the list of Quota group resources by region by Group quota (limit)
6. Use the filters to select **Region** and or **VM Family**, you can also search for region and or VM family in the search bar
7. Select the checkbox to the desired **Quota group resource**, then select the **Increase group quota** button at the top of page
8. On right side view the **New quota request** blade with selected region(s) at the top with details on the selected **Quota group resource**, the **Current group quota** value, and under **New group quota** column enter the  absolute value of desired net new group limit.I.e., If I want 20 cores assigned at group for DSv3 in Australia Central I will enter 20 under **New group quota**
9. Select **Submit** button, notification **We are reviewing your request to adjust the quota** this may take up to ~3 minutes to complete
10. If succesful the **New quota request** view will show the selected **Quota Group resource** by location status of request, the **Increase** value and **New limit**
11. Refresh the Quota Group resources view to view latest **Group quota** / group limit
12. If Quota Group limit increase was rejected notification **We were unable to adjust your quota** will surface
13. Select the **Generate a support ticket** button to start process of creating support ticket
14. In the **Request details** view select **Deployment model** as **Resource Manager** from drop drown, Under **Choose request types** select **Increase quota / expand access region**
15. Request details view will surface the Management GroupID and Group Quota ID, select desired regions(s) from **Locations** drop down adnd select desired VM family from **Quotas** drop down, review the selected VM family **Usage** and new **Limit**, ensure desired value is under **New Limit**, then select **Save and Continue** button
16. In **Additional details** view select required options **Advance diagnostic information** and **Preferred contact method** and select **Next**
17. Review details in **Review + Create** view and select **Create** button, notification **New Support Request** in top right corner will ticketID and link
18. To view request details return to **Quotas** blade and select the **Request** tab under the **Overview** page, see the list of quota requests, you may also search and go to **Help + Support** blade and view request under **Recent support requests** table
--- 


--- 
<!-- OLD
<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->


<!-- OLD
<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->



<!--The below covers end-to-end flow of how to do quota transfer from source subscription to group via API. 
1. Add subscription to Group
2. Verify current subscription quota and usage by doing GET via Usages API (LINK TO Usages REST API); user can also do GET quotaAllocations to view current subscription quota but usage wont be provided
3. Submit quotaAllocation request for a given region and VM family. When submitting quota allocation request the **Limit** should be set as absolute value of the new desired subscription limit. If you want to transfer 10 cores of *standarddv4family* to your group and your current subscription limit is 120, set the new limit to 110.
4. GET quotaAllocationRequests status to view status of request once provising state = Succeeded" then move to below step
5. GET quotaAllocations to view current subscription quota and how many cores have been transferred to group
   	- Limit = current subscription limit  
	- Shareable quota = how many cores have been deallocated/transferred from sub to group
	- ‘-5’ = 5 cores were given from sub to group
6. GET GroupLimit to view current group limit

### Step 1 - Add subscription to group (link to previous section)  
### Step 2 - Verify current subscription quota and usage via Usages API (LINK TO Usages REST API)  
### GET subscription quota and usage
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.Compute/locations/{location}/usages?api-version=2023-07-01
```
Example:  I have 60 cores for *standarddv4family* on subscription that's been added to Quota Group 
Sample response
```json
    {
      "limit": 60,
      "unit": "Count",
      "currentValue": 0,
      "name": {
        "value": "standardDDv4Family",
        "localizedValue": "Standard DDv4 Family vCPUs"
      }
    }
```

### Step 3 - Submit quotaAllocation request for a given region and VM family
### PATCH subscription quota allocation

<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->
<!--To deallocate or transfer quota from source subscription to group, set the limit property as absolute value to the new desired subscription limit. If you want to transfer 10 cores of *standarddv4family* to your group and your current subscription limit is 60 cores, set the new limit to 50.  

To allocate or transfer quota from group to target subscription, set the limit property to the new desired subscription limit. If your current subscription quota is 10 and you want to transfer 10 cores from group to target subscription, set the new limit to 20.  

```http
PATCH https://management.azure.com/"providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/{location}?api-version=2025-03-01"
```
```json
{
  "properties": {
    "value": [{
        "properties": {
          "limit": 50,
          "resourceName": "standardddv4family"
        }
      }]
  }
}
```

Example using `az rest`: 
 I transfer 10 cores  of *standarddv4family* in centralus from subscription in previous step to group by setting limit to 50


```json
az rest –method patch –url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/{location}?api-version=2025-03-01" –body ‘{
  "properties": {
    "value": [
      {
        "properties": {
          "limit": 50,
          "resourceName": "standardddv4family"
        }
      }
    ]
  }
}’ –debug
```
### Step 4- GET quotaAllocationRequests status to view status of request once provising state = Succeeded" then move to below step
### GET subscription allocation quota request status 
The quotaAllocations request is an async operation that may take up to 3 minutes to complete, to view the status of the request you can do below GET call using the allocationID in response header from previous step.
<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->

<!--- If allocation request status is *succeeded*, then GET subscription `quotaAllocations` to view current subscription limit for region x SKU.
	- Accepted
	- In progress
	- Succeeded
	- In progress
	- Escalated
	- Failed


```http
GET /providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/quotaAllocationRequests/{allocationId}?api-version=2025-03-01
```

```json
Status code: 202
Response header:
Location: https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/075216c4-f88b-4a82-b9f8-cdebf9cc097a/providers/Microsoft.Quota/groupQuotas/{groupquota}/quotaAllocationOperationsStatus/3b80d125-ada8-41c8-b6c4-12fae6a7f55b?api-version=2025-03-01
Retry-After: 30 
Response Content
```

Sample response: 

```json
{
  "id": "/providers/Microsoft.Management/managementGroups/yayatest092724/subscriptions/226818a0-4fa2-4c2d-be7f-03b9b92ab3a2/providers/Microsoft.Quota/groupQuotas/yayatestingmgissue/quotaAllocationRequests/b5d6f0aa-70e3-4178-8281-8e5e3f06e924",
  "name": "b5d6f0aa-70e3-4178-8281-8e5e3f06e924",
  "properties": {
    "provisioningState": "Succeeded",
    "requestProperties": {
      "requestSubmitTime": "2025-05-05T21:45:07.4047197+00:00"
    },
    "requestedResource": {
      "properties": {
        "limit": 50,
        "name": {
          "localizedValue": "STANDARDDDV4FAMILY",
          "value": "STANDARDDDV4FAMILY"
        },
        "provisioningState": "Succeeded",
        "region": "centralus"
      }
    }
  },
  "type": "Microsoft.Quota/groupQuotas/quotaAllocationRequests"

```
### Step 5 - GET quotaAllocations to view current subscription quota and how many cores have been transferred to group
### GET quotaAllocations
Once the quotaAllocations request has succeeded you can do a GET quotaAllocations request to view the updated subscription quota and how many cores (shareable quota) have been deallocated from the source subscription to the group. 
<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->

<!--- View current subscription limit for region x SKU  
- Limit = current subscription limit  
- Shareable quota = how many cores have been deallocated/transferred from sub to group  ‘-5’ = 5 cores were given from sub to group  

```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/{subscriptionId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/{location}?api-version=2025-03-01
```

Example using `az rest`:
My new subscription limit is 50 cores for *standarddv4family* in centralus and my shareable quota is -10 because I gave 10 cores to my Quota group. 
```json
az rest --method get --url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/075216c4-f88b-4a82-b9f8-cdebf9cc097a/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/eastus?api-version=2025-03-01&\$filter=resourceName eq 'standardddv4family'" --debug
Response content

{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/subscriptions/075216c4-f88b-4a82-b9f8-cdebf9cc097a/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/quotaAllocations/eastus",
  "name": "eastus",
  "provisioningState": "Succeeded",
  "type": "Microsoft.Quota/groupQuotas/quotaAllocations",
  "value": [
    {
      "properties": {
        "limit": 50,
        "name": {
          "localizedValue": "standardddv4family",
          "value": "standardddv4family"
        },
        "resourceName": "standardddv4family",
        "shareableQuota": -10
      }
    }
  ]
}
```
### Step 6 - GET GroupLimit to view current group limit  
### GET groupQuotaLimits  
The below is the final step to validate that the correct amount of cores were transferred from source subcription to group. Consider the below when interpreting the API response. 
- Available limit = how many  cores do I have at group level to distribute  
- Limit = how many cores have been explicitly requested and approved/stamped on your group via quota increase requests  
- Quota allocated = how many cores the sub has been allocated from group, ‘-‘ value indicates cores have been de-allocated from sub to group 

```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/{location}?api-version=2025-03-01&$filter=resourceName eq standarddv4family" -verbose
```

Example using `az rest`:
I do a GET group limit for my quota group in centralus.  
- For the resource standardddv4family my **availableLimit** = 10 cores which matches the amount of cores I transferred in step 3 from source sub.  
- The **Limit** = 0 because I havent gotten any cores approved/stamped at the group level. Steps on how to submit Quota group increase request is in next section.  
- The **quotaAllocated** = -10 because I de-allocated /transfered 10 cores from source sub to group.    

- For the resource standarddsv3family my **availableLimit** = 5  
- The **Limit** = 10 because I submitted Quota Group increase request and was approved.  
- The **quotaAllocated** = 5 because I allocated / transferred 5 cores from group to target sub.  



```http
az rest --method get --url https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01&$filter=resourceName eq standardDSv3Family


```

Sample reponse:

```json
az rest --method get --url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/eastus?api-version=2025-03-01&$filter=resourceName eq 'standardddv4family'"
{
  "id": "/providers/Microsoft.Management/managementGroups/yayatest092724/providers/Microsoft.Quota/groupQuotas/yayatestingmgissue/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus",
  "name": "centralus",
  "properties": {
    "nextLink": "",
    "provisioningState": "Succeeded",
    "value": [
      {
        "properties": {
          "allocatedToSubscriptions": {
            "value": [
              {
                "quotaAllocated": 5,
                "subscriptionId": "226818a0-4fa2-4c2d-be7f-03b9b92ab3a2"
              }
            ]
          },
          "availableLimit": 5,
          "limit": 10,
          "name": {
            "localizedValue": "standarddsv3family",
            "value": "standarddsv3family"
          },
          "resourceName": "standarddsv3family",
          "unit": "Count"
        }
      },
      {
        "properties": {
          "allocatedToSubscriptions": {
            "value": [
              {
                "quotaAllocated": -10,
                "subscriptionId": "226818a0-4fa2-4c2d-be7f-03b9b92ab3a2"
              }
            ]
          },
          "availableLimit": 10,
          "limit": 0,
          "name": {
            "localizedValue": "standardddv4family",
            "value": "standardddv4family"
          },
          "resourceName": "standardddv4family",
          "unit": "Count"
        }
      }
    ]
  },
  "type": "Microsoft.Quota/groupQuotas/groupQuotaLimits"
}

```



### PATCH Quota Group Limit increase request

```http
PATCH https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/{location}?api-version=2025-03-01
```
Example using `az rest`
- I submit PATCH Quota Group limit increase request of 50 cores for ***standardddv4family*** in centralus
- Use the groupQuotaOperationsStatus ID in my reponse header to validate the status of request in next section

```http
az rest --method patch --uri "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01" --body '{"properties":{"value":[{"properties":{"resourceName":"standardddv4family","limit":20,"comment":"comments"}}]}}' --verbose
```


```json
{
  "properties": {
    "value": [
      {
        "properties": {
          "resourceName": "standardddv4family",
          "limit": 50,
          "comment": "comments"
        }
      }
    ]
  }
}
```
Example response Quota Group increase request
```json
Status code: 202
Response header:
Location: https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaOperationsStatus/6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2?api-version=2025-03-01
Retry-After: 30 
Response Content
```



## GET Quota Group Limit increase request status
This covers how to view the status of your Quota Group Limit inrease request, if request shows as "Escalated" you must submit support ticket via portal
<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->
<!-- OLD- A Quota Group Limit increase will take up to 3 minutes to be processed by Azure, check the status of your request with the below.

### GET Quota Group Limit increase request status
```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaRequests/{requestId}?api-version=2025-03-01
```
Example using `az rest`
I used the groupQuotaOperationsStatus ID from my PATCH Quota Group Limit increase request of 50 cores for ***standardddv4family*** in centralus succeeded
```http
az rest --method get --uri "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaRequests/6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2?api-version=2025-03-01"
Sample response:
```
Sample response of approved Quota Group increase request
```json
{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/groupQuotaRequests/6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2",
  "name": "6c1cdfb8-d1ba-4ade-8a5f-2496f0845ce2",
  "properties": {
    "provisioningState": "Succeeded",
    "requestProperties": {
      "requestSubmitTime": "2025-05-06T17:57:00.0001431+00:00"
    },
    "requestedResource": {
      "properties": {
        "limit": 20,
        "name": {
          "localizedValue": "STANDARDDDV4FAMILY",
          "value": "STANDARDDDV4FAMILY"
        },
        "provisioningState": "Succeeded",
        "region": "centralus"
      }
    }
  },
  "type": "Microsoft.Quota/groupQuotas/groupQuotaRequests"
}
```
Sample response of rejected Quota Group increase request, "Escalated" means that your request has been rejected and you must submit support ticket via portal. 
```json

 "properties": {
    "provisioningState": "Escalated",
    "requestProperties": {
      "requestSubmitTime": "2025-05-12T20:03:12.1551297+00:00"
    },
    "requestedResource": {
      "properties": {
        "limit": 5000,
        "name": {
          "localizedValue": "STANDARDDDV4FAMILY",
          "value": "STANDARDDDV4FAMILY"
        },
        "provisioningState": "Escalated",
        "region": "centralus"
      }
    }
  },
  "type": "Microsoft.Quota/groupQuotas/groupQuotaRequests"
}
```

## Get Quota Group limit 

<!-- Please write clearer instructional content, preferably step-by-step, as with previous sections. Write full sentences, even on bulleted lists. Or mention specific properties that need to be adjusted, and be very explicit about those details. If you need REST and portal tabs, copy the format from previous sections. -->

<!-- OLDThe below is the final step to validate that the correct amount of cores were stamped on the group once I validate that my Quota Group Limit increase request was succesfully approved. Consider the below when interpreting the API response. 
- Available limit = how many  cores do I have at group level to distribute  
- Limit = how many cores have been explicitly requested and approved/stamped on your group via quota increase requests  
- Quota allocated = how many cores the sub has been allocated from group, ‘-‘ value indicates cores have been de-allocated from sub to group 


```http
GET https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/{location}?api-version=2025-03-01&$filter=resourceName eq standarddv4family" -verbose
```

Example using `az rest`:
I do a GET group limit for my quota group in centralus.  
- For the resource standardddv4family my **availableLimit** = 50 cores which matches the amount of cores I requested and got approved at the group level
- The **Limit** = 40 because even though I submitted an increase for 50 I already had 10 cores at the group level from quota transfer example, and Azure only stamped an additional 40 ores 
- The **quotaAllocated** = -10 because I de-allocated /transfered 10 cores from source sub to group from previous section
```http
az rest --method get --url https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01&$filter=resourceName eq standardddv4family

```

Sample reponse:

```json
az rest --method get --url "https://management.azure.com/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/centralus?api-version=2025-03-01&$filter=resourceName eq 'standardddv4family'"
{
  "id": "/providers/Microsoft.Management/managementGroups/{managementGroupId}/providers/Microsoft.Quota/groupQuotas/{groupquota}/resourceProviders/Microsoft.Compute/groupQuotaLimits/eastus",
  "name": "eastus",
  "properties": {
    "nextLink": "",
    "provisioningState": "Succeeded",
        "value": [
      {
        "properties": {
          "allocatedToSubscriptions": {
            "value": [
              {
                "quotaAllocated": -10,
                "subscriptionId": "226818a0-4fa2-4c2d-be7f-03b9b92ab3a2"
              }
            ]
          },
          "availableLimit": 50,
          "limit": 40,
          "name": {
            "localizedValue": "standardddv4family",
            "value": "standardddv4family"
          },
          "resourceName": "standardddv4family",
          "unit": "Count"
        }
      }
    ]
  },
  "type": "Microsoft.Quota/groupQuotas/groupQuotaLimits"
}

```
--- 



<!-- Portal steps on how to do Quota Group Limit increase  -->
<!-- OLD## Submit Quota Group Limit increase and file support ticket if request fails. 
### [Azure portal](#tab/portal-5)

The below covers how to submit Quota Group Limit increase via portal and file support ticket if request fails. 
- If Quota Group  Limit request is rejected via API or porta; then customer must submit support ticket via the self-serve Quota group request portal blade.  
- Support tickets for Quota Groups will be created based on a preselected subscriptionID within the group, the customer has the ability to edit the subID when updating request details. Even though ticket is created using subID, if approved the quota will be stamped at the group level. 
  
1. To view the Quotas page, sign in to the Azure portal and enter "quotas" into the search box, then select **Quotas**.
2. Under settings in left hand side select **Quota groups**.
3. To view existing Quota group select **Management Group** filter and select management group used to create Quota Group
4. Select **Quota group** from list of Quota Group(s)
5. In the Quota Group resources view there will be the list of Quota group resources by region by Group quota (limit)
6. Use the filters to select **Region** and or **VM Family**, you can also search for region and or VM family in the search bar
7. Select the checkbox to the desired **Quota group resource**, then select the **Increase group quota** button at the top of page
8. On right side view the **New quota request** blade with selected region(s) at the top with details on the selected **Quota group resource**, the **Current group quota** value, and under **New group quota** column enter the  absolute value of desired net new group limit.I.e., If I want 20 cores assigned at group for DSv3 in Australia Central I will enter 20 under **New group quota**
9. Select **Submit** button, notification **We are reviewing your request to adjust the quota** this may take up to ~3 minutes to complete
10. If succesful the **New quota request** view will show the selected **Quota Group resource** by location status of request, the **Increase** value and **New limit**
11. Refresh the Quota Group resources view to view latest **Group quota** / group limit
12. If Quota Group limit increase was rejected notification **We were unable to adjust your quota** will surface
13. Select the **Generate a support ticket** button to start process of creating support ticket
14. In the **Request details** view select **Deployment model** as **Resource Manager** from drop drown, Under **Choose request types** select **Increase quota / expand access region**
15. Request details view will surface the Management GroupID and Group Quota ID, select desired regions(s) from **Locations** drop down adnd select desired VM family from **Quotas** drop down, review the selected VM family **Usage** and new **Limit**, ensure desired value is under **New Limit**, then select **Save and Continue** button
16. In **Additional details** view select required options **Advance diagnostic information** and **Preferred contact method** and select **Next**
17. Review details in **Review + Create** view and select **Create** button, notification **New Support Request** in top right corner will ticketID and link
18. To view request details return to **Quotas** blade and select the **Request** tab under the **Overview** page, see the list of quota requests, you may also search and go to **Help + Support** blade and view request under **Recent support requests** table



--- 
<!-- Keep the 3 dashes above this line. That indicates the end of a tabbed section. Remove this note after portal steps are added. -->


<!-- Optional: Steps to clean up resources - H2

Provide steps the user can take to clean up resources that
they might no longer need.

-->

## Next step -or- Related content

> [!div class="nextstepaction"]
> [Next sequential article title](link.md)

-or-

* [Related article title](link.md)
* [Related article title](link.md)
* [Related article title](link.md)
