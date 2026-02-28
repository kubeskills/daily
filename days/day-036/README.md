## Day 36: Jobs and CronJobs - Backoff Explosion

## THE IDEA:
Create a Job that fails repeatedly and watch the backoff delay grow exponentially.
Learn when Jobs give up, how completions work, and parallelism pitfalls.

## THE SETUP:
Deploy Jobs with different backoffLimit settings, trigger failures, and observe
the 10s → 20s → 40s → 80s... backoff progression. Test parallelism deadlocks.

## WHAT I LEARNED:
- backoffLimit controls total retries (default: 6)
- Backoff grows exponentially: 10s, 20s, 40s, 80s, 160s, 320s max
- Completions can exceed parallelism (creates queuing)
- Failed Jobs stay around for inspection (unless TTL set)
- activeDeadlineSeconds caps total runtime

## WHY IT MATTERS:
Job failures cause:
- Batch processing stuck waiting for backoff
- Resource waste from infinite retries
- Delayed failure notification (waiting for retries)

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-036

Tomorrow: Volume mount failures and permission errors.

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes


---

# Killercoda Lab Instructions


## Step 1: Create basic failing Job
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
spec:
  backoffLimit: 4
  template:
    spec:
      containers:
      - name: fail
        image: busybox
        command: ['sh', '-c', 'echo "Attempt failed"; exit 1']
      restartPolicy: Never
EOF
```

**Watch backoff progression:**
```bash
kubectl get pods -l job-name=failing-job -w
```

Creates 5 pods total (initial + 4 retries)

## Step 2: Check backoff timing
```bash
kubectl get events --sort-by='.lastTimestamp' | grep failing-job | tail -20
```

Look for timing between pod creations: 10s, 20s, 40s, 80s...

## Step 3: Check Job status
```bash
kubectl get job failing-job
kubectl describe job failing-job | grep -A 10 "Pods Statuses:"
```

Shows: 0 Succeeded, 5 Failed

## Step 4: Test with backoffLimit=0 (no retries)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: no-retry-job
spec:
  backoffLimit: 0
  template:
    spec:
      containers:
      - name: fail
        image: busybox
        command: ['sh', '-c', 'exit 1']
      restartPolicy: Never
EOF
```

**Check pods:**
```bash
kubectl get pods -l job-name=no-retry-job
```

Only 1 pod created - no retries!

## Step 5: Test activeDeadlineSeconds
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
spec:
  activeDeadlineSeconds: 30
  backoffLimit: 10
  template:
    spec:
      containers:
      - name: slow
        image: busybox
        command: ['sh', '-c', 'sleep 60']
      restartPolicy: Never
EOF
```

**Watch timeout:**
```bash
kubectl get job timeout-job -w
```

Job terminates after 30s regardless of backoffLimit!

## Step 6: Test completions (multiple successful runs)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion
spec:
  completions: 5
  parallelism: 1
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo "Processing..."; sleep 3; echo "Done"']
      restartPolicy: Never
EOF
```

**Watch sequential execution:**
```bash
kubectl get pods -l job-name=multi-completion -w
```

5 pods run sequentially (parallelism: 1)

## Step 7: Test parallelism
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 10
  parallelism: 3
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo "Worker $HOSTNAME"; sleep 5']
      restartPolicy: Never
EOF
```

**Watch parallel execution:**
```bash
kubectl get pods -l job-name=parallel-job -w
```

3 pods run at once until 10 completions reached

## Step 8: Test Job with OnFailure restart policy
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: onfailure-job
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: retry
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          if [ ! -f /tmp/attempted ]; then
            touch /tmp/attempted
            echo "First attempt, failing"
            exit 1
          else
            echo "Retry succeeded"
            exit 0
          fi
      restartPolicy: OnFailure
EOF
```

**Check behavior:**
```bash
kubectl get pods -l job-name=onfailure-job -w
```

Same pod restarts (not new pods like with Never)

## Step 9: Test TTL after finished
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: ttl-job
spec:
  ttlSecondsAfterFinished: 30
  template:
    spec:
      containers:
      - name: quick
        image: busybox
        command: ['sh', '-c', 'echo "Done"; sleep 2']
      restartPolicy: Never
