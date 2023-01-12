# uv-ffi

 An [Foreign function interface](https://en.wikipedia.org/wiki/Foreign_function_interface) ([FFI](https://github.com/libffi/libffi)) for PHP of **[libuv](http://docs.libuv.org/en/v1.x/)** cross-platform event-driven _asynchronous_ I/O library.

This **libuv ffi** implementation is based on extension [ext-uv](https://github.com/amphp/ext-uv).

The _ext-uv_ extension is on version _1.6_ of **libuv**, 1.6 is actually _1.06_, or about _39_ releases behind current 1.44.2.

All **ext-uv 0.3.0** _tests and functions_ been implemented, except **uv_queue_work**. Functionality has only been tested under `Windows` and mainly _PHP 7.4_.

`Linux` usage behavior and corrections has begun.

The actual threading feature of `uv_queue_work` in **ext-uv 0.3.0** is on pause. Getting native PThreads working with FFI, needs a lot more investigation and more likely C development of PHP source code. Seems someone else has started something similar https://github.com/mrsuh/php-pthreads.

**PR** are welcome, see [Documentation] and [Contributing].

Future versions of `uv-ffi` beyond **ext-uv 0.3.0** will include all current `libuv` features.

## Installation

There will be two ways:
    composer require symplely/uv-ffi
and:
    composer create-project symplely/uv-ffi .cdef/libuv

This package/repo is self-contained for Windows, meaning it has **GitHub Actions** building `libuv` _binary_ `.dll`, and committing back to repo. The other platform will you the distributions included `libuv` _binary_ version.

The `require` installation will include all _binaries_ under `vendor` , whereas `create-project` will delete the ones not detected for installers hardware, and setup a different loading/installation area. This feature is still a work in progress.

`FFI` is enabled by default in `php.ini` since `PHP 7.4`, as to `OpCache`, they should not be changed unless already manually disabled.
Only the `preload` section might need setting up if better performance is desired.

```ini
[ffi]
; FFI API restriction. Possible values:
; "preload" - enabled in CLI scripts and preloaded files (default)
; "false"   - always disabled
; "true"    - always enabled
ffi.enable="preload"

; List of headers files to preload, wildcard patterns allowed.
ffi.preload=path/to/.cdef/ffi_preloader.php ; For simple integration with other FFI extensions when used as project
; Or
ffi.preload=path/to/vendor/symplely/uv-ffi/preload.php

zend_extension=opcache
```

## How to use

- The functions in file [UVFunctions.php](https://github.com/symplely/uv-ffi/blob/main/src/UVFunctions.php) is the **only** _means of accessing_ **libuv** features.

The following is a PHP `tcp echo server` converted from `C` [uv book](https://nikhilm.github.io/uvbook/networking.html#tcp) that's follows also. Most of the required `C` setup and cleanup code is done automatically.

This will print "Got a connection!" to console if visited.

```php
require 'vendor/autoload.php';

$loop = uv_default_loop();
define('DEFAULT_PORT', 7000);
define('DEFAULT_BACKLOG', 128);

$echo_write = function (UV $req, int $status) {
    if ($status) {
        printf("Write error %s\n", uv_strerror($status));
    }

    print "Got a connection!\n";
};

$echo_read = function (UVStream $client, int $nRead, string $buf = null) use ($echo_write) {
    if ($nRead > 0) {
        uv_write($client, $buf, $echo_write);
        return;
    }

    if ($nRead < 0) {
        if ($nRead != UV::EOF)
            printf("Read error %s\n", uv_err_name($nRead));

        uv_close($client);
    }
};

$on_new_connection = function (UVStream $server, int $status) use ($echo_read, $loop) {
    if ($status < 0) {
        printf("New connection error %s\n", uv_strerror($status));
        // error!
        return;
    }

    $client = uv_tcp_init($loop);
    if (uv_accept($server, $client) == 0) {
        uv_read_start($client, $echo_read);
    } else {
        uv_close($client);
    }
};

$server = uv_tcp_init($loop);

$addr = uv_ip4_addr("0.0.0.0", DEFAULT_PORT);

uv_tcp_bind($server, $addr);
$r = uv_listen($server, DEFAULT_BACKLOG, $on_new_connection);
if ($r) {
    printf("Listen error %s\n", uv_strerror($r));
    return 1;
}

uv_run($loop, UV::RUN_DEFAULT);
```

The [C source](https://github.com/libuv/libuv/blob/v1.x/docs/code/tcp-echo-server/main.c)

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <uv.h>

#define DEFAULT_PORT 7000
#define DEFAULT_BACKLOG 128

uv_loop_t *loop;
struct sockaddr_in addr;

typedef struct {
    uv_write_t req;
    uv_buf_t buf;
} write_req_t;

void free_write_req(uv_write_t *req) {
    write_req_t *wr = (write_req_t*) req;
    free(wr->buf.base);
    free(wr);
}

void alloc_buffer(uv_handle_t *handle, size_t suggested_size, uv_buf_t *buf) {
    buf->base = (char*) malloc(suggested_size);
    buf->len = suggested_size;
}

void echo_write(uv_write_t *req, int status) {
    if (status) {
        fprintf(stderr, "Write error %s\n", uv_strerror(status));
    }
    free_write_req(req);
}

void echo_read(uv_stream_t *client, ssize_t nread, const uv_buf_t *buf) {
    if (nread > 0) {
        write_req_t *req = (write_req_t*) malloc(sizeof(write_req_t));
        req->buf = uv_buf_init(buf->base, nread);
        uv_write((uv_write_t*) req, client, &req->buf, 1, echo_write);
        return;
    }
    if (nread < 0) {
        if (nread != UV_EOF)
            fprintf(stderr, "Read error %s\n", uv_err_name(nread));
        uv_close((uv_handle_t*) client, NULL);
    }

    free(buf->base);
}

void on_new_connection(uv_stream_t *server, int status) {
    if (status < 0) {
        fprintf(stderr, "New connection error %s\n", uv_strerror(status));
        // error!
        return;
    }

    uv_tcp_t *client = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, client);
    if (uv_accept(server, (uv_stream_t*) client) == 0) {
        uv_read_start((uv_stream_t*) client, alloc_buffer, echo_read);
    }
    else {
        uv_close((uv_handle_t*) client, NULL);
    }
}

int main() {
    loop = uv_default_loop();

    uv_tcp_t server;
    uv_tcp_init(loop, &server);

    uv_ip4_addr("0.0.0.0", DEFAULT_PORT, &addr);

    uv_tcp_bind(&server, (const struct sockaddr*)&addr, 0);
    int r = uv_listen((uv_stream_t*) &server, DEFAULT_BACKLOG, on_new_connection);
    if (r) {
        fprintf(stderr, "Listen error %s\n", uv_strerror(r));
        return 1;
    }
    return uv_run(loop, UV_RUN_DEFAULT);
}
```

## Error handling

Initialization functions `*_init()` or synchronous functions, which may fail will return a negative number on error.
Async functions that may fail will pass a status parameter to their callbacks. The error messages are defined as `UV::E*` constants.

You can use the `uv_strerror(int)` and `uv_err_name(int)` functions to get a `string` describing the error or the error name respectively.

I/O read callbacks (such as for files and sockets) are passed a parameter `nread`. If `nread` is less than 0, there was an error (`UV::EOF` is the end of file error, which you may want to handle differently).

In general, functions and status parameters contain the actual error code, which is 0 for success, or a negative number in case of error.

## Documentation

All `functions/methods/classes` have there original **Libuv** _documentation_, _signatures_ embedded in DOC-BLOCKS.

For deeper usage understanding, see [An Introduction to libuv](https://codeahoy.com/learn/libuv/toc/).

The following functions are present in _Windows_, but not in _Linux_.

```cpp
 void uv_library_shutdown(void);
 int uv_pipe(uv_file fds[2], int read_flags, int write_flags);
 int uv_socketpair(int type,
                            int protocol,
                            uv_os_sock_t socket_vector[2],
                            int flags0,
                            int flags1);
 int uv_try_write2(uv_stream_t* handle,
                            const uv_buf_t bufs[],
                            unsigned int nbufs,
                            uv_stream_t* send_handle);
 int uv_udp_using_recvmmsg(const uv_udp_t* handle);
 uint64_t uv_timer_get_due_in(const uv_timer_t* handle);
 unsigned int uv_available_parallelism(void);
 uint64_t uv_metrics_idle_time(uv_loop_t* loop);
 int uv_fs_get_system_error(const uv_fs_t*);
 int uv_fs_lutime(uv_loop_t* loop,
                           uv_fs_t* req,
                           const char* path,
                           double atime,
                           double mtime,
                           uv_fs_cb cb);
 int uv_ip_name(const struct sockaddr* src, char* dst, size_t size);
```

## Contributing

Contributions are encouraged and welcome; I am always happy to get feedback or pull requests on Github :) Create [Github Issues](https://github.com/symplely/uv-ffi/issues) for bugs and new features and comment on the ones you are interested in.

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
