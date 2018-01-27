# Some useful command-line for Opsview via REST API

## Fetch all the hosts matching the criteria you need.
Search for all hosts that have names starting with 'host' (e.g. host1, host2, host3), you could:
```
root# opsview_rest --username=XXXX--password=YYYY --pretty GET config/host?json_filter='{"name":{"-like":"host%25"}}'
```
where the %25 is the URL encoded value for a %, the SQL wildcard character.

Search for all hosts that in hostgroup `HOSTGROUP`:

```
opsview_rest --username=XXXX --password=YYYY --data-format=json --pretty GET config/host"?&s.hostgroup.name=HOSTGROUP&rows=all"
```

## Delete host using id field:
```
$ opsview_rest --username=XXXX--password=YYYY --pretty DELETE config/host/5
```


