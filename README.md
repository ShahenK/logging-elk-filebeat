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
