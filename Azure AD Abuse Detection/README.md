# BARK Detections

These KQL queries are designed to find use of the abuses in the [BloodHound BARK](https://github.com/BloodHoundAD/BARK) toolkit in your Azure AD tenant. These queries are not designed to detect the use of BARK itself, just the behaviour that BARK simulates. BARK is a wrapper for native Microsoft tooling so the same abuses could be carried out with Postman, or even in the Azure portal itself.

As a defender I recommend using this toolkit against your own Azure AD tenant and validating the queries against your own data set. That way you can tune the queries to ensure they are accurate to your environment. Running this toolkit against your own environment can also help you harden your environment.

For each BARK function, a query is available to detect the event in your Azure AD logs in Microsoft Sentinel / Log Analytics. 

For some abuses remember that the actor could be a user (a user that has been compromised) or service principal (a service principal that has been compromised). The target can also be a user or service principal. For example, a compromised user with a lot of privilege could add privilege to other users. It could also add privilege to service principals. A compromised service principal could also add privilege to either users or to other service principals.

I have also included recommendations to control the abuse in your tenant. Again, these controls are not to stop the specific use of the BARK toolkit. Rather to prevent or enforce guardrails around the behavior itself.

## BARK Function - Get-AZRefreshTokenWithUsernamePassword // Get-MSGraphTokenWithClientCredentials // Get-AzureRMTokenWithClientCredentials

To retrieve a refresh token for Azure AD or Microsoft Graph, you need to sign into a tenant. Obviously any Azure AD tenant is going to have a lot of sign ins, so you can try to limit your query to log on events where the application being accessed is the Azure AD PowerShell Module, the Microsoft Graph PowerShell module, or where the UserAgent contains PowerShell. Of course, not all these signins are malicious, but depending on your environment they may be suspicious.

For this abuse, the actor can be either a user or a service principal.

### Detection Query (User as actor)

```kql
SigninLogs
| where AppDisplayName in~ ("Azure Active Directory PowerShell","Microsoft Graph PowerShell","Microsoft Azure PowerShell") or UserAgent contains "WindowsPowerShell"
| project TimeGenerated, AppDisplayName, UserPrincipalName, IPAddress, UserAgent
```

### Detection Query (Service principal as actor)

Service principal sign in logs are held in the AADServicePrincipalSignInLogs table, because they are non human sign ins there is a lot less data to work with. Anytime Microsoft Graph is accessed a sign in will occur so alerting on individual sign in events is not practical. Instead you should look to limit Graph access via Conditional Access.

### Control / Prevention

The Azure AD PowerShell and Microsoft Graph PowerShell modules are legitimate tools that are likely in use in your environment. To limit access you can only allow certain users or service principals to access the application, as outlined here - https://o365blog.com/post/limit-user-access/

More broadly, you should try to restrict access to Azure management interfaces via Conditional Access to only known devices (such as privileged workstations) - https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-condition-filters-for-devices 

## BARK function - Set-AZUserPassword // Reset-AZUserPassword

Password resets are common in any Azure AD environment, so you can potentially query on password resets that are invoked via a WindowsPowerShell user agent. 

Currently the resetPassword action can only be run under user context.

### Detection Query (User as actor)

```kql
AuditLogs
| where OperationName == "Reset user password"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend UserAgent = tostring(AdditionalDetails[0].value)
| where UserAgent contains "WindowsPowerShell"
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], Target, UserAgent
```

### Control / Prevention 
Use least privilege when assigning users or service principals to Azure AD roles that have access to reset password accounts. Restrict user access to Microsoft Graph endpoints that allow password reset API calls. Enforce MFA on accounts and educate users to alert on suspicious MFA requests.

## BARK function - New-AppRegSecret // Test-MGAddSecretToApp

A secret or certificate for an application in Azure AD is the equivalent of a password. With the client id, the id of your Azure AD tenant and the secret or certificate, you can authenticate as the application. You then inherit the permission that has been granted to it. It is important to keep track when secrets are generated onto applications as it can be a sign of compromise.

For example, a user or another service principal may have the Application Admin role or have Application.ReadWrite.all Microsoft Graph access. That means they can create a secret on any application in the tenant (even one with higher privilege than itself), authenticate as it, and assume the privilege.

For this abuse, the actor can be either a user or a service principal.

### Detection Query (User as actor)

```kql
AuditLogs
| where OperationName has "Update application – Certificates and secrets management"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Target Application Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Application ObjectId'] = tostring(TargetResources[0].id)
| where isnotempty(Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Target Application Name'], ['Target Application ObjectId']
```

### Detection Query (Service principal as actor)

```kql
AuditLogs
| where OperationName has "Update application – Certificates and secrets management"
| extend ['Service Principal Actor Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Service Principal Actor ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| extend ['Target Application Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Application ObjectId'] = tostring(TargetResources[0].id)
| where isnotempty( ['Service Principal Actor ObjectId'])
| project TimeGenerated, OperationName, ['Service Principal Actor Name'], ['Service Principal Actor ObjectId'], ['Target Application Name'], ['Target Application ObjectId']
```

### Control / Prevention
Access to add credentials to applications should be restricted to administrative staff only. The Application Administrator role is extremely privileged as it can create credentials for any application in your tenant. 

## BARK function - New-ServicePrincipalSecret // Test-MGAddSecretToSP 

Adding a credential to a Service Principal is similar to adding credentials to an application object. The event in our hunting is slightly different however. 

For this abuse, the actor can be either a user or a service principal.

### Detection Query (User as actor)

```kql
AuditLogs
| where OperationName == "Add service principal credentials"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| where isnotempty(Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Target Service Principal Name'], ['Target Service Principal ObjectId']
```

### Detection Query (Service principal as actor)

```kql
AuditLogs
| where OperationName == "Add service principal credentials"
| extend ['Service Principal Actor Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Service Principal Actor ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| where ['Service Principal Actor Name'] != "Managed Service Identity"
| where isnotempty( ['Service Principal Actor ObjectId'])
| project TimeGenerated, OperationName, ['Service Principal Actor Name'], ['Service Principal Actor ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId']
```

### Control / Prevention
Similar to applications, access to add credentials to service principals should be restricted to administrative staff only. The Application Administrator role is extremely privileged as it can create credentials for any application in your tenant. 

## BARK function - New-AppRoleAssignment // Test-MGAddSelfToMGAppRole

Adding an app role assignment to a service prinicipal is effectively granting it privilege. It could be privilege to your own internal APIs, or to Microsoft Graph. For instance, you may add the 'Mail.ReadWrite.All' app role to a service principal. This grants it full read/write access to all the mailboxes in your Office 365 tenant. 

For this abuse, the actor can be both a user and a service principal. 

### Detection Query (User as actor)

```kql
AuditLogs
| where OperationName == "Add app role assignment to service principal"
| where Result == "success"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend ['App Role Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Service Principal ObjectId'] = tostring(TargetResources[1].id)
| where isnotempty(Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['App Role Name']
```

### Detection Query (Service principal as actor)

```kql
AuditLogs
| where OperationName == "Add app role assignment to service principal"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| extend RoleAdded = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| where TargetResources[0].type == "ServicePrincipal"
| where isnotempty(['Actor Service Principal ObjectId'])
| extend ['Target Service Principal Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[6].newValue)))
| extend ['Target Service Principal ObjectId'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[5].newValue)))
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId'], RoleAdded
```

### Control / Prevention

Access to grant app role assignments should be limited to administrative users and service principals. Alerting should be configured for all app role assignments to ensure that they aren't malicious and the role assigned is fit for purpose.

## BARK function - New-TestAppReg 

New application objects can be created by users or by other service principals. If not expected, these can be a sign of persistence in your tenant and if granted enough privilege a sign of privilege escalation.

For this abuse, the actor can be either a user or a service principal.

### Detection Query (User as actor)

```kql
AuditLogs
| where OperationName == "Add application"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Application Created Name'] = tostring(TargetResources[0].displayName)
| extend ['Application Created ObjectId'] = tostring(TargetResources[0].id)
| where isnotempty(Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Application Created Name'], ['Application Created ObjectId']
```

### Detection Query (Service principal as actor)

```kql
AuditLogs
| where OperationName == "Add application"
| extend ['Application Created Name'] = tostring(TargetResources[0].displayName)
| extend ['Application Created ObjectId'] = tostring(TargetResources[0].id)
| extend ['Service Principal Actor ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| extend ['Service Principal Actor Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| where isnotempty( ['Service Principal Actor ObjectId'])
| project TimeGenerated, OperationName, ['Service Principal Actor Name'], ['Service Principal Actor ObjectId'], ['Application Created Name'], ['Application Created ObjectId']
```

### Control / Prevention 
Limit the ability to create applications in Azure AD and Microsoft Graph to those users and service prinicipals that require it. Enforce Conditional Access policies to access Azure management interfaces for both users and service principals.

## BARK function - New-TestSP 

A new service principal is created in your tenant a number of ways. If you create an application object in the Azure AD portal, an equivalent service principal object is created. If you install a third party app, into for example, Teams, then only a service principal object is created. They are also created for Managed Identities in Azure AD.

For this abuse, the actor can be either a user or a service principal.

### Detection Query (User as actor)

```kql
AuditLogs
| where OperationName == "Add service principal"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend ['Service Principal ObjectId'] = tostring(TargetResources[0].id)
| extend ['Service Principal Name'] = tostring(TargetResources[0].displayName)
| where isnotempty(Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Service Principal Name'], ['Service Principal ObjectId']
```

### Detection Query (Service principal as actor)

```kql
AuditLogs
| where OperationName == "Add service principal"
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Service Principal Created Name'] = tostring(TargetResources[0].displayName)
| extend ['Service Principal Created ObjectId'] = tostring(TargetResources[0].id)
| where ['Actor Service Principal Name'] != "Managed Service Identity"
| where isnotempty(['Actor Service Principal ObjectId'])
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Service Principal Created Name'], ['Service Principal Created ObjectId']
```

### Control / Prevention
The ability to add applications and service principals should be limited to administrative staff. Regular users should be blocked from adding applications to your tenant and the admin consent workflow should be configured - https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/configure-admin-consent-workflow

## BARK function - Test-MGAddSelfAsOwnerOfApp 

Adding owners to app registrations allows them to change settings of the application, for example they can update redirect URI addresses. Importantly an owner of an application object can generate new secrets for that application. They can then authenticate as it and assume the privilege it holds.

For this abuse, the actor can be either a user or a service principal. The target can also be either a user or a service principal.

### Detection Query (User as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add owner to application"
| where TargetResources[0].type == "User"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Target = tostring(TargetResources[0].userPrincipalName)
| extend ['Application Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Application ObjectId'] = tostring(TargetResources[1].id)
| where isnotempty( Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], Target, ['Application Name'], ['Application ObjectId']
```

### Detection Query (User as actor, service principal as target)

```kql
AuditLogs
| where OperationName == "Add owner to application"
| where TargetResources[0].type == "ServicePrincipal"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend ['Target Application Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Target Application ObjectId'] = tostring(TargetResources[1].id)
| extend ['Subject Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Subject Service Principal ObjectId'] = tostring(TargetResources[0].id)
| where isnotempty( Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Target Application Name'], ['Target Application ObjectId'], ['Subject Service Principal Name'], ['Subject Service Principal ObjectId']
```

### Detection Query (Service principal as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add owner to application"
| where TargetResources[0].type == "User"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty(['Actor Service Principal ObjectId'])
| extend ['Subject Service Principal Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Subject Service Principal ObjectId'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[0].newValue)))
| extend Target = tostring(TargetResources[0].userPrincipalName)
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Subject Service Principal Name'], ['Subject Service Principal ObjectId'], Target
```

### Detection Query (Service Principal as actor, service principal as target)

```kql
AuditLogs
| where OperationName == "Add owner to application"
| where TargetResources[0].type == "ServicePrincipal"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty(['Actor Service Principal ObjectId'])
| extend ['Subject Service Principal Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Subject Service Principal ObjectId'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[0].newValue)))
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Subject Service Principal Name'], ['Subject Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId']
```

### Control / Prevention
Ownership of applications should be assigned on as needed basis. Owners of applications have the ability to configure the application and generate additional credentials on the application. Changes to applications, particular new credential generation should be audited.


## BARK function - Test-MGAddSelfAsOwnerOfSP 

Owners of service principals can change settings on that object, for instance they can add or remove users who have access to sign into that service principal. They can change SSO settings and change permissions on the service principal.

For this abuse, the actor can be either a user or a service principal. The target can also be either a user or a service principal.

### Detection Query (User as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add owner to service principal"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Service Principal Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Service Principal ObjectId'] = tostring(TargetResources[1].id)
| extend Target = tostring(TargetResources[0].userPrincipalName)
| where TargetResources[0].type == "User"
| where isnotempty(Actor)
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], Target, ['Service Principal Name'], ['Service Principal ObjectId']
```

### Detection Query (User as actor, Service Principal as target)

```kql
AuditLogs
| where OperationName == "Add owner to service principal"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend Target = tostring(TargetResources[0].userPrincipalName)
| where TargetResources[0].type == "ServicePrincipal"
| where isnotempty(Actor)
| extend ['Subject Service Principal ObjectId'] = tostring(TargetResources[0].id)
| extend ['Subject Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Target Service Principal ObjectId'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[0].newValue)))
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Subject Service Principal Name'], ['Subject Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId']
```

### Detection Query (Service Principal as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add owner to service principal"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty( ['Actor Service Principal ObjectId'])
| extend ['Target Service Principal Name'] = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[1].id)
| where TargetResources[0].type == "User"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId'],Target
```

