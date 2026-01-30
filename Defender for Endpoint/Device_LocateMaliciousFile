DeviceFileEvents
| where FileName contains "********" // include malicious file name and extension 
| where TimeGenerated > ago(30d) // select time range
| project TimeGenerated, ActionType, DeviceName, DeviceId, FileOriginUrl, FolderPath, InitiatingProcessAccountUpn
