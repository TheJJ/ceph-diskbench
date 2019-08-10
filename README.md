Ceph Disk Benchmarks
====================

In a reasonable [Ceph](https://ceph.com/) setup, block devices for a Ceph OSD are likely the one bottleneck you'll have.
When you place the OSD journal (`block.wal`) or the database (`block.db`) on a SSD, its performance and durability is particularly important.

"Normal" device benchmarks won't typically help you, as Ceph accesses the block devices *differently* than usual filesystems: it synchronizes every write and waits until the drive *confirms the write operation*.
In particular this means that any write cache is always flushed directly. Other benchmarks usually do not consider this special drive access mode.

If you need professional help for your Ceph cluster, try https://croit.io/ !


## List of Devices

| ID                             | Size  | Type  | Proto |    IOPS |  #Jobs | Notes |
|--------------------------------|-------|-------|-------|---------|--------|-------|
| Samsung MZQLW960HMJP-00003     | 960GB | SSD   | NVMe  |  268030 |     16 | on IBM Power9, 1 job: 34090, then linear up to ~8 jobs |
| Samsung PM863a                 | 240GB | SSD   | SATA  |   58876 |     10 | 1 job: 17983, then linear  |
| Pliant LB206S MS04             | 200GB | SSD   | SAS   |    5028 |      1 | 2 jobs: 2651, 2: 1277, 6: 1088, 7: 691, 8: 745, 9: 617, 10: 784 |
| LENSE20512GMSP34MEAT2TA        | 512GB | SSD   | NVMe  |    3164 |      4 | 1 job: 1150, 2: 1588, 3: 2396, 5: 3008 |
| Samsung PM961                  | 128GB | SSD   | NVMe  |    1480 |      1 | 2 jobs: 818, 3: 1092, 4: 525, 5: 569 |
| Samsung MZVLB512HAJQ-000L7     | 512GB | SSD   | NVMe  |    1164 |     10 | 1 job: 384, 2: 771, 3: 603, 4: 715, 5: 786, 10: 1164 |
| Samsung SSD 970 PRO            | 512GB | SSD   | NVMe  |     840 |      2 | 1 job: 456, 3: 817, 4: 782, 5: 785  |


For each device, the optimal number of jobs and resulting IO operations per second are determined.

The more IOPS in sync mode can be done, the more transactions Ceph can process on the OSD.


## Create a Benchmark

#### Device model

Get device model number:

```
smartctl -a /dev/device
```

#### Write cache

As the Ceph journal is written with `O_SYNC` and `O_DIRECT`, each IO operation waits until the drive confirms that the data was written down permanently.
Thus often the disk is *faster* when you **turn off** its write cache:

```
# see current status
hdparm -W /dev/device

# disable the write cache
hdparm -W 0 /dev/device
```

#### [`fio`](https://fio.readthedocs.io/en/latest/index.html) Benchmark

Beware, the device will be written to!

You can create a new (LVM) partition when device is already in use and do the benchmark on it.

To find the **jobcount** with highest IOPS score, try `numjob` values from 1 to 16:

```
for jobcount in `seq 16`; do
    fio --filename=/dev/device --direct=1 --sync=1 --iodepth=1 --runtime=60 --time_based --rw=write --bs=4k --numjobs=$jobcount --group_reporting --name=ceph-journal-write-test
done
```

In the output, after the `ceph-journal-write-test` summary was printed, look for `write: IOPS=XXXXX`.


## Contributions

This list is intended to be expanded by YOU! Just run the test and submit a [pull request](https://help.github.com/articles/creating-a-pull-request/)!

Corrections and verifications of listed benchmarks would be very helpful, too!


## License

This information is released under [CC0](http://creativecommons.org/publicdomain/zero/1.0/).
