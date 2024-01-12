# CISA Known Exploitable Vulnerabilities Dashboard

The following dashboard can be used in Azure Data Explorer to visualize and provide insights to the CISA Known Exploitable Vulnerability data found here - https://www.cisa.gov/known-exploited-vulnerabilities-catalog

To access this dashboard you can sign up to a free Kusto cluter at https://aka.ms/KustoFree, or use an existing paid or free cluster

Select Dashboards on the left

![image](https://github.com/reprise99/Sentinel-Queries/assets/88635951/1c5b9de1-a85d-4766-b2d7-9930a6edf193)

Select Import dashboard from file

![image](https://github.com/reprise99/Sentinel-Queries/assets/88635951/2f5ce68a-a1a1-46b8-8019-7394abf19644)

Download and import the JSON found in this repo (raw link https://raw.githubusercontent.com/reprise99/Sentinel-Queries/main/CISA%20Insights/dashboard-CISA%20KEV%20Insights.json), you can name your dashboard whatever you like

You should then be able to explore the dashboard

![image](https://github.com/reprise99/Sentinel-Queries/assets/88635951/cf83bc4f-4919-4235-83fd-45b865609b41)

![image](https://github.com/reprise99/Sentinel-Queries/assets/88635951/2ab1c6b9-3448-4c5c-9c47-e1f4ca07f3d4)

By default it looks at all vendors, but you can select one or more to look at only those. The vendor list is generated dynmically from the same KEV data

![image](https://github.com/reprise99/Sentinel-Queries/assets/88635951/a731c8a2-8d03-4b5c-8f21-280ff72f4851)

In order to create dashboards, you need a connection to a Kusto cluster, to get around this requirement this dashboard creates a connection to the Microsoft sample Kusto cluster, but it then uses only the data available directly from CISA
