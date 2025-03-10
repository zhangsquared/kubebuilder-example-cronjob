# kubebuilder-example-cronjob

[cronjob tutorial](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial)

## Specs

### A kubebuilder project

- `go.mod`: A new Go module matching our project, with basic dependencies
- `Makefile`: Make targets for building and deploying your controller
- `PROJECT`: Kubebuilder metadata for scaffolding new components

### Designing CronJob API

Fundamentally a CronJob needs the following pieces

- A schedule (the cron in CronJob)
- A template for the Job to run (the job in CronJob)

Optionally, we can add a few extras, which will make our users' lives easier:

- A deadline for starting jobs (if we miss this deadline, we’ll just wait till the next scheduled time)
- What to do if multiple jobs would run at once (do we wait? stop the old one? run both?)
- A way to pause the running of a CronJob, in case something’s wrong with it
- Limits on old job history

To create

```bash
kubebuilder create api --group batch --version v1 --kind CronJob
```
### CronJob Controller

The basic logic of our CronJob controller is this:

1. Load the named CronJob
2. List all active jobs, and update the status
3. Clean up old jobs according to the history limits
4. Check if we’re suspended (and don’t do anything else if we are)
5. Get the next scheduled run
6. Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy
7. Requeue when we either see a running job (done automatically) or it’s time for the next scheduled run.

### Admission Webhooks

Add `CustomDefaulter` and (or) the `CustomValidator` interface.

Kubebuilder takes care of the rest for you, such as

- Creating the webhook server.
- Ensuring the server has been added in the manager.
- Creating handlers for your webhooks.
- Registering each handler with a path in your server.

```bash
kubebuilder create webhook --group batch --version v1 --kind CronJob --defaulting --programmatic-validation
```

- `--defaulting` flag makes defaulting flag
- `--programmatic-validation` flag makes validating webhooks

## Test, Build and Deploy

1. Run locally

Setting up [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#interacting-with-your-cluster) cluster

```bash
kind create cluster
```

Check cluster context

```bash
k ctx
```

2. Install CRDs on the cluster and run

```bash
make manifests
make install

export ENABLE_WEBHOOKS=false
make run
```

3. Test

```bash
k apply -f config/samples/batch_v1_cronjob.yaml
```

4. Delete the cluster

```bash
kind delete cluster
```

## Random notes

- Designing an AOI: `omitempty` struct tag to mark that a field should be omitted from serialization when empty
- `zz_generated.deepcopy.go` contains the autogenerated implementation of the aforementioned `runtime.Object` interface, which marks all of our root types as representing Kinds.
