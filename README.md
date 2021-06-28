Ceph Disk Benchmarks
====================

In a reasonable [Ceph](https://ceph.com/) setup, transactions on block devices for a Ceph OSD are likely the one bottleneck you'll have.
When you place the OSD journal (`block.wal`) or the database (`block.db`) on a SSD, its performance and durability is particularly important.

"Normal" device benchmarks won't typically help you, as Ceph accesses the block devices *differently* than usual filesystems: it synchronizes every write and waits until the drive *confirms the write operation*.
In particular this means that any write cache is always flushed directly. Other benchmarks usually do not consider this special drive access mode.


## List of Devices

Sorted by IOPS - since they're relevant for Ceph. max (but max IOPS are too

* IOPS: write-sync operations per second for *one job*
* max IOPS: sum of parallel write-sync operations for *multiple jobs*
* cache: write cache activation status (`hdparm -W`)

The more IOPS in sync mode can be done, the more transactions Ceph can commit on a [Bluestore](https://docs.ceph.com/en/latest/rados/configuration/storage-devices/#bluestore) OSD in the `bstore_kv_sync` thread, which is usually the bottleneck.
Since reads/writes to the pool data happens in other threads, 8 jobs (`osd_op_num_shards_ssd`) should get IOPS than 1 job!

In this list CPU/RAM/... are ignored since we assume the device is much slower than the server.


### SSDs

| ID                             |    Size | Proto      | 1job IOPS | max IOPS | peak #Jobs | cache | Notes   |
|--------------------------------|--------:|------------|----------:|---------:|-----------:|-------|---------|
| Intel SSD 750 PCIe             |   400GB | NVMe       |     64235 |   192440 |          8 |     - | asymptotic |
| Samsung MZQLW960HMJP-00003     |   960GB | NVMe       |     34090 |   268030 |         16 |     - | linear up to ~8 jobs, then asymptotic |
| Samsung PM1643a                |   960GB | SAS        |     18545 |    93229 |         16 |     - | asymptotic |
| Samsung PM863a                 |   240GB | SATA       |     17983 |    58876 |         10 |   off | asymptotic |
| Samsung PM883                  |  7.68TB | SATA3.2 6G |     12680 |    59338 |         16 |   off | asymptotic; cache on: 5094 @ 1job, 27521 @16 |
| Pliant LB206S MS04             |   200GB | SAS        |      5028 |     5028 |          1 |     - | more jobs slow down. 2: 2651, 6: 1088, 8: 745, 10: 784 |
| Samsung 983DCT                 |   960GB | NVMe       |      4000 |    22570 |          8 |     - | asymptotic |
| WD Blue WDS100T2B0A-00SM50     |     1TB | SATA       |      1751 |     2225 |          2 |   off | 2 jobs already saturate |
| Intel SSD S4510                |   480GB | SATA       |      1600 |    48409 |         15 |   off | linear until capped |
| Intel SSD 545                  |   512GB | SATA       |      1500 |     6460 |          8 |     - | asymptotic |
| Samsung PM961                  |   128GB | NVMe       |      1480 |     1480 |          1 |     - | more jobs slow down. 2: 818, 3: 1092, 4: 525, 5: 569 |
| Transcend SSD 220s             |     1TB | NVMe       |      1420 |     5760 |          8 |     - | asymptotic |
| LENSE20512GMSP34MEAT2TA        |   512GB | NVMe       |      1150 |     3164 |          4 |     - | asymptotic |
| Samsung SSD 860 PRO            |   512GB | SATA       |      1033 |     5915 |         15 |     - | asymptotic |
| Sandisk Extreme Pro            |   960GB | SATA       |       860 |     3400 |          8 |     - | linear until capped |
| Sandisk Ultra II               |   960GB | SATA       |       600 |     3640 |          8 |     - | linear until capped |
| Samsung SSD 860 EVO            |     1TB | SATA       |       490 |     1728 |         14 |     - | linear until capped |
| Samsung SSD 970 PRO            |   512GB | NVMe       |       456 |      840 |          2 |     - | 2 jobs already saturate |
| Samsung MZVLB512HAJQ-000L7     |   512GB | NVMe       |       384 |     1164 |         10 |     - | linear until capped |

Entries are sorted by 1-job IOPS.


## Create a Benchmark

Please add your benchmark (or validate) existing entries, and [submit your changes as pull request](#contributing)!


#### Device model and link

Get device model number (`Device Model:`), connection link version and speed (`SATA Version is:` etc):

```
smartctl -a /dev/device
```

#### Write cache

As the Ceph journal is written with `fdatasync`, each IO operation waits until the drive confirms that the data was written down permanently.
Hence the write cache is written to, and then flushed. If we bypass it, we make the commits *faster* when we **turn off** the write cache:

```
# see current status
hdparm -W /dev/device

# disable the write cache
hdparm -W 0 /dev/device
```

#### [`fio`](https://fio.readthedocs.io/en/latest/index.html) Benchmark

Beware, the device will be written to and existing OSD data **will be corrupted**!

Each OSD with [Bluestore](https://docs.ceph.com/en/latest/rados/configuration/storage-devices/#bluestore) has one `bstore_kv_sync` thread, which invokes `fdatasync` after each transaction. This is what we try to benchmark.

```
fio --filename /dev/device --direct=1 --fdatasync=1 --iodepth=1 --runtime=20 --time_based --rw=write --bs=4k --group_reporting --name=ceph-iops --numjobs=1
```

In the output, after the `ceph-iops` summary was printed, look for `write: IOPS=XXXXX`.

* Increase `numjobs` (e.g. by doubling the value) to find out the performance behavior for parallel transactions and figure out the upper IOPS limit.


## Contributing

This list is intended to be expanded by YOU! Just run the test and submit a [pull request](https://help.github.com/articles/creating-a-pull-request/)!

Corrections and verifications of listed benchmarks would be very helpful, too!


## Contact

If you want to reach out, join `#sfttech:matrix.org` on [Matrix](https://matrix.org).


## License

This information is released under [CC0](http://creativecommons.org/publicdomain/zero/1.0/).
