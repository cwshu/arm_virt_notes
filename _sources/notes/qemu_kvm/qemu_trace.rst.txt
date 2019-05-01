QEMU trace event
================

doc/devel/tracing.txt
---------------------

- Quickstart::

    ./configure --enable-trace-backends=simple
    qemu-system -trace events=/tmp/events
    ./scripts/simpletrace.py trace-events-all trace-<pid>

- Trace events
  
  - Sub-directory setup
  - Using trace events
  - Declaring trace events
  - Hints for adding new trace events

- Generic interface and monitor commands::

    trace/control.h

    monitor)
    info trace-events
    trace-event NAME on|off
    trace-file on|off|flush|set [arg]

- Trace backends
- Trace event properties

Note
----

how to write trace-event:

1. 在 trace-event 新增 trace function 的 prototype 跟 printf format string.
2. 在 source code 要 trace 處呼叫 trace function, 把要紀錄的資訊填入 trace function 的參數.
3. source 中需要 ``#include "trace.h"``

reference
---------

- `doc/devel/tracing.txt <https://git.qemu.org/?p=qemu.git;a=blob;f=docs/devel/tracing.txt>`_
- `Stefan - Tracing in the QEMU emulator <https://vmsplice.net/~stefan/stefanha-tracing-summit-2014.pdf>`_
- `Stefan - How to write trace analysis scripts for QEMU <http://blog.vmsplice.net/2011/03/how-to-write-trace-analysis-scripts-for.html>`_
- source code

  - hw/net/trace-event
  - e1000e-core.c, ...

More 

- Tracing QEMU-KVM Interactions <https://gist.github.com/mcastelino/b31f0648707b25478eb2a44f94a861fd>, 有介紹一點 ftrace backend
