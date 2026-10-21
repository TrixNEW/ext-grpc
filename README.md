# ext-grpc

The gRPC PHP extension from [PECL grpc 1.74.0](https://pecl.php.net/package/grpc), with one completion queue per thread so it's safe on threaded PHP (pmmpthread, PocketMine-MP).

## Why

Stock ext-grpc waits for every call on one completion queue shared by the whole process, without a timeout.

- Every new PHP thread replaces that queue in its `RINIT`. A call created before the swap and waited on after it never returns, so starting any thread can hang every thread that's mid call.
- At most 6 threads may wait on the queue at once. A 7th returns early while gRPC still writes into its stack.

Here `completion_queue` is thread local, so each thread creates and waits on its own queue and neither problem exists. Non-ZTS builds are unchanged. The extension also defines `Grpc\PER_THREAD_COMPLETION_QUEUE` so callers like [pRPC](https://github.com/TrixNEW/pRPC) can tell.

The diff against PECL is the commit after `chore: import grpc 1.74.0 from pecl`.

## Tested

Linux, PHP 8.3.25 ZTS with pmmpthread 6.2.0, against a local Go gRPC server answering at once. Threads started while calls were in flight, 15 runs each:

| Case | Stock 1.74.0 | This |
| --- | ---: | ---: |
| send and receive as two batches, 4 threads | 15 hung | 0 |
| one batch, 6 threads | 7 hung | 0 |
| two batches, 6 threads | 14 hung | 0 |
| one batch, 8 threads | 15 crashed | 0 |
| two batches, 8 threads | 15 crashed | 0 |

Only Linux was tested. Windows uses the same `TSRM_TLS` macro but wasn't built.

## Build

```bash
phpize
./configure --with-php-config=/path/to/php-config
make -j"$(nproc)"
make install
```

## License

Apache-2.0, see [LICENSE](LICENSE).
