# Day 18: CronJob Concurrency - The Job Pile-Up

## THE IDEA:
Create a CronJob that runs every minute but takes 5 minutes to complete. Watch 
jobs pile up or get blocked depending on concurrencyPolicy.

## THE SETUP:
Deploy CronJobs with Allow, Forbid, and Replace policies. Generate long-running 
jobs and observe the different failure modes.

## WHAT I LEARNED:
- Allow creates overlapping jobs (default, dangerous!)
- Forbid skips new job if previous still running
- Replace kills old job and starts new one
- successfulJobsHistoryLimit controls retention

## WHY IT MATTERS:
Concurrency issues cause:
- Resource exhaustion from job pile-ups
- Duplicate data processing
- Database locks from concurrent access

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-018

Tomorrow: ServiceAccount token mounting and projection issues.

---

## Killercoda Lab Instructions

### Step 1: Create CronJob with Allow policy (default)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: slow-job-allow
spec:
  schedule: "*/1 * * * *"  # Every minute
  concurrencyPolicy: Allow
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: sleeper
            image: busybox
            command:
            - /bin/sh
            - -c
            - echo "Job started at $(date)"; sleep 300; echo "Job finished at $(date)"
          restartPolicy: Never
EOF

```

### Step 2: Watch jobs pile up

```bash
# Watch in real-time
kubectl get cronjobs,jobs,pods -w

```

After 5 minutes, you'll have 5 concurrent jobs running!

**Check resource usage:**

```bash
kubectl get jobs -l job-name
kubectl describe cronjob slow-job-allow

```

### Step 3: Create CronJob with Forbid policy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: slow-job-forbid
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: sleeper
            image: busybox
            command:
            - /bin/sh
            - -c
            - echo "Forbid job started at $(date)"; sleep 300; echo "Forbid job finished"
          restartPolicy: Never
EOF

```

**Watch behavior:**

```bash
kubectl get cronjobs slow-job-forbid -w

```

### Step 4: Check skipped runs

```bash
# Wait 3 minutes, then check events
kubectl describe cronjob slow-job-forbid | grep -A 10 Events

```

You'll see: "Saw completed job... while previous job was still running, skipping"

Only ONE job runs at a time. Subsequent schedules are skipped!

### Step 5: Create CronJob with Replace policy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: slow-job-replace
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Replace
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: sleeper
            image: busybox
            command:
            - /bin/sh
            - -c
            - echo "Replace job started at $(date) with ID=$HOSTNAME"; sleep 300; echo "Finished"
          restartPolicy: Never
EOF

```

**Watch replacement behavior:**

```bash
kubectl get jobs,pods -l 'job-name' -w

```

Each minute, the old job is deleted and a new one starts!

### Step 6: Test job completion and history

```bash
# Create fast-completing CronJob
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: fast-job
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: quick
            image: busybox
            command: ['sh', '-c', 'echo "Quick job"; sleep 5']
          restartPolicy: Never
EOF

```

**Wait 5 minutes, then check:**

```bash
kubectl get jobs -l 'job-name'

```

Only 3 successful jobs retained (per successfulJobsHistoryLimit).

### Step 7: Suspend a CronJob

```bash
kubectl patch cronjob slow-job-allow -p '{"spec":{"suspend":true}}'

```

**Verify:**

```bash
kubectl get cronjob slow-job-allow

```

SUSPEND column shows True. No new jobs created!

**Resume:**

```bash
kubectl patch cronjob slow-job-allow -p '{"spec":{"suspend":false}}'

```

### Step 8: Test startingDeadlineSeconds

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: deadline-job
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 30  # Must start within 30s of schedule
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: app
            image: busybox
            command: ['sh', '-c', 'echo "Running"; sleep 10']
          restartPolicy: Never
EOF

```

If system is overloaded and job doesn't start within 30s, it's considered missed.

**Check for missed schedules:**

```bash
kubectl describe cronjob deadline-job | grep -A 5 "Last Schedule Time"

```

### Step 9: Job failure handling

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: failing-job
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      backoffLimit: 3  # Retry 3 times
      template:
        spec:
          containers:
          - name: failer
            image: busybox
            command: ['sh', '-c', 'echo "Attempt failed"; exit 1']
          restartPolicy: Never
EOF

```

**Watch retry behavior:**

```bash
kubectl get jobs,pods -l 'job-name' -w

```

Job creates 4 pods total (initial + 3 retries), then marks as Failed.

### Step 10: Manual job trigger

```bash
# Create a job manually from CronJob template
kubectl create job manual-run --from=cronjob/fast-job

```

**Verify:**

```bash
kubectl get job manual-run
kubectl logs job/manual-run

```

Useful for testing CronJob logic without waiting!

### Step 11: Time zone support (K8s 1.25+)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: timezone-job
spec:
  schedule: "0 9 * * *"  # 9 AM
  timeZone: "America/New_York"  # EST/EDT
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: app
            image: busybox
            command: ['sh', '-c', 'date; echo "Good morning!"']
          restartPolicy: Never
EOF

```

### Step 12: Complex schedule expressions

```bash
# Every 15 minutes during business hours
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: business-hours-job
spec:
  schedule: "*/15 9-17 * * 1-5"  # Mon-Fri, 9AM-5PM, every 15 min
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: app
            image: busybox
            command: ['sh', '-c', 'echo "Business hours task"']
          restartPolicy: Never
EOF

```

### Key Observations

✅ **Allow (default)** - jobs pile up, can exhaust resources
✅ **Forbid** - skips schedules if previous job running
✅ **Replace** - kills old job, starts new one
✅ **History limits** - control job retention
✅ **startingDeadlineSeconds** - handle missed schedules

### Production Patterns

**Data backup CronJob:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  concurrencyPolicy: Forbid  # Don't overlap backups!
  successfulJobsHistoryLimit: 7  # Keep week of history
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 900  # 15 min window to start
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command: ['pg_dump', '-h', 'db', '-U', 'user', 'mydb']
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc

```

**Metrics collection:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: metrics-collector
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  concurrencyPolicy: Replace  # Latest data wins
  successfulJobsHistoryLimit: 1  # Keep minimal history
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: collector
            image: metrics-collector:latest
          restartPolicy: Never

```

**Cleanup job:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-temp-files
spec:
  schedule: "0 3 * * 0"  # Sundays at 3 AM
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - sh
            - -c
            - find /data/temp -type f -mtime +7 -delete

```

### Cleanup

```bash
kubectl delete cronjob slow-job-allow slow-job-forbid slow-job-replace fast-job deadline-job failing-job timezone-job business-hours-job 2>/dev/null
kubectl delete job --all

```

---

Next: Day 19 - ServiceAccount Token Mounting Issues