---
title: "Set up a bcache partition and evaluate its performance"
description: "Set up a bcache partition            Identifying NVME block size      $ sudo nvme..."
pubDatetime: 2026-01-19T15:30:21.386Z
---

## Set up a bcache partition

### Identifying NVME block size

```bash
$ sudo nvme list
...
lbaf  0 : ms:0   lbads:9  rp:0x2 (in use)
lbaf  1 : ms:0   lbads:12 rp:0
```

From the line marked `in use`, `lbads:9` means a block size of 512B
(`echo $((1<<9))`).

```bash
sudo gdisk /dev/nvme0n1
n # add a new partition
w # write table to disk and exit
```



### Identifying block size

```bash
$ sudo smartctl -a /dev/sda
...
Logical block size:   4096 bytes
...
```



### Configuring bcache

Format for bcache:

```bash
sudo wipefs -a /dev/nvme0n1p5
sudo make-bcache -C /dev/nvme0n1p5

sudo wipefs -a /dev/sda
sudo make-bcache --block 4k -B /dev/sda

sudo wipefs -a /dev/sdb
sudo make-bcache --block 4k -B /dev/sdb
```

Get the cache device UUID:

```bash
$ sudo bcache-super-show /dev/nvme0n1p5 | grep cset
cset.uuid               7caf86be-7954-4bda-ae04-18dc5197151c
```

Attach cache device to backing devices:

```bash
sudo sh -c "echo 7caf86be-7954-4bda-ae04-18dc5197151c > /sys/block/bcache0/bcache/attach"
sudo sh -c "echo 7caf86be-7954-4bda-ae04-18dc5197151c > /sys/block/bcache1/bcache/attach"
```

Enable discard:

```bash
sudo sh -c "echo 1 > /sys/block/nvme0n1/nvme0n1p5/bcache/discard"
```

Enable writeback:

```bash
sudo sh -c "echo writeback > /sys/block/bcache0/bcache/cache_mode"
sudo sh -c "echo writeback > /sys/block/bcache1/bcache/cache_mode"
```

Detach example:

```bash
echo 7caf86be-7954-4bda-ae04-18dc5197151c | sudo tee /sys/block/bcache0/bcache/detach
echo 7caf86be-7954-4bda-ae04-18dc5197151c | sudo tee /sys/block/bcache1/bcache/detach
```



## Benchmark

### Creating data

Since memory is 64GB, create larger test data of 256GB:

```bash
sudo dd if=/dev/urandom of=data bs=1M count=$((1024*64)) status=progress
```

### Temporarily disable & clear cache

```bash
sudo sh -c "echo 7caf86be-7954-4bda-ae04-18dc5197151c > /sys/block/bcache0/bcache/detach"
sudo sh -c "echo 7caf86be-7954-4bda-ae04-18dc5197151c > /sys/block/bcache1/bcache/detach"

sudo sh -c "echo 1 > /sys/block/sda/bcache/detach"
sudo sh -c "echo 1 > /sys/block/sdb/bcache/detach"

sudo sh -c "echo 1 > /sys/fs/bcache/7caf86be-7954-4bda-ae04-18dc5197151c/stop"
```

### Re-enable

```bash
sudo sh -c "echo 7caf86be-7954-4bda-ae04-18dc5197151c > /sys/block/bcache0/bcache/attach"
sudo sh -c "echo 7caf86be-7954-4bda-ae04-18dc5197151c > /sys/block/bcache1/bcache/attach"
```

### Benchmark test

Drop memory caches:

```bash
sudo sysctl vm.drop_caches=3
```

Random read test script:

```python
#!/usr/bin/env python

import sys
import os
import tqdm
import contextlib
import random
import time

target_file = sys.argv[1]
num_blocks = 1024*256
block_size = 1024*1024
count = 0

random.seed(12345)

with contextlib.ExitStack() as exit_stack:
    tqdmobj = tqdm.tqdm(total=num_blocks, maxinterval=1)
    exit_stack.callback(tqdmobj.close)

    filesize = os.path.getsize(target_file)
    fp = open(target_file, "rb", buffering=0)
    exit_stack.callback(fp.close)

    for block_num in range(num_blocks):
        sys.stdout.write(f"{time.time()},{block_num}\n")
        pos = random.randint(0, filesize-1)
        fp.seek(pos, os.SEEK_SET)
        fp.read(block_size)
        tqdmobj.update()
```

Run it:

```bash
python3 -u main.py > data.csv
```

Check total amount written to cache:

```
/sys/fs/bcache/7caf86be-7954-4bda-ae04-18dc5197151c/cache0/written
```



## Reference

[https://wiki.archlinux.jp/index.php/Bcache](https://wiki.archlinux.jp/index.php/Bcache)