### Detection Query (Service principal as actor, Service Principal as target)

```kql
AuditLogs
| where OperationName == "Add owner to service principal"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty( ['Actor Service Principal ObjectId'])
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| where TargetResources[0].type == "ServicePrincipal"
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId']
```

### Control / Prevention
Ownership of service principals should be assigned on as needed basis. Owners of service principal have the ability to configure the service principal and generate additional credentials. Changes to service principals, especially credential creation should be audited.

## BARK function - Test-MGAddSelfToAADRole

Azure AD roles govern what users and service principals have access to within a tenant. When a user or service principal is added to a role, they then gain added privilege. For instance a SharePoint Admininstrator can manage all aspects of SharePoint.

For this abuse, the actor can be either a user or a service principal. The target can also be either a user or a service principal.

### Detection Query (User as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add member to role"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| where TargetResources[0].type == "User"
| where Identity != "MS-PIM"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| extend RoleAdded = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], Target, RoleAdded
```

### Detection Query (User as actor, service principal as target)

```kql
AuditLogs
| where OperationName == "Add member to role"
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| where Identity != "MS-PIM"
| where TargetResources[0].type == "ServicePrincipal"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend Target = tostring(TargetResources[0].userPrincipalName)
| extend ['Service Principal Name Added'] = tostring(TargetResources[0].displayName)
| extend ['Service Principal ObjectId Added'] = tostring(TargetResources[0].id)
| extend RoleAdded = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Service Principal Name Added'], ['Service Principal ObjectId Added'],RoleAdded
```

### Detection Query (Service principal as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add member to role"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where TargetResources[0].type == "User"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| extend RoleAdded = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'],Target, RoleAdded
```

