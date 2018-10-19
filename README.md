
## Some sample code for pushing metrics to graphite.

We will need a valid graphite URL/ server address {graphiteURL} and a UUID which is a WRITE APIKey to push metrics to graphite.

This is because we have built a graphite server that takes in metrics only if you have a valid UUID. Under a normal setup you can send metrics without UUID and you would have to block the metrics the server recieves only with IPTables which is a crude way to limit traffic to your service.

#### The simplest way is to talk to graphite via its raw socket, on port 2003. Here is a sample metric update:
```
$ telnet {graphiteURL} 2003
UUID.metric.that.you.want 5.63 1432809481\n
^]
exit
```
It really is that simple. Basically you open up a socket to the graphite server, write out the dot separated oid, a value, a timestamp and finally a new line. You can send more than one metric using the same connection, just start again after the newline.

####Shell example:
```
echo "UUID.metric.that.you.want 5.63 `date +%s`" | nc {graphiteURL} 2003
```

#### Python example (not ver 3+):
```
import socket
import time
connection = socket.socket()
connection.connect(('{graphiteURL}', 2003))

print "connected to graphite server"
connection.send('{getUUID from secretManager}.metric.that.you.want 5.63 {0}\n'.format(time.time()))
connection.close()
```

As you can see, its pretty simple. http://graphite.readthedocs.org/en/latest/feeding-carbon.html has more detail.

#### Ruby example:
```
require 'socket'

connection = TCPSocket.open('{graphiteURL}', 2003)
print "Connected to graphite server"

time = Time.new
epoch = time.to_i
connection.write('{getUUID from vault}.metric.that.you.want 5.63 #{epoch}\n")
connection.close
```

#### NodeJS example:
```
var net = require('net');
metric = "{getUUID from vault}.demo.metric " + parseFloat(30) + " " + Math.floor(Date.now() / 1000) + "\n";

const socket = net.createConnection(2003, '{graphiteURL}', function() {
});

const writeResponse = socket.write(metric, function () {
});

socket.end();
```

#### How should I name my metrics/OIDs?

See below for the detail, but if you just want to get some data into Graphite to play with it, use the sandbox UUID and that put it under the 'sandbox' top level:

echo "UUID.some.metric 1.234 `date +%s`" | nc {graphiteURL} 2003

This should show up in the tree view on the Graphite interface:

Graphite -> sandbox -> some -> metric 1.234 timestamp


The sandbox area will be cleaned out regularly, so don't leave anything here that you would be upset about losing.

OK, the detail... 

The naming hierarchy for OIDs is clearly important in terms of being able to find stuff, but it also plays a more crucial role. The location of an OID within the namespace also determines its retention settings - that is, how long the data for the OID is kept, and at what level of precision. Graphite keeps its data in Whisper databases, which are fixed in size when they are created by specifying how long data is kept and at what level of precision.

Currently, most data within Graphite will use this retention:

```
[default]
retentions = 60s:7d
```

What does this mean?

60s:7d - we will aggregate and keep a datapoint every 60 seconds, and we want to configure Whisper(graphite database) with enough space to keep 7 days' worth of data

The good news is that retentions are super configurable, so subsets of the OID namespace can be configured to use a different retention. So you might choose to have:

* a higher precision (e.g. keeping a data point every 10 seconds instead of every 60 seconds)
* a larger number of data points (allowing you to keep data for a longer period of time)
* a different downsizing mechanism (so instead of averaging out the data, you could take the minimum or maximum values from the data set)

There's no right answer - it depends on the nature of the data, how often you are pushing it into Graphite, and how you want it to be represented on your charts.