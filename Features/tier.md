#Tiering

* Feature page:
http://www.gluster.org/community/documentation/index.php/Features/data-classification

* Design: goo.gl/bkU5qv

##Theory of operation

The tiering feature enables different storage types to be used by the same
logical volume. In Gluster 3.7, the two types are classified as "cold" and
"hot", and are represented as two groups of bricks. The hot group acts as
a cache for the cold group. The bricks within the two groups themselves are
arranged according to standard Gluster volume conventions, e.g. replicated,
distributed replicated, or dispersed.

A normal gluster volume can become a tiered volume by "attaching" bricks
to it. The attached bricks become the "hot" group. The bricks within the
original gluster volume are the "cold" bricks.

For example, the original volume may be dispersed on HDD, and the "hot"
tier could be distributed-replicated SSDs.

Once this new "tiered" volume is built, I/Os to it are subjected to cacheing
heuristics:

* All I/Os are forwarded to the hot tier.

* If a lookup fails to the hot tier, the I/O will be forwarded to the cold
tier. This is a "cache miss".

* Files on the hot tier that are not touched within some time are demoted
(moved) to the cold tier (see performance parameters, below).

* Files on the cold tier that are touched one or more times are promoted
(moved) to the hot tier. (see performance parameters, below).

This resembles implementations by Ceph and the Linux data management (DM)
component.

Performance enhancements being considered include:

* Biasing migration of large files over small.

* Only demoting when the hot tier is close to full.

* Write-back cache for database updates.

##Code organization

The design endevors to be upward compatible with future migration policies,
such as scheduled file migration, data classification, etc. For example,
the caching logic is self-contained and separate from the file migration. A
different set of migration policies could use the same underlying migration
engine. The I/O tracking and meta data store compontents are intended to be
reusable for things besides caching semantics.

**Meta data:**

A database stores meta-data on the files. Entries within it are added or
removed by the changetimerecorder translator. The database is queried by
the migration daemon. The results of the queries drive which files are to
be migrated.

The database resides withi the libgfdb subdirectory. There is one database
for each brick. The database is currently sqlite. However, the libgfdb
library API is not tied to sqlite, and a different database could be used.

For more information on libgfdb see the doc file: libgfdb.txt.

**I/O tracking:**

The changetimerecorder server-side translator generates metadata about I/Os
as they happen. Metadata is then entered into the database after the I/O
completes. Internal I/Os are not included.

**Migration daemon:**

When a tiered volume is created, a migration daemon starts. There is one daemon
for every tiered volume per node. The daemon sleeps and then periodically
queries the database for files to promote or demote. The query callbacks
assembles files in a list, which is then enumerated. The frequencies by
which promotes and demotes happen is subject to user configuration.

Selected files are migrated between the tiers using existing DHT migration
logic. The tier translator will leverage DHT rebalance performance
enhancements.

**tier translator:**

The tier translator is the root node in tiered volumes. The first subvolume
is the cold tier, and the second the hot tier. DHT logic for fowarding I/Os
is largely unchanged. Exceptions are handled according to the dht_methods_t
structure, which forks control according to DHT or tier type.

The major exception is DHT's layout is not utilized for choosing hashed
subvolumes. Rather, the hot tier is always the hashed subvolume.

Changes to DHT were made to allow "stacking", i.e. DHT over DHT:

* readdir operations remember the index of the "leaf node" in the volume graph
(client id), rather than a unique index for each DHT instance.

* Each DHT instance uses a unique extended attribute for tracking migration.

* In certain cases, it is legal for tiered volumes to have unpopulated inodes
(wheras this would be an error in DHT's case).

Currently tiered volume expansion (adding and removing bricks) is unsupported.

**glusterd:**

The tiered volume tree is a composition of two other volumes. The glusterd
daemon builds it. Existing logic for adding and removing bricks is heavily
leveraged to attach and detach tiers, and perform statistics collection.