### Detection Query (Service principal as actor, service principal as target)

```kql
AuditLogs
| where OperationName == "Add member to role"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty( ['Actor Service Principal ObjectId'])
| where ['Actor Service Principal Name'] != "MS-PIM"
| where TargetResources[0].type == "ServicePrincipal"
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend RoleAdded = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'],['Target Service Principal Name'], ['Target Service Principal ObjectId'], RoleAdded
```

### Control / Prevention
The ability to assign Azure AD roles should be limited to few staff. When roles are assigned to users it should be on a least privilege basis and be bound by Azure AD PIM if possible. All non-PIM role assignments should be monitored and audited. When service principals hold an Azure AD role, they should be governed by other controls such as Conditional Access.

## BARK function - Test-MGAddOwnerToRoleEligibleGroup 

A role eligible group is an Azure AD group that has been assigned an Azure AD role. By adding an owner to one of those groups, that person can then manage membership to the group. When a user is added to the grouop they gain access to the Azure AD role. An owner can also potentially delete the group.

The Azure AD audit log doesn't differentiate between an owner being added to a regular group or a role assignable group. I have added a placeholder field for specific group names you want to monitor.

For this abuse, the actor can be either a user or a service principal. The target can also be either a user or a service principal.

### Detection Query (User as actor, user as target)

