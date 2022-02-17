# Sentinel vs Advanced Hunting

While both Microsoft Sentinel and Advanced Hunting leverage KQL, there are differences in schema in certain tables. For instance, TimeGenerated is used in Sentinel while Timestamp is used in Advanced Hunting.

The below tables are designed to help you convert queries between the two products.

## Azure AD Signin Logs

For Microsoft Sentinel sign in logs are kept in two separate tables.

SigninLogs - for interactive signins

```kql
SigninLogs
| take 10
```

AADNonInteractiveUserSignInLogs - for non-interactive signins

```kql
AADNonInteractiveUserSignInLogs
| take 10
```

For Advanced Hunting both types of logs are kept in the same table, but distinguised by a field.

For interactive signins

```kql
AADSignInEventsBeta
| where LogonType == @"[""interactiveUser""]"
| take 10
```

For non-interactive sigins

```kql
AADSignInEventsBeta
| where LogonType == @"[""nonInteractiveUser""]"
| take 10
```

Differences in schema are noted below.

| Field Description | Sentinel Examples | Advanced Hunting Examples | Notes |
| -------------   | ---------- | --------------------------------------------------------------------------------------------------------------------------------| ----- |
| Time  | SigninLogs <br />\| where TimeGenerated > ago (7d)| AADSignInEventsBeta <br />\| where Timestamp > ago(7d) | TimeGenerated becomes Timestamp |
| Username  | SigninLogs <br />\| where UserPrincipalName == "reprise@domain.com"  | AADSignInEventsBeta <br />\| where AccountUpn == "reprise99@domain.com" | UserPrincipalName becomes AccountUpn |
| Account Display Name | SigninLogs <br />\| where UserDisplayName== "Jane Citizen" | AADSignInEventsBeta <br />\| where AccountDisplayName == "Jane Citizen" | UserDisplayName becomes AccountDisplayName
| Azure AD Object Id | SigninLogs <br />\| where UserId == "33d3f3dd-1e47-4cbd-82dd-93e17e0107f2" | AADSignInEventsBeta <br />\| where AccountObjectId == "33d3f3dd-1e47-4cbd-82dd-93e17e0107f2" | UserId becomes AccountObjectId
| Application Name | SigninLogs <br />\| where AppDisplayName == "Microsoft Teams" | AADSignInEventsBeta <br />\| where Application == "Microsoft Teams" | AppDisplayName becomes Application
| Application Id | SigninLogs <br />\| where AppId == "00000002-0000-0ff1-ce00-000000000000" | AADSignInEventsBeta <br />\| where ApplicationId == "00000002-0000-0ff1-ce00-000000000000" | AppId becomes ApplicationId
| Guest User | SigninLogs <br />\| where UserType == "Guest" | AADSignInEventsBeta <br />\| where IsGuestUser == "1" | UserType = guest becomes IsGuestUser = 1
| Status | SigninLogs <br />\| where ResultType == "50126" | AADSignInEventsBeta <br />\| where ErrorCode == "50126" | ResultType becomes ErrorCode
| Conditional Access Status | SigninLogs <br />\| where ConditionalAccessStatus == "success" | AADSignInEventsBeta <br />\| where ConditionalAccessStatus == "0" | Sentinel uses strings vs advanced hunting numbers. success = 0, failure = 1, notApplied = 2
| Device Name | SigninLogs <br />\| extend DeviceName = tostring(DeviceDetail.displayName) <br />\| where DeviceName == "LAPTOP1234" | ADSignInEventsBeta <br />\| where DeviceName == "LAPTOP1234" | Sentinel keeps device name in a nested field and has to be extracted first
| Device Id | SigninLogs <br />\| extend deviceId = tostring(DeviceDetail.deviceId) <br />\| where deviceId = "33f17af6-82c4-4a91-8e4f-6b5ba417efbf" | AADSignInEventsBeta <br />\| where AadDeviceId == "33f17af6-82c4-4a91-8e4f-6b5ba417efbf" | Sentinel keeps device id in a nested field and has to be extracted first
Device Trust Type | SigninLogs <br />\| extend trustType = tostring(DeviceDetail.trustType) <br />\| where trustType == "Azure AD registered" | AADSignInEventsBeta <br />\| where DeviceTrustType == "Azure AD registered" | Sentinel keeps trust type in a nested field and has to be extracted first
| Device OS | SigninLogs <br />\| extend DeviceOS = tostring(DeviceDetail.operatingSystem) <br />\| where DeviceOS = "Windows 10" | AADSignInEventsBeta <br />\| where OSPlatform == "Windows 10" | Sentinel keeps OS in a nested field and has to be extracted first
| Browser | SigninLogs <br />\| extend Browser = tostring(DeviceDetail.browser) <br />\| where Browser == "Edge 98.0.1108" | AADSignInEventsBeta <br />\| where Browser == "Edge 98.0.1108" | Sentinel keeps browser in a nested field and has to be extracted first
| City | SigninLogs <br />\| extend City = tostring(LocationDetails.city) <br />\| where City == "Melbourne" | AADSignInEventsBeta <br />\| where City == "Melbourne" | Sentinel keeps City in a nested field and has to be extracted first
| Country | SigninLogs <br />\| extend Country = tostring(LocationDetails.countryOrRegion) <br />\| where Country == "AU" | AADSignInEventsBeta <br />\| where Country == "AU" | Sentinel keeps Country in a nested field and has to be extracted first
| Latitude | SigninLogs <br />\| extend Latitude = tostring(parse_json(tostring(LocationDetails.geoCoordinates)).latitude) <br />\| where Latitude contains "49.25" | AADSignInEventsBeta <br /> \| where Latitude contains "49.25" | Sentinel keeps Latitude in a nested field and has to be extracted first
| Longitude | SigninLogs <br />\| extend Longitude = tostring(parse_json(tostring(LocationDetails.geoCoordinates)).longitude) <br />\| where Longitude contains "-122" | AADSignInEventsBeta <br />\| where Longitude contains "-122" | Sentinel keeps Longitude in a nested field and has to be extracted first
| State |SigninLogs <br />\| extend State = tostring(LocationDetails.state) <br />\| where State == "British Columbia" | AADSignInEventsBeta <br />\| where State == "British Columbia"  <br /> | Sentinel keeps State in a nested field and has to be extracted first
| Combined Example | SigninLogs <br />\| extend State = tostring(LocationDetails.state) <br />\| where UserType == "Guest" and State == "British Columbia" <br />and ResultType == "0" and AppDisplayName == "Microsoft Teams" | AADSignInEventsBeta <br />\| where IsGuestUser == "1" and State == "British Columbia" and ErrorCode == "0" and Application == "Microsoft Teams" | Look for guests in British Columbia successfully signing into Teams |

