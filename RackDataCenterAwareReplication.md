# Moving to github https://github.com/chrislusf/weed-fs/wiki #

How to use Replication

# Introduction #

Weed-FS can support replication. The replication is implemented not on file level, but on volume level.

# How to use Replication #

Basically, the way it works is:

1. start weed master, and optionally specify the default replication type
```
  ./weed master -defaultReplicationType=001
```
2. start volume servers as this:
```
 ./weed volume -port=8081 -dir=/tmp/1 -max=100
 ./weed volume -port=8082 -dir=/tmp/2 -max=100
 ./weed volume -port=8083 -dir=/tmp/3 -max=100
```

Submitting, Reading, Deleting files has the same steps.

# The meaning of replication type #
Note: This subject to change.

```
000: no replication, just one copy
001: replicate once on the same rack
010: replicate once on a different rack in the same data center
100: replicate once on a different data center
200: replicate twice on two other different data center
110: replicate once on a different rack, and once on a different data center
...
```

So if the replication type is xyz
```
x: number of replica in other data centers
y: number of replica in other racks in the same data center
z: number of replica in other servers in the same rack
```

x,y,z each can be 0, 1, or 2. So there are 27 possible replication types, and can be easily extended. Each replication type will physically create x+y+z+1 copies of volume data files.

# Example Topology Configuration File #
The WeedFS master server tries to read the topology configuration file. It default to /etc/weedfs/weedfs.conf if not specified.
The topology setting to configure data center and racks file format is as this.
```
  <Configuration>
    <Topology>
      <DataCenter name="dc1">
        <Rack name="rack1">
          <Ip>192.168.1.1</Ip>
        </Rack>
      </DataCenter>
      <DataCenter name="dc2">
        <Rack name="rack1">
          <Ip>192.168.1.2</Ip>
        </Rack>
        <Rack name="rack2">
          <Ip>192.168.1.3</Ip>
          <Ip>192.168.1.4</Ip>
        </Rack>
      </DataCenter>
    </Topology>
  </Configuration>
```

# Configure Topology via Volume Server #

Instead of the weed.conf file, volume servers can start with a specific data center name.
```
 weed volume -dir=/tmp/1 -port=8080 -dataCenter=dc1
 weed volume -dir=/tmp/2 -port=8081 -dataCenter=dc2
```


### Allocate File Key on specific data center ###

When requesting a file key, an optional "dataCenter" parameter can limit the assigned volume to the specific data center. For example, this specify
```
 http://localhost:9333/dir/assign?dataCenter=dc1
```