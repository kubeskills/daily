## Day 85: Job and CronJob Disasters - Batch Processing Failures

### Email Copy

**Subject:** Day 85: Job and CronJob Disasters - When Scheduled Tasks Fail

THE IDEA:
Schedule batch jobs and watch them fail. Jobs never complete, CronJobs miss schedules,
pods retry infinitely, old jobs accumulate, and deadlines get exceeded.

THE SETUP:
Create failing Jobs, overlap CronJobs, exceed activeDeadlineSeconds, hit backoffLimit,
and discover what breaks when batch processing fails.

WHAT I LEARNED:
- Jobs retry failed pods up to backoffLimit
- CronJobs create Jobs on schedule
- concurrencyPolicy controls overlapping runs
- activeDeadlineSeconds provides job timeout
- successfulJobsHistoryLimit prevents buildup
- Pods must have restartPolicy: OnFailure or Never

WHY IT MATTERS:
Job failures cause:
- Data processing pipelines stall
- Scheduled maintenance doesn't run
- Infinite retries consume resources
- Old jobs fill namespace
- Reports and exports never complete

TRY IT:
🧪 Interactive Lab: https://killercoda.com/playgrounds/scenario/kubernetes

Tomorrow: Custom Resource Definition (CRD) validation failures.

---
Chad


---

## Killercoda Lab Instructions

## Step 1: Create basic Job

```bash
# Simple successful Job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: success-job
spec:
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ['sh', '-c', 'echo "Job completed successfully"; exit 0']
      restartPolicy: Never
  backoffLimit: 3
EOF
```

```bash
# Wait for completion
kubectl wait --for=condition=complete job/success-job --timeout=60s
```

```bash
# Check status
kubectl get job success-job
kubectl get pods -l job-name=success-job
```

## Step 2: Test Job with wrong restartPolicy

```bash
# Job with invalid restartPolicy
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: wrong-restart-policy
spec:
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ['sh', '-c', 'echo "Running"; sleep 10']
      restartPolicy: Always  # Invalid for Jobs!
EOF

echo "Job creation failed: restartPolicy must be Never or OnFailure"
kubectl get job wrong-restart-policy 2>&1 || echo "Jobs require restartPolicy: Never or OnFailure"
```

## Step 3: Test Job hitting backoffLimit

```bash
# Job that fails repeatedly
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
spec:
  backoffLimit: 3  # Max 3 retries
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ['sh', '-c', 'echo "Failing on purpose"; exit 1']
      restartPolicy: Never
EOF
```

```bash
# Wait for failure
sleep 60
```

```bash
# Check status
kubectl get job failing-job
kubectl get pods -l job-name=failing-job

echo ""
echo "Job created 4 pods (1 initial + 3 retries), then marked Failed"
kubectl describe job failing-job | grep -A 5 "Events:"
```

## Step 4: Test Job exceeding activeDeadlineSeconds

```bash
# Job with timeout
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
spec:
  activeDeadlineSeconds: 10  # Timeout after 10 seconds
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ['sh', '-c', 'sleep 60']  # Takes 60 seconds
      restartPolicy: Never
EOF
```

```bash
# Wait for timeout
sleep 15
```

```bash
# Job terminated due to deadline
kubectl get job timeout-job
kubectl describe job timeout-job | grep -i "deadline\|exceeded"

echo ""
echo "Job exceeded activeDeadlineSeconds and was terminated"
```

## Step 5: Test Job with parallelism

```bash
# Job with parallel pods
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 6
  parallelism: 3  # Run 3 pods at once
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ['sh', '-c', 'echo "Pod $HOSTNAME running"; sleep 5']
      restartPolicy: Never
EOF
```

```bash
# Watch pods being created
kubectl get pods -l job-name=parallel-job -w &
WATCH_PID=$!

sleep 20

kill $WATCH_PID 2>/dev/null
```

```bash
# Check completion
kubectl get job parallel-job
echo ""
echo "Job runs 3 pods at a time until 6 completions achieved"
```

## Step 6: Test CronJob basic schedule

```bash
# CronJob running every minute
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: minute-cronjob
spec:
  schedule: "*/1 * * * *"  # Every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: busybox
            command: ['sh', '-c', 'date; echo "CronJob executed"']
          restartPolicy: Never
EOF
```

```bash
# Wait for first run
sleep 70
```

```bash
# Check created jobs
kubectl get jobs -l cronjob=minute-cronjob

echo ""
echo "CronJob creates a new Job every minute"
```

## Step 7: Test CronJob missing schedule

```bash
# CronJob with long-running job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: slow-cronjob
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid  # Don't allow overlap
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: busybox
            command: ['sh', '-c', 'echo "Starting slow job"; sleep 120']
          restartPolicy: Never
EOF
```

```bash
# Wait for schedule misses
sleep 150
```

```bash
# Check for missed schedules
kubectl get cronjob slow-cronjob
kubectl describe cronjob slow-cronjob | grep -A 5 "Events:"

echo ""
echo "Job still running when next schedule arrives - CronJob misses schedule"
```

## Step 8: Test CronJob concurrency policies

```bash
# Test Allow policy (multiple jobs can run)
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: allow-cronjob
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Allow  # Allow concurrent jobs
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: busybox
            command: ['sh', '-c', 'echo "Starting"; sleep 90']
          restartPolicy: Never
EOF

sleep 130
```

```bash
# Multiple jobs running
kubectl get jobs -l cronjob=allow-cronjob

echo ""
echo "Allow: Multiple jobs run concurrently"
```

```bash
# Test Replace policy
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: replace-cronjob
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Replace  # Kill old job, start new one
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: busybox
            command: ['sh', '-c', 'echo "Starting"; sleep 90']
          restartPolicy: Never
EOF

echo ""
echo "Replace: New job replaces running job"
echo "Forbid: New job skipped if previous still running"
```

## Step 9: Test CronJob with invalid schedule

```bash
# Invalid cron schedule
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: invalid-schedule
spec:
  schedule: "invalid cron syntax"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: busybox
            command: ['sh', '-c', 'echo "Running"']
          restartPolicy: Never
EOF

echo ""
echo "CronJob with invalid schedule - rejected by API server"
```

<!-- Content was truncated. Please add the remaining steps. -->
