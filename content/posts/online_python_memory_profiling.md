---
title: 'Dumping Python heap in production'
date: 2025-04-19T14:46:35+08:00
tags:
- linux
- python
---

Live debugging Python process's memory!

<!--more-->

# The problem

You got called due to an outage of AI/LLM services written in Python and it
turned out to be a memory leak in production. Before restoring the service to
mitigate the impact, you may want to capture as much context as possible.
Luckily, there are many libraries/tools for memory profiling in the ecosystem,
like [scalene](https://github.com/plasma-umass/scalene) or
[pympler](https://github.com/pympler/pympler):

```python
>>> from pympler import muppy, summary
>>> all_objects = muppy.get_objects()
>>> sum1 = summary.summarize(all_objects)
>>> summary.print_(sum1)
                       types |   # objects |   total size
============================ | =========== | ============
                        dict |         546 |    953.30 KB
                         str |        8270 |    616.46 KB
                        list |         127 |    529.44 KB
                       tuple |        5021 |    410.62 KB
                        code |        1378 |    161.48 KB
                        type |          70 |     61.80 KB
          wrapper_descriptor |         508 |     39.69 KB
  builtin_function_or_method |         515 |     36.21 KB
                         int |         900 |     21.09 KB
           method_descriptor |         269 |     18.91 KB
                     weakref |         177 |     15.21 KB
         <class 'abc.ABCMeta |          16 |     14.12 KB
                         set |          48 |     10.88 KB
         function (__init__) |          81 |      9.49 KB
           member_descriptor |         131 |      9.21 KB
```

However, most of them share on major shortage: **we can't use them to analyze
the memory of an already-running Python process** (actually you can add an HTTP
endpoint dumping heap info/start tracking memory allocation if you have included
these libraries as dependencies).

It would be much better if we could have something like
[py-spy](https://github.com/benfred/py-spy) or
[parca](https://github.com/parca-dev/parca) but for memory; they are more than
capable for live debugging cpu issues in production.

# Can eBPF help?

[eBPF](https://ebpf.io/) is a tool to run user programs in privileged mode on
Linux, and many low-overhead, non-intrusive observability tools are built on it,
including the aforementioned [parca](https://github.com/parca-dev/parca) for CPU
profiling.

There are following tools for live debugging Python in
[bcc toolkit](https://github.com/iovisor/bcc) but unfortunately they are not
very uselful in memory debugging:

- [pythongc based on ugc](https://github.com/iovisor/bcc/blob/master/tools/lib/ugc_example.txt):
  tracing python's gc events
- [pythoncalls based on ucall](https://github.com/iovisor/bcc/blob/master/tools/lib/ucalls_example.txt):
  tracing number of calls and duration of each function call
- [pythonflow based on uflow](https://github.com/iovisor/bcc/blob/master/tools/lib/uflow_example.txt):
  function call stack trace
- [pythonstat based on ustat](https://github.com/iovisor/bcc/blob/master/tools/lib/ustat_example.txt):
  count of garbage collections, method calls, object allocations per seconds

# Code Injection

With gdb, we can attach to a running Python process and inject python code for
execution. [Pyrasite](https://github.com/lmacken/pyrasite) and
[debug-toolkit](https://github.com/robusta-dev/debug-toolkit) are two projects
following this route. Here we would use the latter one to demonstrate how we can
dump the heap of a running Python process.

## Requirements

- have gdb on your Linux machine
- install [Poetry](https://github.com/robusta-dev/debug-toolkit)
- the injected Python code could not import any dependency not installed by the
  running Python environment

## How - BareMetal

```bash
git clone https://github.com/robusta-dev/debug-toolkit.git
cd debug-toolkit
poetry shell
poetry install
python src/debug_toolkit/main.py --help

# to inject code by string; 12345 is the PID
python src/debug_toolkit/main.py inject-string 12345 "f = open('test', 'w'); f.write('hello world'); f.close()"

# to inject code by a python script file
python src/debug_toolkit/main.py inject-file 12345 /path/to/python/file.py
# you can find the created file `test` under `/proc/12345/cwd`
```

- for example, you can get a glance at the object allocation via the following
  inject python file:

```python
import gc
import sys

gc.collect()
all_objects = gc.get_objects()
class_info = {}
for obj in all_objects:
  obj_class = obj.__class__
  obj_size = sys.getsizeof(obj)
  if obj_class not in class_info:
    class_info[obj_class] = {'count': 1, 'total_size': obj_size}
  else:
    class_info[obj_class]['count'] += 1
    class_info[obj_class]['total_size'] += obj_size

  refs = gc.get_referrers(obj)
  for ref in refs:
    if gc.is_tracked(ref):
      continue
    obj_class = type(obj)
    obj_size = sys.getsizeof(obj)
    if obj_class not in class_info:
      class_info[obj_class] = {'count': 1, 'total_size': obj_size}
    else:
      class_info[obj_class]['count'] += 1
      class_info[obj_class]['total_size'] += obj_size

sorted_classes = sorted(class_info.items(), key=lambda x: x[1]['total_size'], reverse=True)

with open('heap_alloc.txt', 'w') as f:
  for i, (cls, info) in enumerate(sorted_classes):
    f.write(f'{i} {cls.__name__} {info["count"]} {info["total_size"]}\n')
```

## How - Kubernetes

- `kubectl apply` the following deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-tools
  labels:
    app: python-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python-tools
  template:
    metadata:
      labels:
        app: python-tools
    spec:
      hostPID: true
      containers:
        - name: python-tools
          image: robustadev/debug-toolkit:v7.0.1
          imagePullPolicy: Always
          securityContext:
            privileged: true
            capabilities:
              add:
                - SYS_PTRACE
```

- `kubectl exec` into the pod, find the pid of the target container process on
  the same k8s worker node and then execute

```bash
# to inject code by string; 12345 is the PID of the process inside the pod/container
# because of hostPID, you can `ps aux | grep python` to find
debug-toolkit inject-string 12345 "f = open('test', 'w'); f.write('hello world'); f.close()"
# you can find the created file `test` under `/proc/12345/cwd`
```

## How - only with gdb!

- actually we can achieve the same without using any dependency, but rather
  inconvenient
- create a file named inject_code; here `{python_code}` is the injected python
  code snippet. Remeber to escape any relevant characters here
  `python_code.replace("\\", "\\\\").replace('"', '\\"').replace("\n", "\\n")`

```bash
set trace-commands on
set logging on
set scheduler-locking off
call ((int (*)())PyGILState_Ensure)()
call ((int (*)(const char *))PyRun_SimpleString)("{python_code}")
call ((void (*) (int) )PyGILState_Release)($1)
```

then

```bash
gdb -p <pid> --batch --command=inject_code
```
