# MongoDB packet capture

TLDR: A method for extracting request/response couples from the mongodb wire protocol, without changing your application. Shows how to deal with the produced data. 

(08:24) Idea: If we could sniff mongodb traffic and capture packets, we would be able to know all the queries we do (!) and their latencies. There is a tool for this in mongodb 3.4: https://docs.mongodb.com/manual/reference/program/mongoreplay/

```shell
$ ./mongoreplay monitor -i lo0 \
	--collect json | jq
```

Cool. I run the tests (core:components/api-server) using this:

```shell
$ ./mongoreplay monitor -i lo0 --collect json | jq '.' | tee test.json
```

Now let's filter this down to something we can actually look at (lots of output, chatter about a cluster I don't have.)

```shell
$ cat test.json|  jq 'select(.command != "ismaster") | select(.reply_data.ismaster != true)' | tee 001.json | bat
```

(08:56) The disadvantage of jq is that it mostly operates on single documents. To be able to better filter, let's aggregate request and response in a top-level document that looks like this: 

```js
{
  req: ..., 
  res: ..., 
}
```

Here's the script I use for this: 

```ruby
#!/usr/bin/env ruby

# Takes a filename on the command line; reads that file as a stream of JSON
# documents. Each document should contain a 'request_id' and 'op', which is 
# either 'command' or 'reply'. Outputs another stream of JSON documents, 
# aggregating replies for requests. 

require 'yajl' # stream parser
require 'pp'   # pretty printer

def id_to_ary_map 
  Hash.new { |h, k| h[k] = [] }
end

filename = ARGV.first
io = File.open(filename, 'r')

# Map<rqid, Array<obj>>
requests = id_to_ary_map()
# Map<rqid, Array<obj>>
responses = id_to_ary_map()

parser = Yajl::Parser.new
parser.parse(io) do |obj|
  rqid = obj["request_id"]
  conn_num = obj["connection_num"]

  key = [conn_num, rqid]

  unless rqid && conn_num
    pp obj
    raise "No request id?"
  end

  req = requests[key]
  res = responses[key]
  
  case obj["op"]
    when 'command', 'delete', 'insert', 'update'
      req << obj
    when 'reply'
      res << obj
  else
    raise "Unknown op: #{obj['op']}."
  end

  if req.length == 1 && res.length == 1
    # Flush the document:
    doc = {
      req: req.first, 
      res: res.first,
    }

    Yajl::Encoder.encode(doc, $stdout)
    puts
  end

  if req.length > 1
    pp req
    raise "Too many requests for this request id."
  end

  if res.length > 1
    pp res
    raise "Too many responses for this request id."
  end
end
```

(09:27) Looking at this, we now know quite a bit more about the structure of this log. 

Let's solidify this log:

```shell
$ ./aggregate.rb 001.json | jq '.' > 002.json
```

(09:28) Let's look at high-latency interactions:

```shell
$ cat 002.json| jq 'select(.res.latency_us > 1000) | .req.op' | wc -l
26
```

Only 26 requests take more than 1000 us! Let's look at these.

```shell
$ cat 002.json| jq 'select(.res.latency_us > 1000)' | jq '.' > 003.json
```

Here's what the first request from this file looks like: 

```json
{
  "req": {
    "order": 17,
    "op": "command",
    "command": "createIndexes",
    "ns": "pryv-node.$cmd",
    "request_data": {
      "createIndexes": "cjojprlvm00000xjn8f2fuyxr.accesses",
      "indexes": [
        {
          "background": true,
          "key": {
            "token": 1
          },
          "name": "token_1",
          "sparse": true,
          "unique": true
        }
      ],
      "writeConcern": {
        "w": 1
      }
    },
    "reply_data": null,
    "connection_num": 7,
    "seen": "2018-11-16T08:40:29.740195+01:00",
    "request_id": 10
  },
  "res": {
    "order": 18,
    "op": "reply",
    "request_data": null,
    "reply_data": {
      "createdCollectionAutomatically": true,
      "numIndexesAfter": 2,
      "numIndexesBefore": 1,
      "ok": 1
    },
    "nreturned": 1,
    "connection_num": 7,
    "latency_us": 218992,
    "seen": "2018-11-16T08:40:29.959187+01:00",
    "request_id": 10
  }
}
```

These are some basic summary statistics on what remains in the file, latencies are in ms. 

```
createIndexes 219 222 220 65 65 65 72 73 87 96 27 30 19 21 22 22 17
drop 51 74 
insert(accesses) 41 38 41 38 37 
insert(streams) 81
update(sessions) 19
```

(09:40) Not interesting by itself; keep the method, apply to a real project. 