A number of fields are the same across both tables such as UserAgent or ClientAppUsed, and some only exist in one. For example, Advanced Hunting has LastPasswordChangeTimestamp whereas Sentinel does not. In general, the Sentinel logs are more detailed.

## Azure AD Service Principal Signin Logs

Microsoft Sentinel keeps service principal sign in logs in two tables. For regular service principals they are sent to the AADServicePrincipalSignInLogs table.

```kql
AADServicePrincipalSignInLogs
| take 10
```

For managed identities they are sent to the AADManagedIdentitySignInLogs

```kql
AADManagedIdentitySignInLogs
| take 10
```

For Advanced Hunting both types of logs are kept in the same table, but distinguised by a field.

For regular service principals.

```kql
AADSpnSignInEventsBeta
| where IsManagedIdentity == 0
```

For managed identities

```kql
AADSpnSignInEventsBeta
| where IsManagedIdentity == 1
```

Differences in schema are noted below.

| Field Description | Sentinel Example | Advanced Hunting Example | Notes |
| -------------   | ---------- | -----------| --- |
| Time  | AADServicePrincipalSignInLogs <br />\| where TimeGenerated > ago (7d)| AADSpnSignInEventsBeta <br />\| where Timestamp > ago(7d) | TimeGenerated becomes Timestamp
| Application Id | AADServicePrincipalSignInLogs <br />\| where AppId == "00000002-0000-0ff1-ce00-000000000000" | AADSpnSignInEventsBeta <br />\| where ApplicationId == "00000002-0000-0ff1-ce00-000000000000" | AppId becomes ApplicationId
| Status | AADServicePrincipalSignInLogs <br />\| where ResultType == "7000215" | AADSpnSignInEventsBeta <br />\| where ErrorCode == "7000215" | ResultType becomes ErrorCode
| City | AADServicePrincipalSignInLogs <br />\| extend City = tostring(LocationDetails.city) \| where City == "Melbourne" | AADSpnSignInEventsBeta <br />\| where City == "Melbourne" | Sentinel keeps City in a nested field and has to be extracted first
| Country | AADServicePrincipalSignInLogs <br />\| extend Country = tostring(LocationDetails.countryOrRegion) \| where Country == "AU" | AADSpnSignInEventsBeta <br />\| where Country == "AU" | Sentinel keeps Country in a nested field and has to be extracted first
| Latitude | AADServicePrincipalSignInLogs <br />\| extend Latitude = tostring(parse_json(tostring(LocationDetails.geoCoordinates)).latitude) <br />\| where Latitude contains "49.25" | AADSpnSignInEventsBeta <br />\| where Latitude contains "49.25" | Sentinel keeps Latitude in a nested field and has to be extracted first
| Longitude | AADServicePrincipalSignInLogs <br />\| extend Longitude = tostring(parse_json(tostring(LocationDetails.geoCoordinates)).longitude) <br />\| where Longitude contains "-122" | AADSpnSignInEventsBeta <br />\| where Longitude contains "-122" | Sentinel keeps Longitude in a nested field and has to be extracted first
| State |AADServicePrincipalSignInLogs <br />\| extend State = tostring(LocationDetails.state) <br />\| where State == "British Columbia" | AADSpnSignInEventsBeta <br />\| where State == "British Columbia" | Sentinel keeps State in a nested field and has to be extracted first

A number of fields are the same across both products such as ServicePrincipalId and ServicePrincipalName. The Advanced Hunting logs for Service Principals have quite a bit less information than Microsoft Sentinel, and omit things like Conditional Access entirely.