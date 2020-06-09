---
layout: post
title:  "Bug Hunting in Python"
date:   2020-06-09 09:23:46 -0400
categories: blog debugging fedora
---

Having given the blog a good four years to really settle in and get
comfortable, I thought it was about time I wrote a post. I don't want to strain
myself, so it'll be short and incomplete (don't worry, I'll pad it with lots of
debugging output so you can give that page key a workout).

There are a myriad ways to debug Python applications. My first stop is
typically [pdb](https://docs.python.org/3/library/pdb.html). It's simple and
ships with Python.

When the Python process in question makes pdb difficult to use like when it's
being run by systemd [rpdb](https://pypi.org/project/rpdb/) makes it very easy
to remotely debug the program. rpdb even supports SIGTRAP which is very helpful
in situations where the program is hanging and you want to quickly find out
where: just install the signal handler with `import rpdb; rpdb.handle_trap()`,
run the process until it seems to get stuck, send the process SIGTRAP (consult
`man 7 signal` for the signal number on your platform, it's likely 5) and
connect to the debugger with `nc 127.0.0.1 4444`.

Occasionally, though, (r)pdb isn't enough. This could be because the bug
involves a C extension or Python itself. For example, during Fedora's rebase to
Python 3.9b1, a [package failed to
rebuild](https://bugzilla.redhat.com/show_bug.cgi?id=1841785). Adam Williamson
diagnosed the build failure and isolated the package test that resulted in the
process hanging forever. Unfortunately, rpdb was insufficient in this case.
Upon sending the process a TRAP signal, we're not rewarded with a remote pdb
session:

```
dev in packaging/python-stompest/stompest/src/twisted on  master [?] via 🐍 v3.9.0b1 (stompest)
❯ python -m pytest -vvvvv -s --ignore stompest/twisted/tests/async_client_integration_test.py

platform linux -- Python 3.9.0b1, pytest-5.4.3, py-1.8.1, pluggy-0.13.1 -- /home/jcline/.virtualenvs/stompest/bin/python3
cachedir: .pytest_cache
rootdir: /home/jcline/packaging/python-stompest/stompest/src/twisted
collected 14 items

stompest/twisted/tests/async_client_test.py::AsyncClientConnectTimeoutTestCase::test_connected_timeout <- ../../../stompest-715f358b8503320ea42bd4de6682916339943edc/src/twisted/stompest/twisted/tests/async_client_test.py INFO:twisted:--> stompest.twisted.tests.async_client_test.AsyncClientConnectTimeoutTestCase.test_connected_timeout <--
INFO:twisted:Factory starting on 44893
INFO:twisted:Starting factory <twisted.internet.protocol.Factory object at 0x7f037f652130>
INFO:stompest.twisted.protocol:Connecting to localhost:44893 ...
INFO:twisted:Starting factory <stompest.twisted.protocol.StompFactory object at 0x7f037f669be0>
DEBUG:stompest.twisted.tests.broker_simulator:Connection made
DEBUG:stompest.twisted.protocol:Sending CONNECT frame [version=1.0]
DEBUG:stompest.twisted.tests.broker_simulator:Received CONNECT frame [version=1.0]
ERROR:stompest.twisted.listener:Disconnect because of failure: STOMP broker did not answer on time [timeout=1.0]
INFO:stompest.twisted.listener:Disconnecting ... [reason=STOMP broker did not answer on time [timeout=1.0]]
INFO:stompest.twisted.listener:Disconnected: Connection was closed cleanly.
DEBUG:stompest.twisted.listener:Calling disconnected errback: STOMP broker did not answer on time [timeout=1.0]



Trace/breakpoint trap (core dumped)
```

Not great. Never fear, though, for gdb is here to save us. The first thing we
can do is take a look at that core dump:

```
# In this fantasy there's only one core dump
dev in packaging/python-stompest/stompest/src/twisted on  master [?] via 🐍 v3.9.0b1 (stompest) 
❯ coredumpctl list
TIME                            PID   UID   GID SIG COREFILE  EXE
Tue 2020-06-09 09:55:44 EDT   37144  1000  1000   5 present   /usr/bin/python3.9

dev in packaging/python-stompest/stompest/src/twisted on  master [?] via 🐍 v3.9.0b1 (stompest) 
❯ coredumpctl debug python3
           PID: 37144 (python3)
           UID: 1000 (jcline)
           GID: 1000 (jcline)
        Signal: 5 (TRAP)
     Timestamp: Tue 2020-06-09 09:55:43 EDT (1h 0min ago)
  Command Line: python3 -m pytest -vvvvv -s --ignore stompest/twisted/tests/async_client_integration_test.py
    Executable: /usr/bin/python3.9
       Storage: /var/lib/systemd/coredump/core.python3.1000.a5660049602e418c9ed5d7737a890e5c.37144.1591710943000000000000.lz4
       Message: Process 37144 (python3) of user 1000 dumped core.
                
                Stack trace of thread 37144:
                #0  0x00007f038e37157e PyException_GetContext (libpython3.9.so.1.0 + 0x18d57e)
                #1  0x00007f038e2ffa38 _PyErr_SetObject (libpython3.9.so.1.0 + 0x11ba38)
                #2  0x00007f038e30895d gen_send_ex (libpython3.9.so.1.0 + 0x12495d)
                #3  0x00007f038e3c4a81 _gen_throw (libpython3.9.so.1.0 + 0x1e0a81)
                #4  0x00007f038e3c497a gen_throw (libpython3.9.so.1.0 + 0x1e097a)
                #5  0x00007f038e30a180 method_vectorcall_VARARGS (libpython3.9.so.1.0 + 0x126180)
                #6  0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #7  0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #8  0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #9  0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #10 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #11 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #12 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #13 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #14 0x00007f038e2eccc3 _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108cc3)
                #15 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #16 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #17 0x00007f038e2ef9fa _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x10b9fa)
                #18 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #19 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #20 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #21 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #22 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #23 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #24 0x00007f038e2eb9a1 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x1079a1)
                #25 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #26 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #27 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #28 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #29 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #30 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #31 0x00007f038e2eccc3 _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108cc3)
                #32 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #33 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #34 0x00007f038e2ef9fa _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x10b9fa)
                #35 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #36 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #37 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #38 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #39 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #40 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #41 0x00007f038e2eb9a1 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x1079a1)
                #42 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #43 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #44 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #45 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #46 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #47 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #48 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #49 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #50 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #51 0x00007f038e2eccc3 _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108cc3)
                #52 0x00007f038e3088da gen_send_ex (libpython3.9.so.1.0 + 0x1248da)
                #53 0x00007f038e30157e method_vectorcall_O (libpython3.9.so.1.0 + 0x11d57e)
                #54 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #55 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #56 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #57 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #58 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #59 0x00007f038e2eccc3 _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108cc3)
                #60 0x00007f038e2ebf59 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107f59)
                #61 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #62 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #63 0x00007f038e2eccc3 _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108cc3)
                
                Stack trace of thread 37145:
                #0  0x00007f038e1d4a24 futex_abstimed_wait_cancelable (libpthread.so.0 + 0x12a24)
                #1  0x00007f038e1d4b28 __new_sem_wait_slow (libpthread.so.0 + 0x12b28)
                #2  0x00007f038e35da5e PyThread_acquire_lock_timed (libpython3.9.so.1.0 + 0x179a5e)
                #3  0x00007f038e36c1c1 acquire_timed (libpython3.9.so.1.0 + 0x1881c1)
                #4  0x00007f038e36bfcb lock_PyThread_acquire_lock (libpython3.9.so.1.0 + 0x187fcb)
                #5  0x00007f038e2fa892 method_vectorcall_VARARGS_KEYWORDS (libpython3.9.so.1.0 + 0x116892)
                #6  0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #7  0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #8  0x00007f038e2eb9a1 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x1079a1)
                #9  0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #10 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #11 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #12 0x00007f038e2eb9a1 _PyEval_EvalCode (libpython3.9.so.1.0 + 0x1079a1)
                #13 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #14 0x00007f038e302380 method_vectorcall (libpython3.9.so.1.0 + 0x11e380)
                #15 0x00007f038e356d0c _PyObject_CallNoArg.lto_priv.17 (libpython3.9.so.1.0 + 0x172d0c)
                #16 0x00007f038e3cf9e9 calliter_iternext (libpython3.9.so.1.0 + 0x1eb9e9)
                #17 0x00007f038e2ed165 _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x109165)
                #18 0x00007f038e2ebd6f _PyEval_EvalCode (libpython3.9.so.1.0 + 0x107d6f)
                #19 0x00007f038e2f97b6 _PyFunction_Vectorcall (libpython3.9.so.1.0 + 0x1157b6)
                #20 0x00007f038e2ef9fa _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x10b9fa)
                #21 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #22 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #23 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #24 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #25 0x00007f038e2f44aa call_function (libpython3.9.so.1.0 + 0x1104aa)
                #26 0x00007f038e2ece9e _PyEval_EvalFrameDefault (libpython3.9.so.1.0 + 0x108e9e)
                #27 0x00007f038e2f9a6b function_code_fastcall (libpython3.9.so.1.0 + 0x115a6b)
                #28 0x00007f038e302380 method_vectorcall (libpython3.9.so.1.0 + 0x11e380)
                #29 0x00007f038e3b1346 t_bootstrap (libpython3.9.so.1.0 + 0x1cd346)
                #30 0x00007f038e3b12f4 pythread_wrapper (libpython3.9.so.1.0 + 0x1cd2f4)
                #31 0x00007f038e1cb432 start_thread (libpthread.so.0 + 0x9432)
                #32 0x00007f038e63a9d3 __clone (libc.so.6 + 0x1019d3)

```

Hmm. If we wanted to work hard, we could perhaps do some more investigation on
the core dump, but let's play with another one of gdb's features.  gdb is a
general-purpose debugger that can, among other things, attach to running
processes. Running `gdb --pid=<pytest pid>` will attach us to the running
process where we can poke around. gdb's default interface is rather...
minimal, but can be customized with projects like
[gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard) and that's what I'm
using in the output below.

If you're running on Fedora, you'll likely be presented with a warning about
missing debuginfo when you attach to the process and should install it using
the provided command before continuing. As you can see here, I've already
installed the debuginfo and debugsource packages for Python 3.9.

```
─── Output/messages ────────────────────────────────────────────────────────────
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
PyException_GetContext (self=0x7fd61fe53580) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/exceptions.c:350
350	    PyObject *context = _PyBaseExceptionObject_cast(self)->context;
─── Assembly ───────────────────────────────────────────────────────────────────
~
~
~
~
~
 0x00007fd62eb87570  PyException_GetContext+0  mov    0x28(%rdi),%rax
 0x00007fd62eb87574  PyException_GetContext+4  test   %rax,%rax
 0x00007fd62eb87577  PyException_GetContext+7  jne    0x7fd62eb8757a <PyException_GetContext+10>
 0x00007fd62eb87579  PyException_GetContext+9  retq   
 0x00007fd62eb8757a  PyException_GetContext+10 addq   $0x1,(%rax)
─── Breakpoints ────────────────────────────────────────────────────────────────
─── Expressions ────────────────────────────────────────────────────────────────
─── History ────────────────────────────────────────────────────────────────────
─── Memory ─────────────────────────────────────────────────────────────────────
─── Registers ──────────────────────────────────────────────────────────────────
   rax 0x00007fd61fe53580   rbx 0x00007fd61fe53580      rcx 0x0000000000000000
   rdx 0x00007fd61fe53a00   rsi 0x00007fd62ed18a20      rdi 0x00007fd61fe53580
   rbp 0x00007fd62ed18a20   rsp 0x00007ffdc884cd58       r8 0x0000000000000004
    r9 0x00005609a47be5f0   r10 0x00007fd61fe53580      r11 0x0000000000000050
   r12 0x00007fd61fe53a00   r13 0x00007fd61fe53580      r14 0x00005609a47bd420
   r15 0x00007fd62ed25540   rip 0x00007fd62eb87570   eflags [ IF ]            
    cs 0x00000033            ss 0x0000002b               ds 0x00000000        
    es 0x00000000            fs 0x00000000               gs 0x00000000        
─── Source ─────────────────────────────────────────────────────────────────────
 345  }
 346  
 347  PyObject *
 348  PyException_GetContext(PyObject *self)
 349  {
 350>     PyObject *context = _PyBaseExceptionObject_cast(self)->context;
 351      Py_XINCREF(context);
 352      return context;
 353  }
 354  
─── Stack ──────────────────────────────────────────────────────────────────────
[0] from 0x00007fd62eb87570 in PyException_GetContext+0 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/exceptions.c:350
[1] from 0x00007fd62eb15a38 in _PyErr_SetObject+456 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:145
[2] from 0x00007fd62eb1e95d in _PyErr_SetNone+17 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:199
[3] from 0x00007fd62eb1e95d in PyErr_SetNone+17 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:199
[4] from 0x00007fd62eb1e95d in gen_send_ex+509 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:241
[5] from 0x00007fd62ebdaa81 in _gen_throw+225 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:513
[6] from 0x00007fd62ebda97a in gen_throw+122 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:535
[7] from 0x00007fd62eb20180 in method_vectorcall_VARARGS+176 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/descrobject.c:313
[8] from 0x00007fd62eb0a4aa in _PyObject_VectorcallTstate+46 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Include/cpython/abstract.h:118
[9] from 0x00007fd62eb0a4aa in PyObject_Vectorcall+80 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Include/cpython/abstract.h:127
[+]
─── Threads ────────────────────────────────────────────────────────────────────
[2] id 37996 name python3 from 0x00007fd62e9eaa24 in futex_abstimed_wait_cancelable+42 at ../sysdeps/nptl/futex-internal.h:320
[1] id 37995 name python3 from 0x00007fd62eb87570 in PyException_GetContext+0 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/exceptions.c:350
─── Variables ──────────────────────────────────────────────────────────────────
arg self = 0x7fd61fe53580: {ob_refcnt = 19,ob_type = 0x5609a4c818d0}
loc context = <optimized out>
────────────────────────────────────────────────────────────────────────────────
Missing separate debuginfos, use: dnf debuginfo-install expat-2.2.8-2.fc32.x86_64
>>> 
```

At a glance, we see two operating system threads are running. Let's start by looking at what thread 2 is up to:

```
>>> thread 2
[Switching to thread 2 (Thread 0x7fd61fe19700 (LWP 37996))]
#0  futex_abstimed_wait_cancelable (private=0, abstime=0x0, clockid=0, expected=0, futex_word=0x7fd618001450) at ../sysdeps/nptl/futex-internal.h:320
320	  int err = lll_futex_clock_wait_bitset (futex_word, expected,
>>> bt
0  futex_abstimed_wait_cancelable (private=0, abstime=0x0, clockid=0, expected=0, futex_word=0x7fd618001450) at ../sysdeps/nptl/futex-internal.h:320
#1  do_futex_wait (sem=sem@entry=0x7fd618001450, abstime=0x0, clockid=0) at sem_waitcommon.c:112
#2  0x00007fd62e9eab28 in __new_sem_wait_slow (sem=sem@entry=0x7fd618001450, abstime=0x0, clockid=0) at sem_waitcommon.c:184
#3  0x00007fd62e9eaba1 in __new_sem_wait (sem=sem@entry=0x7fd618001450) at sem_wait.c:42
#4  0x00007fd62eb73a5e in PyThread_acquire_lock_timed (lock=0x7fd618001450, microseconds=<optimized out>, intr_flag=1) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/thread_pthread.h:463
#5  0x00007fd62eb821c1 in acquire_timed (lock=0x7fd618001450, timeout=<optimized out>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Modules/_threadmodule.c:63
...
```

This looks a bit intimidating if you're not familiar with C. Fortunately, gdb
knows about Python:

```
>>> py-bt
Traceback (most recent call first):
  File "/usr/lib64/python3.9/threading.py", line 312, in wait
    waiter.acquire()
  File "/usr/lib64/python3.9/queue.py", line 171, in get
    self.not_empty.wait()
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/_threads/_threadworker.py", line 45, in work
    for task in iter(queue.get, _stop):
  File "/usr/lib64/python3.9/threading.py", line 888, in run
    self._target(*self._args, **self._kwargs)
  File "/usr/lib64/python3.9/threading.py", line 950, in _bootstrap_inner
    self.run()
  File "/usr/lib64/python3.9/threading.py", line 908, in _bootstrap
    self._bootstrap_inner()
```

One thing to note here is that the call stack is "upside down" when compared to
Python tracebacks you're likely used to, with the most recent call at the *top*
of the stack. Perhaps unsurprisingly given the C stack, we see this thread is
waiting for something. This doesn't seem particularly unusual, so lets see what
the other thread is up to:

```
>>> thread 1
[Switching to thread 1 (Thread 0x7fd62e881740 (LWP 37995))]
#0  PyException_GetContext (self=<StompCancelledError at remote 0x7fd61fe53580>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/exceptions.c:350
350	    PyObject *context = _PyBaseExceptionObject_cast(self)->context;
>>> py-bt
Traceback (most recent call first):
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/python/failure.py", line 512, in throwExceptionIntoGenerator
    return g.throw(self.type, self.value, self.tb)
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 1672, in _inlineCallbacks
    return succeed(False)
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 1475, in gotResult
    _inlineCallbacks(r, g, status)
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 910, in _runCallbacks
    if iscoroutine(coro) or isinstance(coro, GeneratorType):
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 568, in _startRunCallbacks
    self._runCallbacks()
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 501, in errback
    self._startRunCallbacks(fail)
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 2488, in _inlineCallbacks
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 1475, in gotResult
    _inlineCallbacks(r, g, status)
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 910, in _runCallbacks
    if iscoroutine(coro) or isinstance(coro, GeneratorType):
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 568, in _startRunCallbacks
    self._runCallbacks()
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 501, in errback
    self._startRunCallbacks(fail)
  File "/home/jcline/packaging/python-stompest/stompest/src/twisted/stompest/twisted/listener.py", line 111, in onConnectionLost
    connection.disconnected.errback(self._disconnectReason)
  File "/home/jcline/packaging/python-stompest/stompest/src/twisted/stompest/twisted/client.py", line 347, in <lambda>
    yield self._notify(lambda l: l.onConnectionLost(self, reason))
  File "/home/jcline/packaging/python-stompest/stompest/src/twisted/stompest/twisted/client.py", line 337, in _notify
    yield notify(listener)
  File "/home/jcline/.virtualenvs/stompest/lib64/python3.9/site-packages/twisted/internet/defer.py", line 1674, in _inlineCallbacks
  ...
```

This call stack is very deep due to pytest, pluggy, trail, and twisted so I
have truncated it. The most interesting call is the top one in any case. It
appears we're in the process of throwing an exception, which as a general rule
shouldn't hang so this thread looks a lot more suspicious than the other
thread. However, it's not clear why it's hanging, so it's time to roll up our
sleeves at what CPython is actually up to:

```
>>> bt
#0  PyException_GetContext (self=<StompCancelledError at remote 0x7fd61fe53580>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/exceptions.c:350
#1  0x00007fd62eb15a38 in _PyErr_SetObject (tstate=0x5609a47bd420, exception=<type at remote 0x7fd62ed18a20>, value=StopIteration()) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:145
#2  0x00007fd62eb1e95d in _PyErr_SetNone (exception=<optimized out>, tstate=<optimized out>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:199
#3  PyErr_SetNone (exception=<optimized out>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:199
#4  gen_send_ex (gen=0x7fd61fe82c10, arg=<optimized out>, exc=<optimized out>, closing=<optimized out>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:241
#5  0x00007fd62ebdaa81 in _gen_throw (gen=0x7fd61fe82c10, close_on_genexit=<optimized out>, typ=<optimized out>, val=<optimized out>, tb=<optimized out>) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:513
#6  0x00007fd62ebda97a in gen_throw (gen=0x7fd61fe82c10, args=args@entry=(<type at remote 0x5609a4c818d0>, <StompCancelledError at remote 0x7fd61fe53580>, <traceback at remote 0x7fd61f5f18c0>)) at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:535
```

Okay, so it looks like we're in the exception portion of CPython (not shocking
given our look at the Python stack) in a function getting some "context". It
doesn't appear that anything is intentionally waiting on some event, so lets
step through the program a bit to see what we're busy doing. It turns out
`PyException_GetContext` doesn't do much and returns to the caller just fine:

```
>>> n
>>> n
>>> n
─── Output/messages ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
147	                if (context == value) {
─── Assembly ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 0x00007fd62eb15a7e  _PyErr_SetObject+526 mov    %rax,%r15
 0x00007fd62eb15a81  _PyErr_SetObject+529 jmp    0x7fd62eb15a0d <_PyErr_SetObject+413>
 0x00007fd62eb15a83  _PyErr_SetObject+531 mov    %rax,%rdi
 0x00007fd62eb15a86  _PyErr_SetObject+534 mov    %rax,0x8(%rsp)
 0x00007fd62eb15a8b  _PyErr_SetObject+539 callq  0x7fd62eaf6e20 <_Py_DECREF>
 0x00007fd62eb15a90  _PyErr_SetObject+544 mov    0x8(%rsp),%r10
 0x00007fd62eb15a95  _PyErr_SetObject+549 cmp    %r10,%r12
 0x00007fd62eb15a98  _PyErr_SetObject+552 jne    0x7fd62eb15a2d <_PyErr_SetObject+445>
 0x00007fd62eb15a9a  _PyErr_SetObject+554 jmpq   0x7fd62ea65487 <_PyErr_SetObject-721897>
 0x00007fd62eb15a9f  _PyErr_SetObject+559 xor    %edx,%edx
─── Breakpoints ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
─── Expressions ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
─── History ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
─── Memory ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
─── Registers ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    rax 0x00007fd61fe53580     rbx 0x00007fd61fe53580    rcx 0x0000000000000000       rdx 0x00007fd61fe53a00    rsi 0x00007fd62ed18a20    rdi 0x00007fd61fe53580    rbp 0x00007fd62ed18a20
    rsp 0x00007ffdc884cd60      r8 0x0000000000000004     r9 0x00005609a47be5f0       r10 0x00007fd61fe53580    r11 0x0000000000000050    r12 0x00007fd61fe53a00    r13 0x00007fd61fe53580
    r14 0x00005609a47bd420     r15 0x00007fd62ed25540    rip 0x00007fd62eb15a90    eflags [ IF ]                 cs 0x00000033             ss 0x0000002b             ds 0x00000000        
     es 0x00000000              fs 0x00000000             gs 0x00000000        
─── Source ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 142             to inline the call to PyException_GetContext. */
 143          if (exc_value != value) {
 144              PyObject *o = exc_value, *context;
 145              while ((context = PyException_GetContext(o))) {
 146                  Py_DECREF(context);
 147>                 if (context == value) {
 148                      PyException_SetContext(o, NULL);
 149                      break;
 150                  }
 151                  o = context;
 152              }
─── Stack ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[0] from 0x00007fd62eb15a90 in _PyErr_SetObject+544 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:147
[1] from 0x00007fd62eb1e95d in _PyErr_SetNone+17 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:199
[2] from 0x00007fd62eb1e95d in PyErr_SetNone+17 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:199
[3] from 0x00007fd62eb1e95d in gen_send_ex+509 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:241
[4] from 0x00007fd62ebdaa81 in _gen_throw+225 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:513
[5] from 0x00007fd62ebda97a in gen_throw+122 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/genobject.c:535
[6] from 0x00007fd62eb20180 in method_vectorcall_VARARGS+176 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Objects/descrobject.c:313
[7] from 0x00007fd62eb0a4aa in _PyObject_VectorcallTstate+46 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Include/cpython/abstract.h:118
[8] from 0x00007fd62eb0a4aa in PyObject_Vectorcall+80 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Include/cpython/abstract.h:127
[9] from 0x00007fd62eb0a4aa in call_function+154 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/ceval.c:5099
[+]
─── Threads ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[2] id 37996 name python3 from 0x00007fd62e9eaa24 in futex_abstimed_wait_cancelable+42 at ../sysdeps/nptl/futex-internal.h:320
[1] id 37995 name python3 from 0x00007fd62eb15a90 in _PyErr_SetObject+544 at /usr/src/debug/python39-3.9.0~b1-1.fc32.x86_64/Python/errors.c:147
─── Variables ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
arg tstate = 0x5609a47bd420: {prev = 0x5609a49fd300,next = 0x0,interp = 0x5609a47be370,frame = Frame 0…, exception = <type at remote 0x7fd62ed18a20>: {ob_refcnt = 7,ob_type = 0x7fd62ed237c0 <PyType_Type>}, value = StopIteration(): {ob_refcnt = 1,ob_type = 0x7fd62ed18a20 <_PyExc_StopIteration>}
loc o = <StompCancelledError at remote 0x7fd61fe53580>: {ob_refcnt = 19,ob_type = 0x5609a4c818d0}, context = <StompCancelledError at remote 0x7fd61fe53580>: {ob_refcnt = 19,ob_type = 0x5609a4c818d0}, exc_value = <StompCancelledError at remote 0x7fd61fe53580>: {ob_refcnt = 19,ob_type = 0x5609a4c818d0}, tb = 0x0: Cannot access memory at address 0x0
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
>>> 
```

Now here's something a little more interesting. That `while ((context =
PyException_GetContext(o)))` is just the sort of thing that could hang forever.
We can see that the loop retrieves a `context` from `o`, checks it against a
`value`, and then updates `o` to be `context`. Those variables are visible in
the variables section above, but for clarity:

```
>>> p o
$1 = <StompCancelledError at remote 0x7fd61fe53580>
>>> p context
$2 = <StompCancelledError at remote 0x7fd61fe53580>
>>> p value
$3 = StopIteration()
>>>
```

Well, well, well. Since `o` is the exact same as `context`, the update at the
end of the loop that is presumably supposed to create an exit condition will
never occur.

Now that we know what's wrong, it's a very good time to do some hunting to
determine if this bug has already been found. This is a process I can give very
little advice on since it appears to be a rule of the universe that all bug
reports are duplicates. What I did in this case was:

```
dev in ~/devel/c/cpython on  3.9 via 🐍 v3.8.3
❯ git log --grep="SetObject"  # Maybe the function name is in a commit fixing it

commit 7f77ac463cff219e0c8afef2611cad5080cc9df1
Author: Miss Islington (bot) <31488909+miss-islington@users.noreply.github.com>
Date:   Fri May 22 14:35:22 2020 -0700

    bpo-40696: Fix a hang that can arise after gen.throw() (GH-20287)
    
    This updates _PyErr_ChainStackItem() to use _PyErr_SetObject()
    instead of _PyErr_ChainExceptions(). This prevents a hang in
    certain circumstances because _PyErr_SetObject() performs checks
    to prevent cycles in the exception context chain while
    _PyErr_ChainExceptions() doesn't.
    (cherry picked from commit 7c30d12bd5359b0f66c4fbc98aa055398bcc8a7e)
    
    Co-authored-by: Chris Jerdonek <chris.jerdonek@gmail.com>
```

Ding ding ding. We can check out the [GitHub pull
request](https://github.com/python/cpython/pull/20287) and [Python
issue](https://bugs.python.org/issue40696) to see that all this debugging has
happened before (and all this debugging will happen again).

The quality of commit messages vary from project to project, so this won't
always work, but it's a good start. If nothing obvious shows up in the commit
log, try testing with the development branch of whatever is causing you pain.
Maybe it's fixed. If that fails, skim through the issue tracker (if the project
has an issue tracker) and hope for the best.
