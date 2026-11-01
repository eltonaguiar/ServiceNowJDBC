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


# Linux - GX10 mid server fix
<img width="1049" height="269" alt="image" src="https://github.com/user-attachments/assets/98c944d7-b5a1-42c0-880b-c2cfa0c9c260" />

// bash
docker restart sn-mid

That's it - the only thing I changed on the box. No config edits, no container recreation, no image pull. The rest was read-only inspection.

Exact command sequence, in order

1. Confirm host and locate the MID (read-only)

// bash
hostname
systemctl list-units -- all -- no-pager | grep -iE "mid|servicenow|sn_"
docker ps -a -- format '{{.Names}}\t{{.Status} } \t{{.Image}}'

sn-mid | Up 4 days | moers/mid-server:latest (and a dead

# -> gx10-c9b9 (so "gx10" = this machine)

sn-mid mid1 | Exited (137) ).

win11-mid: Up

# -> "tail -F ... logs" (PID 1 is not the MID)

log ended

2. Check the instance side (read-only API)

// bash
curl -s -u "$SN_USER:$SN_PASS" -H "Accept: application/json" \
"https://dev296154.service-now.com/api/now/table/ecc_agent?sysparm_fields=name, status, validated, last_refreshed, host_name, version&sysparm_limit=10"

gx10-mid: status Down, validated "validating", last_refreshed 2026-11-01 11:28:30

3. Prove the JVM was dead despite a healthy-looking container

// bash
docker logs -- tail 40 sn-mid
docker exec sn-mid sh -c 'cat /proc/1/cmdline | tr "\0" " "'
docker exec sn-mid sh -c 'ps aux | grep "[j]ava""
docker exec sn-mid ps -o pid,etime,lstart,args -p 15
docker exec sn-mid stat -c %s /opt/agent/logs/agent0.log.0

Shutting down services ... MID Server stopped ... now terminated
still lingered.

4. Rule out the usual suspects

// bash
docker exec sn-mid sh -c 'grep -nE "OutOfMemory|Shutdown requested|SEVERE| FATAL| Killed" /opt/agent/logs/agent0.log.0 | tail -25'
docker inspect sn-mid -- format 'RestartPolicy={{.HostConfig.RestartPolicy.Name}} Status={{.State.Status}}'
free -h
sudo -n dmesg | grep -iE "oom|killed process"

graceful shutdown (not 00M), RestartPolicy=unless-stopped , 72 GiB available. The decisive line:
automatically restarted. Shutting down. MANUALLY START THE MID SERVER TO CONTINUE .

# ran twice, 15 s apart -> size frozen

at 11:31:48, and the

agent0.log.0

stopped growing while the JVM process

Wrapper-less MID Server cannot be
5. The fix

// bash
docker restart sn-mid

6. Verify (this is the part that matters)

// bash
docker ps -- filter name=sn-mid -- format '{{.Names}} | {{.Status}} | {{.RunningFor}}'
docker exec sn-mid ps -o pid,etime,args -C java
docker logs -- tail 400 sn-mid | grep -viE "DEBUG|^$" | tail -25
docker logs -- tail 3000 sn-mid | grep -iE "Operational State|connected to instance host|401|Unable to load key store" | tail -12

MID Server is connected to instance host: dev296154.service-now.com

new JVM PID 16,

// bash
for i in $(seq 1 10); do
curl -s -u "$SN_USER:$SN_PASS" \
"https://dev296154.service-now.com/api/now/table/ecc_agent?sysparm_query=name=gx10-mid&sysparm_fields=name, status,validated, last_refreshed,
version"
echo "$R" | grep -q '"status":"Up"' && break; sleep 20
done

AutoUpgrade.run(), MID server Operational State=UP

first poll:

Also produced

ServiceNowMidServer/FIX-MID-LINUX.md - the runbook version of the above (triage + restart + keystore/401/ownership repairs + supervisor relaunch
watchdog verification don'ts). I syntax-checked it: 17 bash blocks, 0 syntax errors, and the embedded watchdog script parses clean with bash
-n

The commands took ~90 seconds wall clock from

gx10-mid | Up | validated: true | last_refreshed 2026-11-01 11:34:32

hostname

to

status: Up ; the restart itself was ~45 s to reconnect.
