# logging-elk-filebeat
Logging Nginx access, error logs and Laravel app logs

<h3><b>Setup environment</b></h3>

Ubuntu 20.04.5 LTS

ELK stack on docker https://github.com/deviantony/docker-elk

nginx version: nginx/1.18.0 (Ubuntu)

filebeat version 8.4.1 (amd64), libbeat 8.4.1 [fe210d46ebc339459e363ac313b07d4a9ba78fc7 built 2022-08-25 19:48:45 +0000 UTC]


After setup environment by mentioned requirements then should be configured <b>filebeat.yml</b> and created <b>logstash.conf</b> in logstash pipeline.

Changes in <b>filebeat.yml</b>
```
# ============================== Filebeat inputs ===============================

filebeat.inputs:

- type: log
  id: access-filestream-id
  enabled: true
  paths:
    - /var/log/nginx/access.log
  fields:
    type: access
    log_prefix: nginx-access
  fields_under_root: true

- type: log
  id: error-filestream-id
  enabled: true
  paths:
    - /var/log/nginx/error.log
  fields:
    type: error
    log_prefix: nginx-error
  fields_under_root: true

- type: log
  id: laravel-filestream-id
  enabled: true
  paths:
  - /var/www/html/aio-board/storage/logs/laravel.log
  exclude_lines: ['^#[0-9]', '^\"}','^\[stacktrace\]']
  fields:
    type: laravel
    log_prefix: aio-laravel
  fields_under_root: true
```
```
# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
# hosts: ["0.0.0.0:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "http"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
#username: "elastic"
#password: "5ZfV*wW2LaaiS0SGq763"
#ssl:
#    enabled: true
#    ca_trusted_fingerprint: "6c58d93cf68ac2e47d3b4ffa1a8f77edf36448d167b09b7f84e2fd16860431f7"

# ------------------------------ Logstash Output -------------------------------
output.logstash:
  # The Logstash hosts
 hosts: ["0.0.0.0:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"
```
then filebeat service should be started

```
service filebeat start
```
Filebeat will trying to sent log entries row by row to logstash, also there are rules for excluding some trace rows in laravel log and defined <b>fields:</b> <b>type</b> and <b>log_prefix</b> that will used in logstash configuration file.

It's need to create <b>logstash.conf</b> in pipeline folder for receiving filebeat stream, applying filters, and dynamically create indixces for separate log types.

<b>logstash.conf</b>

```
input {  
   beats {
      # The port to listen on for filebeat connections.
      port => 5044
      # The IP address to listen for filebeat connections.
      host => "0.0.0.0"
   }
}
filter {
   if [type] == "access" {
       grok {
           match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user} \[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:agent}\"" }
       }
   }
   if [type] == "laravel" {
       grok {
          #match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{DATA:env}\.%{DATA:severity}: %{DATA:message}" }
          match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{DATA:env}\.%{DATA:severity}: %{DATA:message} \{" } 
      }
   }
   if [type] == "error" {
      grok {
         match => { "message" => "(?<timestamp>%{YEAR}[./]%{MONTHNUM}[./]%{MONTHDAY} %{TIME}) \[%{LOGLEVEL:severity}\] %{POSINT:pid}#%{NUMBER:threadid}\: \*%{NUMBER:connectionid} %{GREEDYDATA:errormessage}, client: %{IP:client}, server: %{GREEDYDATA:server}, request: \"(?<httprequest>%{WORD:httpcommand} %{UNIXPATH:httpfile} HTTP/(?<httpversion>[0-9.]*))\"(, )?(upstream: \"(?<upstream>[^,]*)\")?(, )?(host: \"(?<host>[^,]*)\")?" }
      }
   }
   mutate {
      copy => {
         "[log_prefix]" => "[@metadata][log_prefix]"
      }
   }
}
output {
   elasticsearch {
       hosts => ["elasticsearch:9200"]
       index => "%{[@metadata][log_prefix]}.logs"
   }
   stdout { codec => rubydebug }
}
```
Then we should starting ELK containers. 

After receiving event that will add row in logs logstash will pass formatted data to elasticsearch.

In our case for checking logstash container log use command below.

```
sudo docker logs -f container_id
```

After we need to configure <b>Kibana</b> to creating <b>Data View</b>.
