# ServiceNowJDBC

//mysql should be in path otherwise.. 
//need to find something like this..
//C:\Program Files\MySQL\MySQL Server 8.0\bin> 

.\mysql -h mysql.<webshost>.com -u ejaguiar1_events -pMYPASSWORDHERE -e "SELECT * from ejaguiar1_events.events_log;"



# Windows 11 mid servers was under services.msc 
C:\ServiceNow\MID\agent\bin\wrapper-windows-x86-64.exe -s C:\ServiceNow\MID\agent\conf\wrapper.conf

#Windows 11
ServiceNow MID Server
service name:snc_mid
DisplayName:ServiceNow MID Server
path:C:\ServiceNow\MID\agent\bin\wrapper-windows-x86-64.exe -s C:\ServiceNow\MID\agent\conf\wrapper.conf
logon : local system account
