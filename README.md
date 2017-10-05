# Consul ACL client tutorial

## Configure master token

First, generate master token with `uuidgen`:

```
$ uuidgen
FBAF54CC-E03D-4763-9F19-376114D3857B
```

Set `acl_master_token` field with this value in `config/consul.json` file:

```
{
  "datacenter": "scaleway-paris",
  "data_dir": "/consul/data/",
  "log_level": "INFO",
  "node_name": "node1",
  "server": true,
  "bootstrap_expect": 1,
  "client_addr": "0.0.0.0",
  "ui": true,
  "acl_datacenter": "scaleway-paris",
  "acl_master_token": "FBAF54CC-E03D-4763-9F19-376114D3857B",
  "acl_default_policy": "deny",
  "acl_down_policy": "extend-cache",
  "acl_agent_token": "f710a920-bb12-b356-ea1f-80f85f88f80b"
}
```

For greater comfort with next cli operations, put this token in `master.token` file:

```
$ echo "FBAF54CC-E03D-4763-9F19-376114D3857B" > master.token
```


## Start consul server

Now you are ready to start first *consul server*.

First, download all images:

```
$ docker-compose pull
```

Start `consul1` server:

```
$ docker-compose up consul1
Starting testconsul_consul1_1 ...
Starting testconsul_consul1_1 ... done
Attaching to testconsul_consul1_1
consul1_1     | ==> WARNING: BootstrapExpect Mode is specified as 1; this is the same as Bootstrap mode.
consul1_1     | ==> WARNING: Bootstrap mode enabled! Do not enable unless necessary
consul1_1     | ==> Starting Consul agent...
consul1_1     | ==> Consul agent running!
consul1_1     |            Version: 'v0.9.3'
consul1_1     |            Node ID: 'da5b0f53-41a4-6710-5e37-86d40e62c0b6'
consul1_1     |          Node name: 'node1'
consul1_1     |         Datacenter: 'scaleway-paris' (Segment: '<all>')
consul1_1     |             Server: true (Bootstrap: true)
consul1_1     |        Client Addr: 0.0.0.0 (HTTP: 8500, HTTPS: -1, DNS: 8600)
consul1_1     |       Cluster Addr: 192.168.192.2 (LAN: 8301, WAN: 8302)
consul1_1     |            Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false
consul1_1     |
consul1_1     | ==> Log data will now stream in as it occurs:
consul1_1     |
consul1_1     |     2017/10/05 11:32:11 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:192.168.192.2:8300 Address:192.168.192.2:8300}]
consul1_1     |     2017/10/05 11:32:11 [INFO] raft: Node at 192.168.192.2:8300 [Follower] entering Follower state (Leader: "")
consul1_1     |     2017/10/05 11:32:11 [INFO] serf: EventMemberJoin: node1.scaleway-paris 192.168.192.2
consul1_1     |     2017/10/05 11:32:11 [INFO] serf: EventMemberJoin: node1 192.168.192.2
consul1_1     |     2017/10/05 11:32:11 [INFO] consul: Handled member-join event for server "node1.scaleway-paris" in area "wan"
consul1_1     |     2017/10/05 11:32:11 [INFO] consul: Adding LAN server node1 (Addr: tcp/192.168.192.2:8300) (DC: scaleway-paris)
consul1_1     |     2017/10/05 11:32:11 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
consul1_1     |     2017/10/05 11:32:11 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
consul1_1     |     2017/10/05 11:32:11 [INFO] agent: Started HTTP server on [::]:8500
consul1_1     |     2017/10/05 11:32:18 [ERR] agent: failed to sync remote state: No cluster leader
consul1_1     |     2017/10/05 11:32:21 [WARN] raft: Heartbeat timeout from "" reached, starting election
consul1_1     |     2017/10/05 11:32:21 [INFO] raft: Node at 192.168.192.2:8300 [Candidate] entering Candidate state in term 2
consul1_1     |     2017/10/05 11:32:21 [INFO] raft: Election won. Tally: 1
consul1_1     |     2017/10/05 11:32:21 [INFO] raft: Node at 192.168.192.2:8300 [Leader] entering Leader state
consul1_1     |     2017/10/05 11:32:21 [INFO] consul: cluster leadership acquired
consul1_1     |     2017/10/05 11:32:21 [INFO] consul: New leader elected: node1
consul1_1     |     2017/10/05 11:32:21 [INFO] consul: Created ACL master token from configuration
consul1_1     |     2017/10/05 11:32:21 [INFO] consul: ACL bootstrap disabled, existing management tokens found
consul1_1     |     2017/10/05 11:32:21 [INFO] consul: member 'node1' joined, marking health alive
consul1_1     |     2017/10/05 11:32:22 [ERR] agent: failed to sync remote state: ACL not found
consul1_1     |     2017/10/05 11:32:34 [ERR] agent: Coordinate update error: ACL not found
```

