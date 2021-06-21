# nap-dos-demo

## Introduction
At the time of this writing, NGINX App Protect DoS (NAP DoS) is still in beta version. You may sign up for free trial at https://f5beta.centercode.com/welcome/, you will receive instruction to download binaries and instructions to get started.

The purpose of this repo is to provide an easy setup to learn and demonstrate NAP DoS.

I have clone the NAP DoS dashboard from https://github.com/f5devcentral/nap-dos-elk-dashboards into this repo for ease of setup.

## Setup
1. Clone the repo
```
git clone https://github.com/mcheo-nginx/nap-dos-demo.git
cd nap-dos-demo
```

2. Step up the stacks
```
docker-compose -f docker-compose.yaml up -d
```
The stack consists of:

- 1. NGINX App Protect DoS instance
- 2. Juice Shop as backend app server
- 3. Legitimate container to generate good traffic
- 4. Attacker container to generate attack traffic
- 5. Elasticsearch for NAP DoS dashboard


3. Complete Elasticsearch setup</br>
Use browser to visit http://localhost:5601, once the page is successfully loaded which means startup has completed, execute the following steps:

```
step 3.1
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.disk.threshold_enabled": "false"
  }
}'

step 3.2
curl -X PUT "localhost:9200/_all/_settings?pretty" -H 'Content-Type: application/json' -d'
{
	"index.blocks.read_only_allow_delete": null
}
'

step 3.3
cd elk

curl -XPUT "http://localhost:9200/app-protect-dos-logs"  -H "Content-Type: application/json" -d  @apdos_mapping.json

curl -XPOST "http://localhost:9200/app-protect-dos-logs/_mapping"  -H "Content-Type: application/json" -d  @apdos_geo_mapping.json

step 3.4
KIBANA_CONTAINER_URL=http://localhost:5601

jq -s . kibana/apdos-dashboard.ndjson | jq '{"objects": . }' | \
curl -k --location --request POST "$KIBANA_CONTAINER_URL/api/kibana/dashboards/import" \
    --header 'kbn-xsrf: true' \
    --header 'Content-Type: text/plain' -d @- \
    | jq

```

4. Enable NAP DoS logging</br>
Edit nginx/nginx.conf file and uncomment the following 2 lines:
```
#app_protect_dos_security_log "/etc/app_protect_dos/log-default.json" syslog:server=elasticsearch:5261;

#access_log syslog:server=elasticsearch:5561 log_dos if=$loggable;
```
Execute NGINX reload
```
docker exec -it nap-dos_nginx_1 nginx -s reload
```

You may browse the application at http://localhost

5. Generate baseline traffic
```
docker exec -it nap-dos_legitimate_1 /bin/bash
cd /home
./good.sh
```
Let it runs for at least 5-8 minutes, let NAP DoS to do initial baselining. If everything is working as expected, Elasticsearch AP_DOS: AppProtectDOS dashboard should display charts.

You may visit the Elasticsearch dashboard at http://localhost:5601
In the dashboard, apply or remove filter as you wish.

6. Generate attack traffic</br>
While the good traffic still running at the background, you may start the attack traffic
```
docker exec -it nap-dos_legitimate_1 /bin/bash
cd /home
./bad.sh
```



## Interpreting the Graph

<img src="elk/images/dashboard1.png" width="800px"/>

The general idea is NAP DoS leverage on its ML to perform behaviour dos mitigation. No manual (human) tuning, re-tuning required.

- When the attack happens, a sharp increase of traffic in "AP_DOS: Client HTTP transaction/s" panel and "AP_DOS: Server HTTP transactions/s"
- NGINX detects server stress, spike in AP_DOS: Server_stress_level panel
- NGINX will goes into self defense mode by doing global challenge. Red patch (Challenge global RPS) shows in "AP_DoS: HTTP Mitigation" panel
- Once NAP DoS has sufficient data and generate dynamic attack signatures in response to this particular attack, attack signature shows in "AP_DoS: Attack Signatures" panel, purple patch (Challenge signatures RPS) shows in "AP_DOS: HTTP Mitigation" panel
- If the attack prolongs and those source IP has been identified as bad actors in "AP_DOS: Detected bad actors" panel, yellow patch (Challenge bad actors RPS) shows in "AP_DOS: HTTP Mitigation" panel
- Despite the attack traffic keeps coming in (high traffic in "AP_DOS: Client HTTP transaction/s" panel), NAP DoS has mitigate and send only legitimate traffic to backend server (lower traffic in "AP_DOS: Server HTTP transactions/s" panel). As a good user, you may visit the page in http://localhost and it is working fine.