```kql
AuditLogs
| where TimeGenerated > ago(30m)
| where OperationName == "Add owner to group"
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| where TargetResources[0].type == "User"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], Target, GroupName
```

### Detection Query (User as actor, service principal as target)

```kql
AuditLogs
| where TimeGenerated > ago(30m)
| where OperationName == "Add owner to group"
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| where TargetResources[0].type == "ServicePrincipal"
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Target Service Principal Name'], ['Target Service Principal ObjectId'], GroupName
```

### Detection Query (Service principal as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add owner to group"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty(['Actor Service Principal ObjectId'])
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| where TargetResources[0].type == "User"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], Target, GroupName
```

### Detection Query (Service Principal as actor, service principal as target)

```kql
AuditLogs
| where TimeGenerated > ago(30m)
| where OperationName == "Add owner to group"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty(['Actor Service Principal ObjectId'])
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| where TargetResources[0].type == "ServicePrincipal"
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].displayName)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].id)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId'], GroupName
```

### Control / Prevention
Ownership of groups that are assigned Azure AD roles should be limited, otherwise privilege can easily creep. Alert on any changes to ownership of high privilege groups.

## BARK function - Test-MGAddMemberToRoleEligibleGroup 

A role eligible group is an Azure AD group that has been assigned an Azure AD role. By adding a member to one of those groups, that user or service principal then has access to the Azure AD role. 

The Azure AD audit log doesn't differentiate between an member being added to a regular group or a role assignable group. I have added a placeholder field for specific group names you want to monitor.

For this abuse, the actor can be either a user or a service principal. The target can also be either a user or a service principal.

### Detection Query (User as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add member to group"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| where TargetResources[0].type == "User"
| extend Target = tostring(TargetResources[0].userPrincipalName)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], Target, GroupName
```

