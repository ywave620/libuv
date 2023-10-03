# Transition to io_uring based networking

## windfall



## src notes

internal stuffs are named as uv__*

The event loop `src/unix/linux.c uv__io_poll()`
it either calls uv__io_cb or uv__poll_io_uring

uv__io_cb for uv_stream_t is uv__stream_io() with the execption that for a server uv_tcp_t, it is uv__server_io()


## build & test

see README.md


## How to debug a single test

export UV_RUN_AS_ROOT=1 && test/run-tests callback_stack

build/libuv.a expose symbols prefixed with uv

mkdir Debug
cd Debug
cmake -DCMAKE_BUILD_TYPE=Debug ..
cmake --build Debug

git apply debug-test-tcp-write-in-a-row.diff && gcc -g -c -I include/ test/test-tcp-write-in-a-row.c -o test/test-tcp-write-in-a-row.o && gcc -g test/test-tcp-write-in-a-row.o -o test/test-tcp-write-in-a-row -LDebug -l:libuv.a

  -LDebug <!--lib serach path--> -luv <!--lib name-->

test/test-tcp-write-in-a-row is an executable with symbols



## use io_uring for socket reading

uv__iou_stream_read() is a brand new function, just like other uv__iou_XXX(), it is used to prepare and submit SQE to io_uring

uv__poll_io_uring() is changed to retrieve SQE created by stream read operation, in addition to fs operations
uv__read() is changed to handle SQE created by stream read operation
uv__io_poll() stays unchanged, it calls uv__poll_io_uring() whenever the epoll_pwait() returned with there is some CQE to reap

uv_read_start
1. call alloc_cb to get a buffer and submit another io_uring operation(read() or recvmsg())

uv_read_stop
1. mark the stream as not wanting read

uv__read
1. call read_cb
2. if user wants more read, then call alloc_cb to get a buffer and submit another io_uring operation(read() or recvmsg())

race: if user calls uv_read_stop in middle of a in flight io_uring read, then the read_cb will be fired anyway

