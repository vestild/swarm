# Swarm: a JavaScript object sync library

Swarm is a minimal library that magically synchronizes JavaScript
objects between all your clients and servers, in real-time. All
instances of an object will form a dynamic graph, exchanging deltas
the most efficient way and merging changes.  Swarm needs Node.js and
[einaros ws][ws] on the server side and HTML5 WebSocket in the browser.

The vision behind Swarm is to get back to the good old days: program a
real-time multiuser multiserver app like a local MVC app, let things
automatically synchronize in the background. Ideally, once view
rendering is model event driven, there is hardly any difference
in reacting to either local or remote changes.

The Swarm is the M of MVC. Regarding V and C parts, you may either
attach your own View or Controller components (like [Handlebars][hb])
or insert Swarm into your MVC framework of choice (like
[Backbone.js][bb]).

[ws]: https://github.com/einaros/ws "Einar O. Stangvik WebSocket lib"
[hb]: http://handlebarsjs.com/ "Handlebars templating lib"
[bb]: http://backbonejs.org/ "Backbone.js MVC lib"

## Architecture

We target an objective described by some as the Holy Grail of
JavaScript real-time apps.  With HTML5 and node.js, server and client
sides converge: both run JavaScript, both have their own storage
and there is a continous two-way WebSocket connection between them.
The Holy Grail idea is to run the same codebase as needed either on
the server or on the client. In particular, we'd like to serve static
HTML from the server to make our site indexable. Further on, we'd like
to update HTML incrementally on the client. Once a user initiates an
event, we often need to relay that event to other users.  In bigger
setups, we also need to synchronize our multiple servers. Even at a
single server, multiple node.js worker processes often need some sort
of synchronization.

There are various ways of resolving those matters, which mostly boil
down to retrofitting the classic HTTP request-response architecture
which dates back to the epoch of Apache CGI, Perl scripts and RDBMS
backends. Instead, Swarm takes a fresh look at the problem, partially
by borrowing some pages from peer-to-peer design books. Servers and
clients form a single network to synchronize changes and relay events
without relying too much on the (database) backend.

We assume that both server and client processes are unreliable alike.
Server processes connect to each other in P2P fashion, forming a full
mesh.  Servers may routinely join and depart, but dusruptions to the
mesh are temporary. A client connects to a server of his choice by
WebSocket.

Once a client opens an object, the subscription comes to the server by
a WebSocket connection.  In multi-server setups, the server contacts
another server which is the 'root' for that object.  The 'root' is
assigned algorithmically, using consistent hashing.  The root server
conducts all DB reads and writes for the object to minimize DB access
concurrency. The current state of an object is sent back to the
client.  Further on, all replicas of the object at all the server and
client processes will have to synchronize.  Every change will be
serialized as a "diff" object. Diffs will propagate to all replicas by
a spanning tree where the root server will serve as the root
(tautology, right) and clients will be the leaves.  Normally, spanning
trees have depth of three (the root, servers, clients).  In case of
topology disruptions, the spanning tree may temporarily have more
tiers due to transient inconsistencies, e.g. while the responsible
node is not fully connected yet.  Once some clients stop listening to
an object, the spanning tree contracts to the necessary minimum in a
process that is reminiscent of garbage collection, albeit at the swarm
level.  Once no client is listening, all replicas are garbage
collected.

## API

Swarm is built around its concise four-method interface that expresses
the core function of the library: synchronizing distributed object
replicas. The interface is essentially a combination of two well
recognized conventions, namely get/set and on/off method pairs, also
available as getField/setField and addListener/removeListener calls
respectively.  The most common object lifecycle pattern is:

    var obj = swarmPeer.on(id, callbackFn); // also addListener()
    obj.set('field',value);
    obj.getField()===obj.get('field')===obj.field;
    obj.on('field', fieldCallbackFn);
    obj.off('field', fieldCallbackFn);
    swarmPeer.off(id, callbackFn);  // also removeListener()

Each set() call is essentially relayed to all the replicas of an object
triggering all the attached listeners.

Although the standard field-value signature is accepted, further down
the call chain it is generalized to fully unambiguously specify the
event in question.  Namely, the event/field name is generalized as a
*specifier*, a compound identifier containing the class of the object,
the object id, the field name and a version identifier. At the same
time, the value is generalized as a {field:value} map. In particular,
event listeners have to deal with the most "generalized" form of the
specifier-value signature:

    function fieldCallbackFn (spec, val) {
        spec.id === obj.id;
        val.field === obj.field === val[Peer.wireNames['field']];
    }


The document on [Specifiers][sp] has detailed information on specifiers and API signatures.

[sp]: Specifiers.html "Swarm: specifying events"

## Future work

* comment the mouse.js example extensively
* explicitly mark load state of an object, obj.\_state as a bitmask of
  EMPTY, READY, DIRTY...
* add WebStorage cache to fasten load, enable offline operation
* add derived/secondary fields calculated from regular state fields
  (kind of cache)
* add static event listeners, like onKeyChange(oldval,newval)
* add PEX ability to server processes
* add RPC functionality, still avoiding any oplog
