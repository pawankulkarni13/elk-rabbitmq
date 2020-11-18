# elk-rabbitmq

Step by Step Process to Configure below Flow

#### **Postman(Any other application/service publishing logs) --> RabbitMQ --> Logstash --> ElasticSearch --> Kibana --> User**

## RabbitMQ Exchange/Queue creation.
Use rabbitmq UI to login and create exchange/queue if its easier for you.
You will need to enable management plugin in case if you cannot access the UI.
rabbitmq-plugins.bat enable rabbitmq_management

Once you have the rabbitmq service running.

The script can be found on management site on url. - http://localhost:15672/cli/rabbitmqadmin 

Once the script is downloaded (in my case it is saved as rabbitmqadmin.py)

`python rabbitmqadmin.py declare exchange name=pawan-logger type=topic -u guest -p guest`

`python rabbitmqadmin.py declare queue  name=pawan-queue auto_delete=false durable=true -u guest -p guest`

`python rabbitmqadmin.py declare binding source=pawan-logger destination=pawan-queue routing_key=pawan -u guest -p guest`

Now the rabbitmq setup is ready. If you change any of above exchange/queue/key name ensure further steps also are reflected with the same.

## LogStash Configuration to receive from RMQ.
Navigate to the logstash directory.
1. Create a configuration file with name logstash-rabbitmq.conf. Paste below contents in it.

```
input {
        rabbitmq {
        host => "localhost"
        queue => "pawan-queue"
        heartbeat => 30
        durable => true
        password => "guest"
        user => "guest"
    }
}
filter {

    grok {
        match => { "message" => "TimeStamp=%{TIMESTAMP_ISO8601:logdate} CorrelationId=%{UUID:correlationId} Level=%{LOGLEVEL:logLevel} Message=%{GREEDYDATA:logMessage}" }
    }

        date {
        match => [ "logdate", "yyyy-MM-dd HH:mm:ss.SSSS" ]
    }

        date {
        match => [ "logdate", "yyyy-MM-dd HH:mm:ss.SSSS" ]
                target => "logdate"
    }

}
output {
    elasticsearch {
                hosts => "localhost:9200"
        }
    stdout {}
}
```

2. Once saved, run the logstash with below command
logstash -f logstash-rabbitmq.conf

As the logstash starts, you should see some logs like below.

```
[2020-11-18T12:25:33,600][INFO ][logstash.outputs.elasticsearch][main] New Elasticsearch output {:class=>"LogStash::Outputs::ElasticSearch", :hosts=>["//localhost:9200"]}
[2020-11-18T12:25:33,657][INFO ][logstash.outputs.elasticsearch][main] Using a default mapping template {:es_version=>7, :ecs_compatibility=>:disabled}
[2020-11-18T12:25:33,720][INFO ][logstash.outputs.elasticsearch][main] Attempting to install template {:manage_template=>{"index_patterns"=>"logstash-*", "version"=>60001, "settings"=>{"index.refresh_interval"=>"5s", "number_of_shards"=>1, "index.lifecycle.name"=>"logstash-policy", "index.lifecycle.rollover_alias"=>"logstash"}, "mappings"=>{"dynamic_templates"=>[{"message_field"=>{"path_match"=>"message", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false}}}, {"string_fields"=>{"match"=>"*", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false, "fields"=>{"keyword"=>{"type"=>"keyword", "ignore_above"=>256}}}}}], "properties"=>{"@timestamp"=>{"type"=>"date"}, "@version"=>{"type"=>"keyword"}, "geoip"=>{"dynamic"=>true, "properties"=>{"ip"=>{"type"=>"ip"}, "location"=>{"type"=>"geo_point"}, "latitude"=>{"type"=>"half_float"}, "longitude"=>{"type"=>"half_float"}}}}}}}
[2020-11-18T12:25:33,874][INFO ][logstash.javapipeline    ][main] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>12, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50, "pipeline.max_inflight"=>1500, "pipeline.sources"=>["/usr/local/Cellar/logstash-full/7.9.2/bin/logstash-rabbit.conf"], :thread=>"#<Thread:0x3d893bc2 run>"}
[2020-11-18T12:25:34,624][INFO ][logstash.javapipeline    ][main] Pipeline Java execution initialization time {"seconds"=>0.75}
[2020-11-18T12:25:34,714][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
[2020-11-18T12:25:34,790][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2020-11-18T12:25:34,978][INFO ][logstash.inputs.rabbitmq ][main][173c6ff6454aab84dfeb56c86a12a1083c7637022a49681e686a5b4a7e913c86] Connected to RabbitMQ at 
[2020-11-18T12:25:35,083][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```
If you see similar logs as above with Success message and correct port. You are good to go. ! 

## Start ElasticSearch.
Since I use brew on my mac
brew services start elasticsearch-full

## Kibana Configuration.

1. Navigate to the Kibana directory and find the kibana.yml file.
Un-comment the below line in the file.
elasticsearch.hosts: ["http://localhost:9200"]

2. Start Kibana
brew services start kibana-full

Once the Kibana starts, goto Home
Choose 
Use Elasticsearch data --> Connect to your Elasticsearch index --> Create Index Pattern

logstash-*
Time Filter field name: '@timestamp'

Create it with above pattern and filter.

3. Once above step is done.
Navigate to UI.


## Send Data from Postman/Services

At this part of stage you have almost finished the setup/configuration needed.
In Postman, Since its simple App to send the data/logs.
Use below request to send to RMQ and you should be able to see the corresponding on Kibana UI.

```
curl --location --request POST 'http://localhost:15672/api/exchanges/%2F/pawan-logger/publish' \
--header 'Authorization: Basic Z3Vlc3Q6Z3Vlc3Q=' \
--header 'Content-Type: application/json' \
--data-raw '{
  "vhost": "/",
  "name": "pawan-logger",
  "properties": {
    "delivery_mode": 2,
    "headers": {}
  },
  "routing_key": "pawan",
  "delivery_mode": "2",
  "payload": "TimeStamp=2020-11-01 00:13:01.1669 CorrelationId=68350588-9e3b-59c2-ggp1-31823d891c24 Level=INFO Message=Request completed with status code: 200",
  "headers": {},
  "props": {},
  "payload_encoding": "string"
}'
```

You can also view this index created under 

http://localhost:9200/_cat/indices?v

http://localhost:9200/logstash-2020.11.18-000001/_search?pretty

You can find Relevant Screenshots in the directory. 
