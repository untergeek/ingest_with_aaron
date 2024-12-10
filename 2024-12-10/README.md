# 2024-12-10 Ingest Series

# Fleet Setup

## Add Logstash output to Fleet

Fleet -> Settings -> Add Output (+)

- Name it
- Type -> Logstash
- Additional Logstash configuration required:
    - Click "View Steps"
        - Click "Generate API Key"
            - Don't click more than once. It will create multiple API keys, each named `Fleet Logstash output`, which will cause confusion later on.
            - This will auto-populate the `api_key` in the `elasticsearch` output in the sample plugin.
            - COPY THE VALUE AND SET IT ASIDE (it cannot be retrieved again later).
        - Copy pipeline def to target
        - Step 4: "Replace the parts between the brackets with your generated SSL certificate file paths. View our documentation(external) to generate the certificates." Visit the external documentation
            - Generate Fleet certificates ([in Documentation](https://www.elastic.co/guide/en/fleet/8.16/secure-logstash-connections.html)): Visit [this doc](evernote:///view/147704/s3/0fefd6d2-dd5b-3ac6-8dec-2e1eca9ae334/87230cc5-518e-4737-8209-53b2647019db) for breakout
                - Copy certs to paths on Logstash host as directed.
- Add hostname (or IP) of Logstash host. 
- Paste CA data (ostensibly `ca/ca.crt`) into web form where **Server SSL certificate authorities (optional)** is indicated
- Paste Client Public Cert data (ostensibly `client/client.crt`) into web form where **Client SSL certificate** is indicated.
- Paste Client Private Key data (ostensibly `client/client.key`) into web form where **Client SSL certificate key** is indicated

### Logstash Pipeline

Edit Logstash Pipeline:

```ruby
input {
  elastic_agent {
    port => 5044
    ssl_enabled => true
    ssl_certificate_authorities => ["/path/to/ca.crt"]
    ssl_certificate => "/path/to/logstash.crt"
    ssl_key => "/path/to/logstash.pkcs8.key"
    ssl_verify_mode => "force_peer"
  }
}

filter {
  elastic_integration { 
    hosts => [ "<es_host>" ]
    api_key => "<api_key>"
    # ssl_verification_mode => "none"
    geoip_database_directory => "/usr/share/logstash/vendor/bundle/jruby/3.1.0/gems/logstash-filter-geoip-7.3.1-java/vendor/GeoLite2-City.mmdb"
    # Can substitute GeoLite2-ASN.mmdb as the last leaf of that path
  }
}

output {
  elasticsearch {
    hosts => "<es_host>"
    api_key => "<api_key>"
    data_stream => true
    ecs_compatibility => "v8"
    ssl_enabled => true
    # cacert => "<elasticsearch_ca_path>"
  }
}
```

- In `elastic_agent`:
    - `ssl` should be `ssl_enable` (`ssl` is deprecated)
- Ensure that `elastic_integration` is the first filter
- Add any other filters (proof we passed through Logstash)
- In the `elasticsearch` output plugin:
    - `data_stream => true` 
    - `ecs_compatibility => "v8"`
    - `hosts => [ "<es_host>" ]` 
    - `api_key => "<api_key>"` 
    - `ssl_enabled => true`  
    - `ssl_verification_mode => "none"`  # Because we are connecting to an internal k8s in this case

## Update API Key

The API key they generate offers _most_ necessary roles, but does not include everything necessary for Integrations to work. The easiest path forward is to update the API key that was created with the necessary bits.

### Navigate to API Key UI

In Kibana, select the hamburger menu, scroll to the bottom, and select **Stack Management**

In the resulting window, in the left panel, under **Security**, select **API keys**

### Search for "Fleet Logstash output"

In the list of API keys, one will be called **Fleet Logstash output**. It should be the most recently created API key. If you click on the **Created** column to sort so it shows a down arrow, the most recently created keys will show first. Search doesn't seem to work correctly for me (typed a single character and all API keys vanished), so sorting by created seems to be a good bet here.

Use caution. If you clicked on the `Generate API Key` button multiple times, you won't necessarily know which one of these entries you kept. If you need to delete each of the entries labeled `Fleet Logstash output`, and create another in the dialog to add another Logstash, that's fine. Just copy/keep/assign that value where needed, then come back and edit the API key privileges in the UI.

### Add the following cluster privileges:

- `read_pipeline` - Only necessary if using centralized pipeline management, but it shouldn't hurt to add regardless
- `manage_index_templates`

The resulting block should look like this:

```json
{
  "logstash-output": {
    "cluster": [
      "monitor",
      "read_pipeline",
      "manage_index_templates"
    ],
    ... # Other entries below
}
```

Save and apply these changes by clicking **Update API key** in the lower-right corner.

## Deploy Pipeline to Logstash

There are a number of ways this can be accomplished. Today, I'm using Centralized Pipeline Management so I can edit the pipeline in Kibana.

### logstash.yml

```yaml
config:
  reload:
    automatic: true
    interval: 3s
log.level: info
pipeline:
  workers: 4
  ordered: auto
  ecs_compatibility: v8
xpack:
  management:
    enabled: true
    logstash.poll_interval: 5s
    elasticsearch:
      hosts: [ "<es_host>" ]
      api_key: "<api_key>"
      ssl:
        verification_mode: "none" # If needed, as it is for me
    pipeline.id:
      - integrations
```

Many of these can be one-line, but with so many nested, it sometimes makes sense to visualize it this way.

The important bits here for Centralized Pipeline Management are all under `xpack.management`:

- `enabled: true` - This enables centralized pipeline management (must have X-Pack enabled ES Cluster)
- `logstash.poll_interval` - How frequently Logstash will poll for pipeline changes
- `elasticsearch.*` - The connection parameters
- `pipeline.id` - A YAML array/list of named pipelines that this instance of Logstash will load and run. These must be specified here at runtime! You cannot add more without restarting Logstash.

### Add pipeline to Centralized Pipeline Management

In Kibana, navigate from the hamburger menu to **Stack Management**

In the resulting window, in the left pane near the top under **Ingest**, click on **Logstash Pipelines**. This option may not appear for you if your license level is not sufficient.

Click on **Create Pipeline** on the right side.

In the resulting window, you can paste in your pipeline. There's an API for this so you can manage your infrastructure as "code" by committing to git, then pull and publish your pipelines to the API, and Logstash will execute them.

Feel free to edit the selections at the bottom accordingly.

- Pipeline workers
- Pipeline batch size
- Pipeline batch delay
- Queue type
- Queue max bytes
- Queue checkpoint writes

# Agent Setup in Fleet

## Two Agent Policies

1. Ship to ES
2. Ship to LS

## Two Hosts

1. ship2es
2. ship2ls

## Install Agent on each host

Follow the "add agent" dialog
