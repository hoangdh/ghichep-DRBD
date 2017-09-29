## How to fix DRBD recovery from split brain

DRBD® refers to block devices designed as a building block to form high availability (HA) clusters. This is done by mirroring a whole block device via an assigned network. DRBD can be understood as network based raid-1.

Problem
DRBD does not connect between server

```
#/proc/drbd 
version: 8.4.0 (api:1/proto:86-100)
GIT-hash: 28753f559ab51b549d16bcf487fe625d5919c49c build by gardner@, 2011-12-1
2 23:52:00
 0: cs:StandAlone ro:Secondary/Unknown ds:UpToDate/DUnknown   r-----
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:76
 
#log message on the node1
Mar  7 15:38:05 node1 kernel: block drbd0: Split-Brain detected but unresolved, dropping connection!
 
#/proc/drbd on the node2
version: 8.4.0 (api:1/proto:86-100)
GIT-hash: 28753f559ab51b549d16bcf487fe625d5919c49c build by gardner@, 2011-12-1
2 23:52:00
 0: cs:StandAlone ro:Primary/Unknown ds:UpToDate/DUnknown   r-----
    ns:0 nr:0 dw:144 dr:4205 al:5 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:100
```

### Solution:

- Step 1: Start drbd manually on both nodes

- Step 2: Define one node as secondary and discard data on this

```
drbdadm secondary all
drbdadm disconnect all
drbdadm -- --discard-my-data connect all
```

- Step 3: Define anoher node as primary and connect

```
drbdadm primary all
drbdadm disconnect all
drbdadm connect all
```

Link: http://www.ipserverone.info/dedicated-server/linux-2/how-to-fix-drbd-recovery-from-split-brain/