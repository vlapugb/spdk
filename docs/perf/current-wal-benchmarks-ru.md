# Benchmark текущей WAL-реализации

## Подготовка окружения

Пересоберите WAL и benchmark-бинарники перед запуском. Это важно: старые
`bdevperf` или `spdk_bdev` могут быть перелинкованы со старым WAL.

```bash
make -C module/bdev/wal -j"$(nproc)"
rm -f build/examples/bdevperf build/fio/spdk_bdev
make -C examples/bdev/bdevperf -j"$(nproc)"
make -C app/fio/bdev -j"$(nproc)"
make -C test/unit/lib/bdev/wal -j"$(nproc)"
./test/unit/lib/bdev/wal/wal_ut
```

## JSON config для WAL

Сохраните config как `wal-malloc.json`:

```json
{
  "subsystems": [
    {
      "subsystem": "bdev",
      "config": [
        {
          "method": "bdev_malloc_create",
          "params": {
            "name": "MallocMain",
            "num_blocks": 65536,
            "block_size": 4096
          }
        },
        {
          "method": "bdev_malloc_create",
          "params": {
            "name": "MallocJournal",
            "num_blocks": 65536,
            "block_size": 4096
          }
        },
        {
          "method": "wal_bdev_create",
          "params": {
            "name": "wal0",
            "bdev_name": "MallocMain",
            "journal_name": "MallocJournal"
          }
        }
      ]
    }
  ]
}
```

## Запуск bdevperf

Локальный no-huge запуск требует явного размера памяти. Если в `/var/tmp`
остались root-owned `spdk_cpu_lock_*`, добавьте `--disable-cpumask-locks`.

WAL qd1:

```bash
build/examples/bdevperf \
  --disable-cpumask-locks \
  --no-huge -s 1024 \
  -u -m 0x1 \
  -c wal-malloc.json \
  -q 1 -o 4096 -w write -t 5 \
  -T wal0
```

## Запуск fio

fio job должен использовать `thread=1`: SPDK fio plugin работает в thread mode,
а не через fork-per-job.

Пример job для WAL qd1:

```ini
[global]
ioengine=/path/to/spdk/build/fio/spdk_bdev
spdk_json_conf=/path/to/wal-malloc.json
env_context=--no-pci --no-huge --legacy-mem --iova-mode=va
spdk_mem=1024
thread=1
direct=1
group_reporting=1
time_based=1
runtime=5
ramp_time=0
rw=write
bs=4k
iodepth=1
numjobs=1

[wal_qd1]
filename=wal0
```

Запуск:

```bash
fio fio-wal-qd1.fio --output=wal-q1.json --output-format=json
```
