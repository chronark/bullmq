# BullMQ For Python

This is the official BullMQ Python library. It is a close port of the NodeJS version of the library.
Python Queues are interoperable with NodeJS Queues, as both libraries use the same .lua scripts that
power all the functionality.

## Features

Currently, the library does not support all the features available in the NodeJS version. The following
have been ported so far:

- [ ] Add jobs to queues.

  - [x] Regular jobs.
  - [ ] Delayed jobs.
  - [ ] Job priority.
  - [ ] Repeatable.

- [ ] Workers
- [ ] Job events.
- [ ] Job progress.
- [ ] Job retries.
- [ ] Job backoff.
- [ ] Getters.

## Installation

```bash
pip install bullmq
```

## Usage

```python
from bullmq import Queue

queue = Queue('my-queue')

job = await queue.add('my-job', {'foo': 'bar'})

```

## Documentation

The documentation is available at [https://docs.bullmq.io](https://docs.bullmq.io/python)

## License

MIT

## Copyright

Copyright (c) 2018-2023, Taskforce.sh Inc. and other contributors.
