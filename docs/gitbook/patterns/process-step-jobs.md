# Process Step jobs

Sometimes, it is useful to break processor function into small pieces that will be processed depending on the previous executed step, we could handle this kind of logic by using switch blocks:

```typescript
enum Step {
  Initial,
  Second,
  Finish,
}

const worker = new Worker(
  queueName,
  async job => {
    let step = job.data.step;
    while (step !== Step.Finish) {
      switch (step) {
        case Step.Initial: {
          await doInitialStepStuff();
          await job.update({
            step: Step.Second,
          });
          step = Step.Second;
          break;
        }
        case Step.Second: {
          await doSecondStepStuff();
          await job.update({
            step: Step.Finish,
          });
          step = Step.Finish;
          return Step.Finish;
        }
        default: {
          throw new Error('invalid step');
        }
      }
    }
  },
  { connection },
);
```

As you can see, we should save the step value; in this case, we are saving it into the job's data. So even in the case of an error, it would be retried in the last step that was saved (in case we use a backoff strategy).

# Delaying

Another use case is to delay a job at runtime.

This could be handled using the moveToDelayed method:

```typescript
import { DelayedError, Worker } from 'bullmq';

enum Step {
  Initial,
  Second,
  Finish,
}

const worker = new Worker(
  queueName,
  async job => {
    let step = job.data.step;
    while (step !== Step.Finish) {
      switch (step) {
        case Step.Initial: {
          await doInitialStepStuff();
          await job.moveToDelayed(Date.now() + 200, token);
          await job.update({
            step: Step.Second,
          });
          step = Step.Second;
          break;
        }
        case Step.Second: {
          await doSecondStepStuff();
          await job.update({
            step: Step.Finish,
          });
          throw new DelayedError();
        }
        default: {
          throw new Error('invalid step');
        }
      }
    }
  },
  { connection },
);
```

# Waiting Children

A common use case is to add children at runtime and then wait for the children to complete.

This could be handled using the moveToWaitingChildren method:

```typescript
import { WaitingChildrenError, Worker } from 'bullmq';

enum Step {
  Initial,
  Second,
  Third,
  Finish,
}

const worker = new Worker(
  parentQueueName,
  async (job, token) => {
    let step = job.data.step;
    while (step !== Step.Finish) {
      switch (step) {
        case Step.Initial: {
          await doInitialStepStuff();
          await childrenQueue.add(
            'child-1',
            { foo: 'bar' },
            {
              parent: {
                id: job.id,
                queue: job.queueQualifiedName,
              },
            },
          );
          await job.update({
            step: Step.Second,
          });
          step = Step.Second;
          break;
        }
        case Step.Second: {
          await doSecondStepStuff();
          await childrenQueue.add(
            'child-2',
            { foo: 'bar' },
            {
              parent: {
                id: job.id,
                queue: job.queueQualifiedName,
              },
            },
          );
          await job.update({
            step: Step.Third,
          });
          step = Step.Third;
          break;
        }
        case Step.Third: {
          const shouldWait = await job.moveToWaitingChildren(token);
          if (!shouldWait) {
            await job.update({
              step: Step.Finish,
            });
            step = Step.Finish;
            return Step.Finish;
          } else {
            throw new WaitingChildrenError();
          }
        }
        default: {
          throw new Error('invalid step');
        }
      }
    }
  },
  { connection },
);
```

{% hint style="info" %}
Bullmq-Pro: this pattern could be handled by using observables; in that case, we do not need to save next step.
{% endhint %}

# Chaining Flows

Another use case is to add flows at runtime and then wait for the children to complete.

For example, we can add children dynamically in the processor function of a worker. This could be handled in this way:

```typescript
import { FlowProducer, WaitingChildrenError, Worker } from 'bullmq';

enum Step {
  Initial,
  Second,
  Third,
  Finish,
}

const flow = new FlowProducer({ connection });
const worker = new Worker(
  parentQueueName,
  async (job, token) => {
    let step = job.data.step;
    while (step !== Step.Finish) {
      switch (step) {
        case Step.Initial: {
          await doInitialStepStuff();
          await flow.add({
            name: 'child-job',
            queueName: 'childrenQueueName',
            data: {},
            children: [
              {
                name,
                data: { idx: 0, foo: 'bar' },
                queueName: 'grandchildrenQueueName',
              },
              {
                name,
                data: { idx: 1, foo: 'baz' },
                queueName: 'grandchildrenQueueName',
              },
            ],
            opts: {
              parent: {
                id: job.id,
                queue: job.queueQualifiedName,
              },
            },
          });

          await job.update({
            step: Step.Second,
          });
          step = Step.Second;
          break;
        }
        case Step.Second: {
          await doSecondStepStuff();
          await job.update({
            step: Step.Third,
          });
          step = Step.Third;
          break;
        }
        case Step.Third: {
          const shouldWait = await job.moveToWaitingChildren(token);
          if (!shouldWait) {
            await job.update({
              step: Step.Finish,
            });
            step = Step.Finish;
            return Step.Finish;
          } else {
            throw new WaitingChildrenError();
          }
        }
        default: {
          throw new Error('invalid step');
        }
      }
    }
  },
  { connection },
);
```