EOF
```

**Wait and check:**
```bash
sleep 35
kubectl get job ttl-job
```

Job automatically deleted after 30s!

## Step 10: Test CronJob
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: test-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: worker
            image: busybox
            command: ['sh', '-c', 'date; echo "Cron job run"']
          restartPolicy: Never
EOF
```

**Wait for execution:**
```bash
kubectl get cronjob test-cron -w
kubectl get jobs -l parent-cronjob=test-cron
```

## Step 11: Test CronJob with failedJobsHistoryLimit
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: failing-cron
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: fail
            image: busybox
            command: ['sh', '-c', 'exit 1']
          restartPolicy: Never
EOF
```

**Wait a few minutes:**
```bash
sleep 150
kubectl get jobs | grep failing-cron
```

Only 1 failed job kept!

## Step 12: Test Job suspend
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: suspendable-job
spec:
  suspend: true
  parallelism: 5
  completions: 10
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'sleep 30']
      restartPolicy: Never
EOF
```

**Check pods:**
```bash
kubectl get pods -l job-name=suspendable-job
```

No pods created! Job is suspended.

**Resume:**
```bash
kubectl patch job suspendable-job -p '{"spec":{"suspend":false}}'
kubectl get pods -l job-name=suspendable-job -w
```

Pods start!

## Step 13: Test Job with indexed completion mode
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 2
  completionMode: Indexed
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Processing index: $JOB_COMPLETION_INDEX"
          sleep 5
      restartPolicy: Never
EOF
```

**Check environment variable:**
```bash
kubectl logs -l job-name=indexed-job --tail=5 | grep "Processing index"
```

Each pod knows its index!

## Step 14: Test parallelism deadlock scenario
```bash
# Create Job that needs more resources than available
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: resource-starved
spec:
  completions: 10
  parallelism: 10
  template:
    spec:
      containers:
      - name: hungry
        image: busybox
        command: ['sleep', '30']
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
      restartPolicy: Never
EOF
```

**Check pods:**
```bash
kubectl get pods -l job-name=resource-starved
```

Some pods Pending (insufficient resources)

## Step 15: Debug failed Job
```bash
# Get failed pods
kubectl get pods -l job-name=failing-job --field-selector=status.phase=Failed

# Check logs from failed pod
FAILED_POD=$(kubectl get pods -l job-name=failing-job --field-selector=status.phase=Failed -o jsonpath='{.items[0].metadata.name}')
kubectl logs $FAILED_POD

# Check exit code
kubectl get pod $FAILED_POD -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'
echo ""

# Get reason
kubectl get pod $FAILED_POD -o jsonpath='{.status.containerStatuses[0].state.terminated.reason}'
echo ""
```

## Key Observations

✅ **backoffLimit** - total retries before giving up
✅ **Exponential backoff** - 10s, 20s, 40s, 80s, 160s, 320s (max)
✅ **activeDeadlineSeconds** - caps total runtime
✅ **completions** - how many successful runs needed
✅ **parallelism** - how many pods run concurrently
✅ **TTL** - auto-cleanup after completion

## Production Patterns

**Batch processing Job:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 100
  parallelism: 10
  backoffLimit: 3
  activeDeadlineSeconds: 3600  # 1 hour max
  ttlSecondsAfterFinished: 86400  # Keep for 1 day
  template:
    spec:
      containers:
      - name: processor
        image: data-processor:v1
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
      restartPolicy: OnFailure
```

**Reliable CronJob:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  concurrencyPolicy: Forbid  # Don't overlap runs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 300  # Start within 5 min
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800  # 30 min max
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
          restartPolicy: OnFailure
```

**Indexed Job for distributed processing:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-task
spec:
  completions: 20
  parallelism: 5
  completionMode: Indexed
  template:
    spec:
      containers:
      - name: worker
        image: worker:latest
        env:
        - name: TASK_INDEX
          value: $(JOB_COMPLETION_INDEX)
      restartPolicy: Never
```

## Cleanup
```bash
kubectl delete job failing-job no-retry-job timeout-job multi-completion parallel-job onfailure-job ttl-job suspendable-job indexed-job resource-starved 2>/dev/null
kubectl delete cronjob test-cron failing-cron 2>/dev/null
```

---