### Detection Query (User as actor, Service Principal as target)

```kql
AuditLogs
| where OperationName == "Add member to group"
| extend Actor = tostring(parse_json(tostring(InitiatedBy.user)).userPrincipalName)
| extend ['Actor IP Address'] = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].id)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].displayName)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| where TargetResources[0].type == "ServicePrincipal"
| project TimeGenerated, OperationName, Actor, ['Actor IP Address'], ['Target Service Principal Name'], ['Target Service Principal ObjectId'], GroupName
```

### Detection Query (Service Principal as actor, user as target)

```kql
AuditLogs
| where OperationName == "Add member to group"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty(['Actor Service Principal ObjectId'])
| where ['Actor Service Principal Name'] != "MS-PIM"
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| where TargetResources[0].type == "User"
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| extend Target = tostring(TargetResources[0].userPrincipalName)
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], Target, GroupName
```

### Detection Query (Service Principal as actor, service principal as target)

```kql
AuditLogs
| where OperationName == "Add member to group"
| extend ['Actor Service Principal Name'] = tostring(parse_json(tostring(InitiatedBy.app)).displayName)
| extend ['Actor Service Principal ObjectId'] = tostring(parse_json(tostring(InitiatedBy.app)).servicePrincipalId)
| where isnotempty(['Actor Service Principal ObjectId'])
| where ['Actor Service Principal Name'] != "MS-PIM"
| where TargetResources[0].type == "ServicePrincipal"
| extend GroupName = tostring(parse_json(tostring(parse_json(tostring(TargetResources[0].modifiedProperties))[1].newValue)))
| extend ['Target Service Principal Name'] = tostring(TargetResources[0].id)
| extend ['Target Service Principal ObjectId'] = tostring(TargetResources[0].displayName)
| where GroupName in~ ("PrivilegedGroup1","PrivilegedGroup2")
| project TimeGenerated, OperationName, ['Actor Service Principal Name'], ['Actor Service Principal ObjectId'], ['Target Service Principal Name'], ['Target Service Principal ObjectId'], GroupName
```

### Control / Prevention

Membership to groups that are assigned Azure AD roles should be limited, otherwise privilege can easily creep. Alert on any changes to membership of high privilege groups.

## Related Links

[Bloodhound BARK](https://github.com/BloodHoundAD/BARK)

[Detecting privilege escalation with Azure AD service principals in Microsoft Sentinel](https://learnsentinel.blog/2022/01/04/azuread-privesc-sentinel/)

[Azure Privilege Escalation via Azure API Permissions Abuse](https://posts.specterops.io/azure-privilege-escalation-via-azure-api-permissions-abuse-74aee1006f48)
