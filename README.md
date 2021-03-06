# fash: consistent hashing library for node.js

[![NPM](https://nodei.co/npm/fash.png)](https://nodei.co/npm/fash/)
[![NPM](https://nodei.co/npm-dl/fash.png)](https://nodei.co/npm/fash/)


This module provides a consistent hashing library. Notably, this module has the
ability to **deterministically generate the same hash ring topology across a
set of distributed hosts**. Fash also handles collisions of nodes on the ring
and ensures no two nodes will share the same spot on the ring. Additionally,
fash provides the ability to add, remove, or remap physical nodes on the ring
-- useful when a particular physical node has hit its scaling bottleneck.

<a name="index"></a>
# Index

* [Design Summary](#design-summary)
* [Examples for Consumers](#examples-for-consumers)
    - [Bootstrapping a New Hash Ring](#bootstrapping-a-new-hash-ring)
    - [Adding More Pnodes to the Ring](#adding-more-pnodes-to-the-ring)
    - [Adding Optional Data to a Virtual Node](#adding-optional-data-to-a-virtual-node)
* [Development Information](#development-information)
    - [Command Line Ring Creation](#command-line-ring-creation)
        * [Backends: an interlude](#backends-an-interlude)
    - [Walkthrough: working with the hash ring](#walkthrough-working-with-the-hash-ring)
        * [Adding Data to Vnodes](#adding-data-to-vnodes)
        * [Remapping Vnodes](#remapping-vnodes)
        * [Removing Pnodes](#removing-pnodes)
    - [LevelDB schema](#leveldb-schema)
    - [LevelDB-supported CLI](#leveldb-supported-cli)
    - [Running Tests](#running-tests)

# Design Summary

Fash consists of a mapping of a set of fixed virtual nodes (vnodes) -- usually
a large number, say 1000000 -- distributed across the hash ring. It is then
possible to map these virtual nodes to a set of physical nodes (pnodes).  In
practice, pnodes are usually physical shards or servers in a distributed
system. This gives the flexibility of mutating the hashspace of pnodes and the
number of pnodes by re-mapping the vnode assignments.  For a more in-depth
explanation, see the [Development Information](#development-information) section.

# Examples for Consumers

Examples can be found in the unit tests. Here are a few.

## Bootstrapping a New Hash Ring

    var fash = require('fash');
    var Logger = require('bunyan');

    var LOG = new Logger({
        name: 'fash',
        level: 'info'
    });

    var chash = fash.create({
        log: LOG, // optional [bunyan](https://github.com/trentm/node-bunyan) log object.
        algorithm: 'sha-256', // Can be any algorithm supported by openssl.
        pnodes: ['A', 'B', 'C', 'D', 'E'], // The set of physical nodes to insert into the ring.
        vnodes: 1000000 // The virtual nodes to place onto the ring. Once set, this can't be changed for the lifetime of the ring.
        backend: fash.BACKEND.IN_MEMORY
    });

    var node = chash.getNode('someKeyToHash');
    console.log('key hashes to pnode', node);

If the config used to bootstrap fash is the same across all clients, then the
ring toplogy will be the same as well. By default, fash will evenly distribute
vnodes across the set of pnodes. If you wish to have a custom mapping of pnodes
to vnodes, see the later section on serialization.

## Remapping Vnodes in the Ring
Fash gives you the ability to add and rebalance the vnodes in the ring by using
the remapVnode() function, which returns an optional callback.

You can also remove pnodes from the ring, but **you must first rebalance the
ring by reassigning its vnodes to other pnodes** via remapVnode(). Then you can
invoke removePnode(), which will return an optional callback.

You can assign an arbitrary number of vnodes to the new pnode -- also -- the
pnode can be a new node, or an existing one.  Again, as long as the order of
removes and remaps is consistent across all clients, the ring toplogy will be
consistent as well.

    var fash = require('fash');
    var Logger = require('bunyan');

    var LOG = new Logger({
        name: 'fash',
        level: 'info'
    });

    var chash = fash.create({
        log: LOG,
        algorithm: 'sha256',
        pnodes: ['A', 'B', 'C', 'D', 'E'],
        backend: fash.BACKEND.IN_MEMORY,
        vnodes: 100000
    });

    // get vnodes from A
    var aVnodes = chash.getVnodes('A');
    aVnodes = aVnodes.slice(aVnodes.length / 2);

    // remap some of A's vnodes to B
    chash.remapVnode('B', aVnodes, function(ring, pnodes) {
        console.log('new ring topology', ring);
        console.log('changed pnode->vnode mappings', pnodes);
    });

## Adding More Pnodes to the Ring

You can add additional pnodes to the ring after fash has been initialized by
invoking remapVnode(), which optionally returns a callback. Note that adding
the callback will cause fash to create a new copy of the ring topology across
each invocation -- do not do this if you have millions of vnodes, as this is
quite slow and may consume all available memory, bringing the operation to a
standstill and locking up other resources.

    var fash = require('fash');
    var Logger = require('bunyan');

    var LOG = new Logger({
        name: 'fash',
        level: 'info'
    });

    var chash = fash.create({
        log: LOG,
        algorithm: 'sha256',
        pnodes: ['A', 'B', 'C', 'D', 'E'],
        backend: fash.BACKEND.IN_MEMORY,
        vnodes: 100000
    });
    // specify the set of virtual nodes to assign to the new physical node.
    var vnodes = [0, 1, 2, 3, 4];

    // add the physical node 'F' to the ring.
    chash.remapVnode('F', vnodes, function(ring, changedNodes) {
        console.log('ring topology updated', ring);
        console.log('removed mappings', changedNodes);
    });

Fash will remove the vnodes from their previously mapped physical nodes, and
map them to the new pnode.

## Removing Pnodes from the Ring
You can remove physical nodes from the ring by first remapping the pnode's
vnodes to another pnode, and then removing the pnode.

    var fash = require('fash');
    var Logger = require('bunyan');

    var LOG = new Logger({
      name: 'fash',
      level: 'info'
    });

    var chash = fash.create({
        log: LOG,
        algorithm: 'sha256',
        pnodes: ['A', 'B', 'C', 'D', 'E'],
        backend: fash.BACKEND.IN_MEMORY,
        vnodes: 10000
    });

    // get the vnodes that map to B
    var vnodes = chash.getVnodes('B');
    // rebalance them to A
    chash.remapVnode('A', vnodes, function(ring, removedMap) {
        // remove B
        chash.removePnode('B', function(ring, pnode) {
            if (!err) {
                console.log('removed pnode %s', pnode);
            }
        });
    });

## Adding Optional Data to a Virtual Node

Sometimes, it might be helpful to associate state with a set of vnodes. An
example of this would be during routine system maintenance, an administrator
may want to set a certain set of vnodes to read only, and then set them back to
write mode after the maintenance has completed.

Fash gives enables you to add arbitrary objects to vnodes by invoking the
`addData` function.

    chash.addData(10, 'foo');

Subsequence `chash.getNode()` invocations which map to vnode 10 will return:

    {
        pnode: 'A',
        vnode: 10,
        data: 'foo'
    }

The data associated with a virtual node is persistent across serializations and
remaps.

## Serializing and Persisting the Ring Toplogy
At any time, the ring toplogy can be accessed by:

    chash.serialize();

Which returns the ring topology, which is a JSON serialized object string which
looks like

    {
        pnodeToVnodeMap: {
            A: {0, 1, 2},
            ...
        }, // the pnode to vnode mapings.
        vnode // the total number of vnodes in the ring.
    }

Additionally, anytime remapVnode() is invoked, it can return a cb which
contains an updated version of this object that is **not** JSON serialized.
Fash can be instantiated given a topology object instead of a list of nodes.
This also allows you to specify a custom pnode to vnode topology -- as
mentioned in the earlier bootstrapping section.

    var fash = require('fash');
    var Logger = require('bunyan');

    var LOG = new Logger({
        name: 'fash',
        level: 'info'
    });

    var chash = fash.create({
       log: LOG,
       algorithm: 'sha256',
       pnodes: ['A', 'B', 'C', 'D', 'E'],
       backend: fash.BACKEND.IN_MEMORY,
       vnodes: 10000
    });

    var topology = chash.serialize();

    var chash2 = fash.deserialize({
        log: LOG,
        topology: topology
    });

That's it, chash and chash2 now contain the same ring toplogy.

# Development Information

The following information is intended to serve developers making changes or
enhancements to this repository.  The command-line interface provides insight
into the structure of the hash ring topology at a deeper level than is made
available by the functions exposed for consumers of the node program.  Though
`help` can be a good reference once the data structure is understood, this
documentation seeks to give an overview of this structure more comprehensively
than makes sense to include in the tool itself.

The "data structure" is, at a high level, a mapping of pnodes or "physical
nodes" to vnodes or "virtual nodes."  It is intended to provide database-level
associations for an operator's (your) key-value store.  The advantage of this
setup is the flexibility to move vnodes between pnodes, while the association
between the key-value pairings (where the "key" is hashed to an integer value
and the "value" is a piece of plaintext data) and their assigned vnode is
preserved.  This gives users the ability to retain relative groupings of keys
(such as entries in the same file directory) that you've stored even in the
event that you need to move data around.  For example, if the data set grows
large enough that additional pnodes are necessary to contain it all, this would
necessitate re-assignment of key-value pairings as some are moved to the new
pnode.  The level of abstraction the vnodes enable makes it possible to keep
different key-value pairings associated with one another in a granular and
simple way.

## Command Line Ring Creation

Here is the creation of a LevelDB-backed (meaning the key-value store is
represented by the specs of LevelDB -- more information on that can be found [here](#leveldb-schema).
This is a hash ring for distributing 6 vnodes throughout 2 pnodes using the
default hashing algorithm, sha256, which will be stored at the location `/var/tmp/rings/readme_sample`.

    $ ./bin/fash.js create -l /var/tmp/rings/readme_sample -v 6 -a sha256 -p "tcp://1.moray.emy-11.joyent.us:2020, tcp://2.moray.emy-11.joyent.us:2020" -b leveldb

Here is a <a name="diagram">diagram</a> of what that ring would then look like:

```
                   PNODES                                       VNODES

                                                             +-----------+
+-------------------------------------------+   +--------+   |  "0": 1   |
|                                           |                +-----------+
|                                           |
|   "tcp://1.moray.emy-11.joyent.us:2020"   |                +-----------+
|                                           |   +--------+   |  "2": 1   |
|                                           |                +-----------+
|                                           |
+-------------------------------------------+                +-----------+
                                                +--------+   |  "4": 1   |
                                                             +-----------+

                                                             +-----------+
+-------------------------------------------+   +--------+   |  "1": 1   |
|                                           |                +-----------+
|                                           |
|   "tcp://2.moray.emy-11.joyent.us:2020"   |                +-----------+
|                                           |   +--------+   |  "3": 1   |
|                                           |                +-----------+
|                                           |
+-------------------------------------------+                +-----------+
                                                +--------+   |  "5": 1   |
                                                             +-----------+
```

There is another way to create a ring topology if you already have a JSON file
representation of one.  If you want to manufacture one, you can use the [`print-hash`](#print-hash)
command to print a JSON representation of a ring after creating it from scratch
(as above).  Copying that into a `.json` file will give you the raw materials
for testing out the [`deserialize-ring`](#deserialize) method of ring creation
below.

<a name="sample-deserialize"></a>
This is what the `.json` file should look like:

    $ cat readme_sample.json
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1,"4":1},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

And then to actually generate and store that LevelDB-backed ring:

    $ ./bin/fash.js deserialize-ring -l /var/tmp/rings/readme_sample_deserialized -f /var/tmp/rings/readme_sample.json

To see that the deserialized ring is the same as the original, the new ring can
be viewed with the same `print-hash` command used to show the original.
<a name="sample-print"></a>

    $ ./bin/fash.js print-hash -l /var/tmp/rings/readme_sample_deserialized -b leveldb
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1,"4":1},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

The fash tool can also do a direct comparison on its own with `diff`:
<a name="sample-diff"></a>

    $ ./bin/fash.js diff -b leveldb /var/tmp/rings/readme_sample /var/tmp/rings/readme_sample_deserialized
    {"tcp://1.moray.emy-11.joyent.us:2020":{"removed":[0],"added":[0]},"tcp://2.moray.emy-11.joyent.us:2020":{"removed":[1],"added":[1]}}

### Backends: an interlude

Please note that for commands added before version 2.4.0 (of which `create` is
one), `-b` option has to be specified.  This command's purpose is to indicate
whether the topology is backed by a LevelDB or in-memory data store.

As of version 2, fash supported a LevelDB backend in addition to its original
in-memory backend. The LevelDB backend has several advantages over the
in-memory backend.  Notably, performance at scale should be faster since the
ring is no longer in v8.  Also the ring can be persisted on disk via LevelDB,
removing the need to load the ring into memory when the process is restarted.

For these reasons, at the time of writing, the **in-memory option** for
creating hash rings **is considered deprecated**, and may be removed in the
future, once any existing dependencies on it can be verified and mitigated.
That is why this section focuses on working with the LevelDB-backed hash ring
functionality, though it would be remiss not to explain why some commands will
not work without passing in `-b leveldb`.

The way to create a ring -- specifying a backend -- from the command line is
[demonstrated above](#command-line-ring-creation).  Choosing a backend as a
consumer is done during fash initialization via the `backend` field.

```javascript

    var fash = require('fash');
    var Logger = require('bunyan');

    var LOG = new Logger({
        name: 'fash',
        level: 'info'
    });

    fash.create({
        log: LOG, // optional [bunyan](https://github.com/trentm/node-bunyan) log object.
        algorithm: 'sha-256', // Can be any algorithm supported by openssl.
        pnodes: ['A', 'B', 'C', 'D', 'E'], // The set of physical nodes to insert into the ring.
        vnodes: 1000000 // The virtual nodes to place onto the ring. Once set, this can't be changed for the lifetime of the ring.
        backend: fash.BACKEND.LEVEL_DB,
        location: '/tmp/chash'
    }, function(err, chash) {
        console.log('chash created');
    });
```

## Walkthrough: working with the hash ring

Getting back to the [example topology created above](#diagram); in order to
understand how this is related to both the LevelDB key-value store and also
the pnode-vnode mapping, we will take just one "vnode" and break down the
information inside it.

                        +-----------+
                        | "4": 1    |
                        +-----------+

The `4` is the vnode number.  It is also the post-hash key in the sense of the
key-value LevelDB store.  What this mean is that if you have the key
`/yunong/yunong.txt`, and then hash it using one of the available algorithms --
say, sha256 -- you will get back, in this ring topology, the value `4`.  So
vnode `4` is where any data (value) you wanted to associate with the `/yunong/yunong.txt`
key will be stored.  In this case, we are storing the default value of 1 in
that vnode, associated with the key `/yunong/yunong.txt`.  To help illustrate
this point, there is a `get-node` command to show operators what any key is
associated with in terms of its data, pnode, and vnode mapping.
<a name="sample-node"></a>

    $ ./bin/fash.js get-node "/yunong/yunong.txt" -l /var/tmp/rings/readme_sample -b leveldb
    { pnode: 'tcp://1.moray.emy-11.joyent.us:2020',
      vnode: 4,
      data: 1 }

This operation is idempotent *as long as the topology does not change*. So `/yunong/yunong.txt`
will always map to vnode `4` UNLESS the number of pnodes or vnodes changes.
For this reason, remapping vnodes must *always be a manual process* -- we
cannot recreate the ring with more pnodes or vnodes and expect that `/yunong/yunong.txt`
will still map to vnode `4` on the `tcp://1.moray.emy-11.joyent.us:2020` pnode.

### Adding Data to Vnodes

So in an example where we are approaching full capacity on our two pnodes and
need to add a third, it might make sense to mark the vnodes we're going to move
in some way.  Then our application can check for that demarcation and, say, not
accept writes that might be lost during a move.  If the chosen identifer was `ro`
(for READ ONLY), we could assign the value `ro` to the vnodes with the keys
we're shifting over.  This is done with the [`add-data`](#add-data) command:

    $ ./bin/fash.js add-data -v 4 -d 'ro' -l /var/tmp/rings/readme_sample -b leveldb -o
    {"4":"ro"}
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1,"4":"ro"},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

To achieve this:

                        +-----------+
                        | "4": "ro" |
                        +-----------+

So now vnode `4`, to which the key `/yunong/yunong.txt` maps, contains the
value `ro`.

### Remapping Vnodes

Now that the vnode we want to move can be identified, we will want to actually
move it to the new pnode.  To see a full list of vnodes you have marked use
the command [`get-data-vnodes`](#get-data-vnodes).
<a name="sample-datav"></a>

    $ ./bin/fash.js get-data-vnodes -l /var/tmp/rings/readme_sample
    vnodeArray:  [ 4 ]

If this array were much larger, say, hundreds or thousands of vnodes, but you
were still just interested in knowing what data vnode `4` contained (or if at
any point you needed to know which pnode it was assigned to) the [`get-vnode-pnode-and-data`](#get-vnode-pnode-and-data)
command displays all available information about a given vnode.
<a name="sample-datapv"></a>

    $ ./bin/fash.js get-vnode-pnode-and-data -v 4 -l /var/tmp/rings/readme_sample
    vnodes:  { '4': { pnode: 'tcp://1.moray.emy-11.joyent.us:2020', vnodeData: 1 } }

In our example, from this you could conclude that vnode 4 was marked as
read-only for relocation, so it is intended to be remapped to a new pnode `tcp://3.moray.emy-11.joyent.us:2020`
with [`remap-vnode`](#remap-vnode):

    $ ./bin/fash.js remap-vnode -v 4 -b leveldb -p "tcp://3.moray.emy-11.joyent.us:2020" -l /var/tmp/rings/readme_sample -o
    {"tcp://3.moray.emy-11.joyent.us:2020":"4"}
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1},"tcp://3.moray.emy-11.joyent.us:2020":{"4":"ro"}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

So that the topology now looks like this:

```
                   PNODES                                       VNODES

                                                             +-----------+
+-------------------------------------------+   +--------+   |  "0": 1   |
|                                           |                +-----------+
|                                           |
|   "tcp://1.moray.emy-11.joyent.us:2020"   |                +-----------+
|                                           |   +--------+   |  "2": 1   |
|                                           |                +-----------+
|                                           |
+-------------------------------------------+


                                                             +-----------+
+-------------------------------------------+   +--------+   |  "1": 1   |
|                                           |                +-----------+
|                                           |
|   "tcp://2.moray.emy-11.joyent.us:2020"   |                +-----------+
|                                           |   +--------+   |  "3": 1   |
|                                           |                +-----------+
|                                           |
+-------------------------------------------+                +-----------+
                                                +--------+   |  "5": 1   |
                                                             +-----------+

+-------------------------------------------+
|                                           |
|                                           |
|   "tcp://3.moray.emy-11.joyent.us:2020"   |                +-----------+
|                                           |   +--------+   | "4": "ro" |
|                                           |                +-----------+
|                                           |
+-------------------------------------------+
```

This command will also take a comma-separated list of vnodes (rather than just
a single one) to move a group of vnodes to the same pnode in one batch.  It is
also possible to remap vnodes to an existing pnode, rather than a new one.

### Removing Pnodes

If you've decided that you no longer need a particular pnode, you can remove
it.  There is one caveat, which is that you have to remap the vnodes it
contains first.  A list of vnodes on any pnode is available with [`get-vnodes`](#get-vnodes).
<a name="sample-vnodes"></a>

    $ ./bin/fash.js get-vnodes 'tcp://3.moray.emy-11.joyent.us:2020' -l /var/tmp/rings/readme_sample -b leveldb
    [ 4 ]

So now we would remap vnode `4`, this time to an existing pnode:

    $ ./bin/fash.js remap-vnode -v 4 -b leveldb -p "tcp://1.moray.emy-11.joyent.us:2020" -l /var/tmp/rings/readme_sample -o
    {"tcp://1.moray.emy-11.joyent.us:2020":"4"}
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1,"4":"ro"},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

Although it appears from that output that the vnode
"tcp://3.moray.emy-11.joyent.us:2020" is already gone, that is not true.  This
is because the LevelDB keys the [print-hash](#print) command uses to
display the LevelDB keys does not use the `/PNODE` key, which keeps track of
all the pnodes in the ring.  This behavior is also present in [the -o flag available on some commands](#o-flag-commands). In any case, we can see the full list of the pnodes on the ring using [`get-pnodes`](#get-pnodes).
<a name="sample-pnodes"></a>

    $ ./bin/fash.js get-pnodes -l /var/tmp/rings/readme_sample -b leveldb
    [ 'tcp://1.moray.emy-11.joyent.us:2020',
      'tcp://2.moray.emy-11.joyent.us:2020',
      'tcp://3.moray.emy-11.joyent.us:2020' ]

So now we can remove the pnode from LevelDB:

    $ ./bin/fash.js remove-pnode -p "tcp://3.moray.emy-11.joyent.us:2020" -b leveldb -l /var/tmp/rings/readme_sample -o
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1,"4":"ro"},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

And check that it's really gone:

    $ ./bin/fash.js get-pnodes -l /var/tmp/rings/readme_sample -b leveldb
    [ 'tcp://1.moray.emy-11.joyent.us:2020',
       'tcp://2.moray.emy-11.joyent.us:2020' ]

## LevelDB Schema

The key-value store that we use is called LevelDB.  For more information on how
LevelDB works generally, see https://github.com/google/leveldb.  For
information on the LevelDOWN library that serves as a Node.js binding for
LevelDB, see https://github.com/level/leveldown/.

In fash's setup, we create the following keys to manipulate and describe the
structure (LevelDB key-value store) that underlies a ring topology (pnode-vnode
mapping).  In the `leveldb.js` file, each of these are assigned to a constant
variable prefaced with `LKEY_` with `P` and `V` standing in for `%s` and `%d`.
The fact that the key names resemble file paths does not imply that a navigable
file system backs this structure. (For example, there is no key that
corresponds to `/VNODE`).

```
    KEY               VALUE
```
<a name="lkey_vnode_count"></a>
```
VNODE_COUNT      |    The total number of vnodes in the ring
```
<a name="lkey_vnode_data"></a>
```
VNODE_DATA       |    The array of vnodes that contain optional data
```
<a name="lkey_vnode_v"></a>
```
/VNODE/%d        |    The pnode a given vnode (%d) is assigned to
```
<a name="lkey_pnodes"></a>
```
/PNODE           |    The array of all pnodes in the hash ring
```
<a name="lkey_vnodes"></a>
```
/PNODE/%s        |    The array of all vnodes that are assigned to the given pnode (%s)
```
<a name="lkey_data"></a>
```
/PNODE/%s/%d     |    The value of the data on the given vnode (%d), which is on the given pnode (%s)
```
<a name="lkey_algorithm"></a>
```
ALGORITHM        |    The hashing algorithm used to create the ring topology
```
<a name="lkey_version"></a>
```
VERSION          |    Supposedly the version of fash we are on -- at this time, hard-coded to 2.1.0
```
<a name="lkey_complete"></a>
```
COMPLETE         |    Essentially a flag that is set to 1 when the ring creation finishes
```

## LevelDB-supported CLI

The purpose of these descriptions is to identify and explain each command's
relationship to the LevelDB keys it needs to function.  Many of these are
explained in the context of their use in the [walkthrough](#walkthrough)
section.  This section will describe them in terms of their relationship to
LevelDB.

  * [`create`](#create)
  * [`deserialize-ring`](#deserialize-ring)
  * [`add-data`](#add-data)
  * [`remap-vnode`](#remap-vnode)
  * [`remove-pnode`](#remove-pnode)
  * [`get-pnodes`](#get-pnodes)
  * [`get-vnodes`](#get-vnodes)
  * [`get-vnode-pnode-and-data`](#get-vnode-pnode-and-data)
  * [`get-data-vnodes`](#get-data-vnodes)
  * [`get-node`](#get-node)
  * [`print-hash`](#print-hash)
  * [`diff`](#diff)

### create
[Click here to see an example](#command-line-ring-creation)

`create` opens the LevelDB database and populates all of the keys in the
schema.  A step-by-step listing of the order in which the keys are populated is
below:

1. First, the count of vnodes is stored in [`VNODE_COUNT`](#lkey_vnode_count).
2. In batches of 1000, each [`/VNODE/%d`](#lkey_vnode_v) key is written to its
correspondent pnode.
3. Each [`/PNODE/%s/%d`](#lkey_data) key is assigned the `LVAL_NULL` default
data value of `1`.
4. The [`/PNODE/%s`](#lkey_vnodes) keys are written from the `pnodeToVnodeMap`
created during the previous step, and the [`/PNODE`](#lkey_pnodes) array is
populated from the keys of the `pnodeMap` object created from the pnode options
passed in this command.
5. (and 6 and 7) The [`ALGORITHM`](#lkey_algorithm), [`VERSION`](#lkey_version),
and [`COMPLETE`](#lkey_complete) keys are written -- the latter two hard-coded
to default values.  The algorithm is taken from the options passed, and, if none
are found, defaults to sha256.

The effect of this is to create a hash ring with a one-to-many relationship
between pnodes and vnodes, and a one-to-one relationship between vnodes and the
data they hold.  The [LevelDB keys](#leveldb-schema) end up serving as a kind of
metadata for describing aspects of this structure.

### deserialize-ring
[Click here to see an example](#sample-deserialize)

`deserialize-ring` takes a json representation of a topology from a file or
stdin and turns it into a LevelDB data structure.  In this case, only the
LevelDB keys used for accessing the underlying mappings will be written.  Since
the hashing has already necessarily completed in order to create the input
given, [`ALGORITHM`](#lkey_algorithm), [`VERSION`](#lkey_version), and [`COMPLETE`](#lkey_complete)
do not need to be written. This command writes [`/VNODE/%d`](#lkey_vnode_v), [`/PNODE/%s/%d`](#lkey_data),
[`/PNODE/%s`](#lkey_vnodes), and [`VNODE_DATA`](#lkey_vnode_data).

### add-data
[Click here to see an example](#adding-data-to-vnodes)

`add-data` takes a vnode number and allows you to assign any data to it.  If
there is any use-case for which the consuming application needs to
differentiate between arbitrary vnodes, assigning an identifiable data string
to it is the way to do that.  The way it works is that the value of the key [`/PNODE/%s/%d`](#lkey_data)
has to be overwritten to contain the data you've passed, and the vnode chosen
to put the data inside has to be added to the array stored in [`VNODE_DATA`](#lkey_vnode_data).
To access [`/PNODE/%s/%d`](#lkey_data), you need the pnode name.  The function
does a lookup of the [`/VNODE/%d`](#lkey_vnode_v) key to find the pnode that
the vnode you passed in is assigned to, and then assigns the data to that key.
For [`VNODE_DATA`](#lkey_vnode_data), the function does a lookup of the current
value of the key (an array `vnodeArray`), and if the chosen vnode isn't in that
array, and the data is not null, it is pushed into `vnodeArray`, which is
reassigned to [`VNODE_DATA`](#lkey_vnode_data).

### remap-vnode
[Click here to see an example](#remapping-vnodes)

`remap-vnode` takes a list of vnodes (which can be just one vnode) and a
target pnode, and moves those vnodes from their current location onto the
target pnode.  This is a multi-step process that involves multiple LevelDB
keys.

1. First there is an attempt to look up the vnodes under the target pnode using
the [`/PNODE/%s`](#lkey_vnodes) key, not to retrieve the
list of vnodes, but to see if a `NotFoundError` is generated.  By
recording this if/else in a boolean variable, this gives the rest of the
function access to whether or not the pnode is a new or an existing one.
2. Next there is a lookup on the [`/VNODE/%d`](#lkey_vnode_v)
key to retrieve the current pnode of the vnode(s) set to be remapped.  If the
current and intended pnode are the same, an error is thrown and the operation
errors out.
3. Then there is a lookup of the [`/PNODE/%s/%d`](#lkey_data)
key to get the data on the vnode.
4. Next the keys to assign data to the moving vnodes -- [`/PNODE/%s/%d`](#lkey_data)
-- are deleted.  The vnodes for the old pnode are retrieved from the [`/PNODE/%s`](#lkey_vnodes) 
key, then put into a batch for later.  Then, the [`/PNODE/%s/%d`](#lkey_data)
keys are deleted.
5. The new vnodes-from-pnode mappings key ([`/PNODE/%s`](#lkey_vnodes)),
data keys ([`/PNODE/%s/%d`](#lkey_data)), and
pnode-from-vnode ([`/VNODE/%d`](#lkey_vnode_v)) are put
into batches to be committed as LevelDB keys in the final commit step.
6. The new pnode is written to the [`/PNODE`](#lkey_pnodes) key.

### remove-pnode
[Click here to see an example](#removing-pnodes)

`remove-pnode` checks that the [`/PNODE/%s`](#lkey_vnodes)
key exists, and if it has any vnodes.  If it has vnodes, it will ask the
operator to re-assign those first.  If not, it will delete the the [`/PNODE/%s`](#lkey_vnodes)
key and alter the the [`/PNODE`](#lkey_pnodes) key's array so that it no longer
contains the pnode in question.

### get-pnodes
[Click here to see an example](##sample-pnodes)

`get-pnodes` lists the array of pnodes in the hash ring.  It does this via a get
on the [`/PNODE`](#lkey_pnodes) key in LevelDB.

### get-vnodes
[Click here to see an example](#sample-vnodes)

`get-vnodes` lists the array of vnodes in a given pnode in the hash ring.  It
does this via a get on the [`/PNODE/%s`](#lkey_vnodes) key in LevelDB, where `%s`
is the argument you pass to the `-p` flag when invoking this command.

### get-vnode-pnode-and-data
[Click here to see an example](#sample-datapv)

`get-vnode-pnode-and-data` takes a (number of) vnode(s), and from that returns
the values of the pnode it is assigned to via the [`/VNODE/%d`](#lkey_vnode_v)
key and also the data it contains via the [`/PNODE/%s/%d`](#lkey_data) key.

### get-data-vnodes
[Click here to see an example](#sample-datav)

`get-data-vnodes` returns the value of the [`VNODE_DATA`](#lkey_vnode_data) key.
This will return any vnode that has a value other than the default value of 1 --
it cannot, at the time of writing, select on different data values held in
vnodes.  However, it offers a verbose flag so that an operator can view the data
held within each vnode, as well as which pnode it is assigned to.  The keys used
to accomplish this are those used in [`get-vnode-pnode-and-data`](get-vnode-pnode-and-data).

### get-node
[Click here to see an example](#sample-node)

`get-node` takes a key, runs it through fash's deterministic hashing algorithm
to determine the vnode it is on, and does a get on the [`/VNODE/%d`](#lkey_vnode_v)
key in LevelDB, which returns the pnode information needed to do a get on the [`/PNODE/%s/%d`](#lkey_data)
key in LevelDB, which gives back the data on the vnode and pnode derived from
the key.

### print-hash
[Click here to see an example](#sample-print)

`print-hash` lets you view the hash ring topology.  This operation is also
invoked by passing the -o flag to the <a name='o-flag-commands'>commands</a>
[create](#create), [add-data](#add-data), [remap-vnode](#remap-vnode), or [remove-pnode](#remove-pnode)
-- but do so *WITH EXTREME CAUTION*. For large hash rings, such as those with a
million or more vnodes, this operation will be extremely slow, or may not even
complete.

    ./bin/fash.js print-hash -l /var/tmp/rings/readme_sample -b leveldb
    {"vnodes":6,"pnodeToVnodeMap":{"tcp://1.moray.emy-11.joyent.us:2020":{"0":1,"2":1,"4":"ro"},"tcp://2.moray.emy-11.joyent.us:2020":{"1":1,"3":1,"5":1}},"algorithm":{"NAME":"sha256","MAX":"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF","VNODE_HASH_INTERVAL":"2aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"},"version":"2.1.0"}

### diff
[Click here to see an example](#sample-diff)

`diff` shows certain differences between two hash ring topologies.  The first
topology you pass will be compared to the second -- bear this in mind for the
differences returned will be in a format that uses the words "added" and
"removed."  In particular, the names of the pnodes in the ring, as well as the
number and allocations of vnodes, will be compared.  **Data within each vnode
will be ignored.**  A sample return is below:

    $ ./bin/fash.js diff -b leveldb /var/tmp/rings/sample1 /var/tmp/rings/sample2
    {"tcp://1.moray.emy-11.joyent.us:2020":{"added":[20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99]}}

In terms of schema keys, the function gets the values of [`/PNODE`](#lkey_pnodes)
from the two topologies passed to the command, then looks up the values of [`/PNODE/%s`](#lkey_vnodes)
to get the complete set of data it compares.

## Running Tests

Just run `make prepush` or `make test`, first exporting the `DB_LOCATION`
variable for the load test file to give it the location of your hash ring.

Copyright (c) 2017, Joyent, Inc.

This Source Code Form is subject to the terms of the Mozilla Public
License, v. 2.0. If a copy of the MPL was not distributed with this
file, You can obtain one at http://mozilla.org/MPL/2.0/.
