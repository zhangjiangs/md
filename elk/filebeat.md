filebeat.yaml

```yaml
filebeat.inputs:

# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input specific configurations.

- type: log
  enabled: true
  paths:
    - /nfs/dp/dispatching/edpa-dispatching.log
  fields:
    log_source: dp-edpa-dispatching

- type: log
  enabled: true
  paths:
    - /nfs/dp/ingestion/edpa.log
  fields:
    log_source: dp-edpa

- type: log
  enabled: true
  paths:
    - /nfs/dp/edpa-ingestion/edpa-ingestion.log
  fields:
    log_source: dp-edpa-ingestion

```

```yaml
#================================ Processors =====================================
#--------------------- redis output ----------------------

output.redis:
  hosts: ["xx.xx.xx.xx:6379"]
  password: "1xxxx"
  key: "dp-edpa-dispatching"
  db: 12
  timeout: 5

output.redis:
  hosts: ["xx.xx.xx.xx:6379"]
  password: "xxxx6"
  key: "dp-edpa"
  db: 12
  timeout: 5

output.redis:
  hosts: ["xx.xx.xx.xx:6379"]
  password: "xxxxx6"
  key: "dp-edpa-ingestion"
  db: 12
  timeout: 5

```