After the servers are restarted above, you will see new errors in the logs of the Consul servers related to permission denied errors:

```
consul1_1     |     2017/10/05 11:32:22 [ERR] agent: failed to sync remote state: ACL not found
consul1_1     |     2017/10/05 11:32:34 [ERR] agent: Coordinate update error: ACL not found
```

These errors are because the agent doesn't yet have a properly configured `acl_agent_token` that it can use for its own
internal operations like updating its node information in the catalog and performing anti-entropy
syncing ([more information](https://www.consul.io/docs/guides/acl.html#create-an-agent-token)).

Use *consul-cli* to create Agent Token:

```
$ docker-compose run --rm consul-cli acl --token=`cat master.token` create --name="node1 agent" --rule="node::write" --rule="service::read" | tr -d "\r" > node1.token
```

Note: here I use my [harobed/consul-cli](https://hub.docker.com/r/harobed/consul-cli/) Docker image to use *consul-cli* version `0.5.0` because
I need this patch [Add node rule ACL support (fix #49) #51](https://github.com/mantl/consul-cli/pull/51) to set `node` ACL rules.

Check configuration:

```
$ docker-compose run --rm consul-cli acl --token=`cat master.token` list
[
  {
    "CreateIndex": 10,
    "ModifyIndex": 10,
    "ID": "3c694360-9a6d-9609-cd95-d98d7536418c",
    "Name": "node1 agent",
    "Type": "client",
    "Rules": "{\"node\":{\"\":{\"Policy\":\"write\"}},\"service\":{\"\":{\"Policy\":\"read\"}}}"
  },
  {
    "CreateIndex": 5,
    "ModifyIndex": 5,
    "ID": "FBAF54CC-E03D-4763-9F19-376114D3857B",
    "Name": "Master Token",
    "Type": "management",
    "Rules": ""
  },
  {
    "CreateIndex": 4,
    "ModifyIndex": 4,
    "ID": "anonymous",
    "Name": "Anonymous Token",
    "Type": "client",
    "Rules": ""
  }
]
```

We can now add this to our Consul server configuration:

```
$ curl --request PUT --header "X-Consul-Token: `cat master.token`" --data '{"Token": "'`cat node1.token`'"}' http://127.0.0.1:8500/v1/agent/token/acl_agent_token
```

After few seconds, we can see in *consul1* logs:

```
consul1_1     |     2017/10/05 14:00:15 [INFO] Updated agent's ACL token "acl_agent_token"
consul1_1     |     2017/10/05 14:00:37 [INFO] agent: Synced node info
```

## Explore Consul Admin UI

Browse http://127.0.0.1:8500/ui/

```
$ open -a firefox http://127.0.0.1:8500/ui/
```

You can set *ACL Master token* in Settings (http://127.0.0.1:8500/ui/#/settings) page.


## Create other agent client token to read / write keys values

Create *bob* and *alice* client agent tokens, and allow write access in their namespaces:

```
$ docker-compose run --rm consul-cli acl --token=`cat master.token` create --name="bob" --rule="key:bob:write" | tr -d "\r" > bob.token
$ docker-compose run --rm consul-cli acl --token=`cat master.token` create --name="alice" --rule="key:alice:write" | tr -d "\r" > alice.token
```

## Write and read datas

*bob* can write in *bob* namespace but not in *alice* namespace:

```
$ docker-compose run --rm consul-cli kv --token=`cat bob.token` write bob/foo bar
$ docker-compose run --rm consul-cli kv --token=`cat bob.token` write alice/foo bar
Unexpected response code: 403 (Permission denied)
$ docker-compose run --rm consul-cli kv --token=`cat alice.token` write alice/foo bar
```

*master token* can read all key records:

```
$ docker-compose run --rm consul-cli kv --token=`cat master.token` read --recurse / --format prettyjson
[
  {
    "Key": "alice/foo",
    "CreateIndex": 83,
    "ModifyIndex": 83,
    "LockIndex": 0,
    "Flags": 0,
    "Value": "YmFy",
    "Session": ""
  },
  {
    "Key": "bob/foo",
    "CreateIndex": 59,
    "ModifyIndex": 78,
    "LockIndex": 0,
    "Flags": 0,
    "Value": "YmFy",
    "Session": ""
  }
]
```

*bob* can only read its records:

```
$ docker-compose run --rm consul-cli kv --token=`cat bob.token` read --recurse / --format prettyjson
[
  {
    "Key": "bob/foo",
    "CreateIndex": 59,
    "ModifyIndex": 78,
    "LockIndex": 0,
    "Flags": 0,
    "Value": "YmFy",
    "Session": ""
  }
]
```
