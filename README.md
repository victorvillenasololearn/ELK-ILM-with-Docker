# ELK-ILM-with-Docker
Build a ELK stack instance with multiple nodes for managing the indexes lifecycle

To simulate a multi-tier architecture with nodes playing different roles, we have a docker-compose stack that initializes three Elasticsearch containers and one Kibana. Each Elasticsearch container will be configured wth appropriate node roles, as follow:

- es01: master, data_dontent, data_hot
- es02: master, data_warm
- es03: master, data_cold
- es04: data_frozen

For each Elasticsearch container there is an environment variable set that defines which role the Elasticsearch service should play.
All services play a data role as well as the master role, to not to have to bring up dedicated master eligible nodes. In a real scenario you'd probably want to have dedicated master eligible nodes and separated data nodes.

![image](https://user-images.githubusercontent.com/99866081/168020931-c700dd04-673a-4466-81e1-fc76f04629fe.png)

- *Hot:* The index is actively being updated and queried.
- *Warm:* The index is no longer being updated but is still being queried.
- *Cold:* The index is no longer being updated and is queried infrequently. The information still needs to be searchable, but it’s okay if those queries are slower.
- *Frozen:* The index is no longer being updated and is queried rarely. The information still needs to be searchable, but it’s okay if those queries are extremely slow.
- *Delete:* The index is no longer needed and can safely be removed.

### Lets start with the steps to run the stack:

1.- Run `docker-compose up -d` Be sure you are in the same directory as the docker-compose.yml file.

2.- If everything went fine, you should be able to access Kibana instance by going to the following url: http://localhost:5601


### Defining an automatic ILM policy

The policy that we are going to be implementing for this demo, will be strictly guided by the age of our indices. We want them to migrate from tier to tier as they get older and older. Considering a real life policy, an index should migrate from *hot* to *warm* after 1 day, from *warm* to *cold* after 30 days and from *cold* to *frozen* after 365 days. When it reaches the *frozen* phase the index could also be deleted (since it will be searchable by mounting it from the snapshots). Finally, a rollover action can be added in the *hot* phase so a new write index is created every hour.

For the sake of this demo, lets consider the following time adaptations (so we can see everything happening in less than 5 minutes, instead of 365 days):
- Rollover every *hour* -> every *30 seconds*
- Migrate from hot to warm after *1 day* -> after *1 minute*
- Migrate from warm to cold after *30 days* -> after *2 minutes*
- Migrate from cold to frozen and then deleted after *365 days* -> after *3 minutes*


### Lets go with the ILM policy

1.- Lets define a snapshot policy

`PUT _slm/policy/orders-snapshot-policy
{
  "schedule": "0 * * * * ?", 
  "name": "<orders-snapshot-{now d}="">", 
  "repository": "orders-snapshots-repository", 
  "config": { 
    "indices": ["orders"] 
  },
  "retention": { 
    "expire_after": "30d", 
    "min_count": 5, 
    "max_count": 50 
  }
`

2.- Create an ILM policy to manage our *orders* indices

`
PUT _ilm/policy/orders-lifecycle-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "30s"
          }
        }
      },
      "warm": {
        "min_age": "1m",
        "actions": {}
      },
      "cold": {
        "min_age": "2m",
        "actions": {}
      },
      "frozen": {
        "min_age": "3m",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "orders-snapshots-repository"
          }
        }
      },
      "delete": {
        "min_age": "3m",
        "actions": {
          "wait_for_snapshot": {
            "policy": "orders-snapshot-policy"
          },
          "delete": {}
        }
      }
    }
  }
}
`

3.- By default ILM checks every **10 minutes** if there's any action to execute. As in our demo we use very small intervals of time, we have to configure a transient cluster parameter reducing the interval in whch ILM checks for the need of executing actions. Run the following command before moving on:

`
PUT _cluster/settings
{
  "transient": {
    "indices.lifecycle.poll_interval": "15s" 
  }
}
`

4.- Then we need to assign an ILM policy to an index. We can do that by defining the *index.lifecycle.name* attribute as the ILM policy name in the index's settings:

`
PUT orders-20211020/_settings
{
  "index.lifecycle.name": "orders-lifecycle-policy”
}
`

5.- We want this policy to be applied for every *order* index that is created, but we don't want to have to do t manually. The proper way of solving ths is to make use of and **index template**. With an index template you can define some common settings and mappings that should be applied to every index that is created and that the name matches a pattern defined in the template.
To create our index template we first need to create the components of the template and then use them when we finally create the template. Run the commands below to create settings and mappings components:

`
PUT _component_template/orders-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "orders-lifecycle-policy",
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  },
  "_meta": {
    "description": "Settings for orders indices"
  }
}
`

`
PUT _component_template/orders-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date",
          "format": "date_optional_time||epoch_millis"
        },
        "client_id": {
          "type": "integer"
        },
        "created_at": {
          "type": "date"
        }
      }
    }
  },
  "_meta": {
    "description": "Mappings for orders indices"
  }
}
`

Now run the following command to create the index template for the *orders* indices that wil lget created by the data stream:

`
PUT _index_template/orders
{
  "index_patterns": ["orders"],
  "data_stream": { },
  "composed_of": [ "orders-settings", "orders-mappings" ],
  "priority": 500,
  "_meta": {
    "description": "Template for orders data stream"
  }
}
`


6.- It's time to finally create our inbdex and check if all this wiring works. We have two options:
  - The first one is to crfeate the index manually with its name matching the pattern `^.*-\d+$` (for example orders-000001) and then create an alias pointing to it (orders -> orders-000001). So all your applications would use only the alias and the ILM policy would be in chargeof rolling the underneath indices.
  - The second option is to use **data streams**, which were introduced in Elasticsearch **7.9** and makes it easier to implement this pattern where you have append-only data that will be spread across multiple indices and should be made available to the clients as a single named resource. When you create a data stream, Elasticsearch will actually create a hidden index to store the documents.

We will go with the second one. Remember that we already have an ILM policy in place and it is being executed by Elasticsearch every 15 seconds. Once we create our data stream things will start to move fast.

Lets create a data stream for the *orders* index:

`PUT _data_stream/orders`



7.- Now we can finally watch the terminal and see Elasticsearch applying our ILM policy. You should see the following events happening:

`
(00m00s: the data stream immediately creates write index that is allocated in the hot tier)
index                        node
.ds-orders-2021.10.07-000001 es01
 
(00m30s: the write index backing the data stream is rolled over, also allocated in the hot tier)
index                        node
.ds-orders-2021.10.07-000001 es01
.ds-orders-2021.10.07-000002 es01
 
*** Here we perform a snapshot manually, simulating what would've happened automatically through our SLM policy, but that we couldn't rely on for the purpose of this demo because the minimal interval you can schedule it is 15 minutes ***
 
(01m00s: index is rolled over one more time)
index                        node
.ds-orders-2021.10.07-000001 es01
.ds-orders-2021.10.07-000002 es01
.ds-orders-2021.10.07-000003 es01
 
(01m00s: also, the oldest index is migrated to the warm tier)
index                        node
.ds-orders-2021.10.07-000001 es02
.ds-orders-2021.10.07-000002 es01
.ds-orders-2021.10.07-000003 es01
 
(01m30s: one more roll over)
index                        node
.ds-orders-2021.10.07-000001 es02
.ds-orders-2021.10.07-000002 es01
.ds-orders-2021.10.07-000003 es01
.ds-orders-2021.10.07-000004 es01
 
(01m30s: the second index, now 1 minute old, is migrated to the warm tier)
index                        node
.ds-orders-2021.10.07-000001 es02
.ds-orders-2021.10.07-000002 es02
.ds-orders-2021.10.07-000003 es01
.ds-orders-2021.10.07-000004 es01
 
(02m00s: one more roll over and the first index, now 2 minutes old, is migrated to the cold tier)
index                        node
.ds-orders-2021.10.07-000001 es03
.ds-orders-2021.10.07-000002 es02
.ds-orders-2021.10.07-000003 es01
.ds-orders-2021.10.07-000004 es01
.ds-orders-2021.10.07-000005 es01
 
(02m30s: the second index, now 2 minutes old, is migrated to the cold tier and the third index, now 1 minute old, is migrated to the warm tier)
index                        node
.ds-orders-2021.10.07-000001 es03
.ds-orders-2021.10.07-000002 es03
.ds-orders-2021.10.07-000003 es02
.ds-orders-2021.10.07-000004 es01
.ds-orders-2021.10.07-000005 es01
 
(03m00s: one more roll over) 
index                        node
.ds-orders-2021.10.07-000001 es03
.ds-orders-2021.10.07-000002 es03
.ds-orders-2021.10.07-000003 es02
.ds-orders-2021.10.07-000004 es01
.ds-orders-2021.10.07-000005 es01
.ds-orders-2021.10.07-000006 es01
 
(03m30s: the fourth index, now 1 minute old, is migrated to the warm tier)
index                        node
.ds-orders-2021.10.07-000001 es03
.ds-orders-2021.10.07-000002 es03
.ds-orders-2021.10.07-000003 es02
.ds-orders-2021.10.07-000004 es02
.ds-orders-2021.10.07-000005 es01
.ds-orders-2021.10.07-000006 es01
 
(04m00s: the first index, now 4 minutes old, is deleted and the snapshot where it is backed up is mounted for search in the frozen tier)
index                                node
.ds-orders-2021.10.07-000002         es03
.ds-orders-2021.10.07-000003         es03
.ds-orders-2021.10.07-000004         es02
.ds-orders-2021.10.07-000005         es01
.ds-orders-2021.10.07-000006         es01
partial-.ds-orders-2021.10.07-000001 es04
 
(04m30s: one more roll over)
index                                node
.ds-orders-2021.10.07-000002         es03
.ds-orders-2021.10.07-000003         es03
.ds-orders-2021.10.07-000004         es02
.ds-orders-2021.10.07-000005         es01
.ds-orders-2021.10.07-000006         es01
.ds-orders-2021.10.07-000007         es01
partial-.ds-orders-2021.10.07-000001 es04
 
(05m00s: the fifth index, now 1 minute old, is migrated to the warm tier) 
index                                node
.ds-orders-2021.10.07-000002         es03
.ds-orders-2021.10.07-000003         es03
.ds-orders-2021.10.07-000004         es02
.ds-orders-2021.10.07-000005         es02
.ds-orders-2021.10.07-000006         es01
.ds-orders-2021.10.07-000007         es01
partial-.ds-orders-2021.10.07-000001 es04
 
(05m30s: the second index, now 4 minutes old, is deleted and the snapshot where it is backed up is mounted for search in the frozen tier)
index                                node
.ds-orders-2021.10.07-000003         es03
.ds-orders-2021.10.07-000004         es03
.ds-orders-2021.10.07-000005         es02
.ds-orders-2021.10.07-000006         es01
.ds-orders-2021.10.07-000007         es01
partial-.ds-orders-2021.10.07-000001 es04
partial-.ds-orders-2021.10.07-000002 es04
 
And so on...
`

Here you have an diagram of the chain of events that are performed:

![image](https://user-images.githubusercontent.com/99866081/168027119-4a46e942-66d9-4c67-8d56-25210a15ba87.png)
