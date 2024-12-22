// Line breaks are for legibility only.

POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token HTTP/1.1
Host: login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

tenantId=1daa4fd5-b537-46b2-b92b-556628c922ed
client_id=da43b39a-01f9-4960-bc79-c98e883d4e0c
&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default
&client_secret=ArE8Q~1pQL_W_8IZiTnG6TNB-.kGWlfzN61fUa2U
&grant_type=client_credentials
