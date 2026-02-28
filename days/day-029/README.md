## Day 29: Custom Resource Definitions - Validation Failures


## THE IDEA:
Create a CRD with strict OpenAPI schema validation and watch custom resources 
get rejected for violating type constraints, required fields, and enum values.

## THE SETUP:
Define a CRD for a fictional "Database" resource with validation rules. Attempt 
to create instances that violate constraints and decode the validation errors.

## WHAT I LEARNED:
- CRDs use OpenAPI v3 schema for validation
- Validation happens at API server admission time
- Required fields are enforced strictly
- Enum violations provide helpful error messages
- Default values can prevent validation errors

## WHY IT MATTERS:
CRD validation issues cause:
- Custom resources rejected with cryptic JSON schema errors
- Operator controllers receiving malformed input
- Production incidents from typos in YAML (storageSize: "10Gi" vs "10GiB")

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-029

---

# Killercoda Lab Instructions



## Step 1: Check existing CRDs

```bash
kubectl get crds
kubectl api-resources | grep -i custom
```

## Step 2: Create a simple CRD without validation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            x-kubernetes-preserve-unknown-fields: true
EOF
```

**Verify CRD:**
```bash
kubectl get crd databases.example.com
```

## Step 3: Create invalid custom resource (no validation yet)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: test-db
spec:
  engine: "anything-goes"
  size: "not-a-number"
  replicas: "should-be-int"
EOF
```

**Success!** No validation = accepts anything.

```bash
kubectl get database test-db -o yaml
```

## Step 4: Delete CRD and recreate with strict validation

```bash
kubectl delete crd databases.example.com
kubectl delete database test-db 2>/dev/null

cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - engine
            - storageSize
            properties:
              engine:
                type: string
                enum:
                - postgres
                - mysql
                - mongodb
              storageSize:
                type: string
                pattern: '^[0-9]+Gi$'
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
              backup:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: false
                  schedule:
                    type: string
                    pattern: '^(\*|[0-9]{1,2}|\*/[0-9]+) (\*|[0-9]{1,2}|\*/[0-9]+) (\*|[0-9]{1,2}|\*/[0-9]+) (\*|[0-9]{1,2}|\*/[0-9]+) (\*|[0-9]|\*/[0-9]+)$'
EOF
```

## Step 5: Test required field validation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: missing-fields
spec:
  replicas: 3
EOF
```

**Error:**
```
The Database "missing-fields" is invalid: 
spec.engine: Required value
spec.storageSize: Required value
```

## Step 6: Test enum validation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: invalid-engine
spec:
  engine: "oracle"
  storageSize: "10Gi"
EOF
```

**Error:**
```
spec.engine: Unsupported value: "oracle": supported values: "postgres", "mysql", "mongodb"
```

## Step 7: Test pattern validation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: invalid-size
spec:
  engine: "postgres"
  storageSize: "10GB"
EOF
```

**Error:**
```
spec.storageSize: Invalid value: "10GB": spec.storageSize in body should match '^[0-9]+Gi$'
```

## Step 8: Test integer range validation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: too-many-replicas
spec:
  engine: "postgres"
  storageSize: "10Gi"
  replicas: 20
EOF
```

**Error:**
```
spec.replicas: Invalid value: 20: spec.replicas in body should be less than or equal to 10
```

## Step 9: Create valid database

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: valid-db
spec:
  engine: "postgres"
  storageSize: "50Gi"
  replicas: 3
  backup:
    enabled: true
    schedule: "0 2 * * *"
EOF
```

**Success!**

```bash
kubectl get database valid-db -o yaml
```

## Step 10: Test default values

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: defaults-test
spec:
  engine: "mysql"
  storageSize: "20Gi"
  # replicas omitted - should default to 1
  # backup.enabled omitted - should default to false
EOF
```

**Check defaults applied:**
```bash
kubectl get database defaults-test -o jsonpath='{.spec.replicas}'
echo ""
kubectl get database defaults-test -o jsonpath='{.spec.backup.enabled}'
echo ""
```

## Step 11: Test nested validation

```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: invalid-cron
spec:
  engine: "postgres"
  storageSize: "10Gi"
  backup:
    enabled: true
    schedule: "not-a-cron-expression"
EOF
```

**Error:**
```
spec.backup.schedule: Invalid value: "not-a-cron-expression": spec.backup.schedule in body should match '^(\*|[0-9]{1,2}|\*/[0-9]+)...'
```

## Step 12: Add status subresource

```bash
kubectl delete crd databases.example.com

cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - engine
            - storageSize
            properties:
              engine:
                type: string
                enum: ["postgres", "mysql", "mongodb"]
              storageSize:
                type: string
                pattern: '^[0-9]+Gi$'
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Creating", "Ready", "Failed"]
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                    lastTransitionTime:
                      type: string
                      format: date-time
EOF
```

## Step 13: Update status (separate from spec)

```bash
# Create database
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: status-test
spec:
  engine: "postgres"
  storageSize: "10Gi"
EOF

# Try to update status in spec (fails)
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: status-test
spec:
  engine: "postgres"
  storageSize: "10Gi"
status:
  phase: "Ready"
EOF
```

Status is ignored when updating spec!

**Update status separately:**
```bash
kubectl patch database status-test --subresource=status --type=merge -p '{"status":{"phase":"Ready"}}'
```

## Step 14: Test additional properties restriction

```bash
kubectl delete crd databases.example.com

cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            additionalProperties: false  # Strict!
            required: ["engine", "storageSize"]
            properties:
              engine:
                type: string
              storageSize:
                type: string
EOF
```

**Try unknown field:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Database
metadata:
  name: unknown-field
spec:
  engine: "postgres"
  storageSize: "10Gi"
  typo: "this-field-doesnt-exist"
EOF
```

**Error:**
```
spec: Additional property typo is not allowed
```

## Step 15: Test multiple versions

```bash
kubectl delete crd databases.example.com

cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              type:
                type: string
              size:
                type: string
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["engine", "storageSize"]
            properties:
              engine:
                type: string
              storageSize:
                type: string
EOF
```

**Create with v1alpha1:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1alpha1
kind: Database
metadata:
  name: alpha-version
spec:
  type: "postgres"
  size: "10Gi"
EOF
```

**Check stored version:**
```bash
kubectl get database alpha-version -o yaml | grep apiVersion
```

Shows v1 (storage version)!

## Key Observations

✅ **OpenAPI v3 schema** - defines validation rules
✅ **Required fields** - enforced at admission time
✅ **Enum validation** - restricts allowed values
✅ **Pattern matching** - regex validation for strings
✅ **Default values** - automatically applied
✅ **Status subresource** - separates spec from status
✅ **additionalProperties: false** - strict field validation

## Production Patterns

**Database CRD with comprehensive validation:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
    singular: database
    shortNames: [db]
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
      scale:
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.replicas
    schema:
      openAPIV3Schema:
        type: object
        required: [spec]
        properties:
          spec:
            type: object
            required: [engine, storageSize]
            properties:
              engine:
                type: string
                enum: [postgres, mysql, mongodb, redis]
              version:
                type: string
                pattern: '^[0-9]+\.[0-9]+(\.[0-9]+)?$'
              storageSize:
                type: string
                pattern: '^[0-9]+Gi$'
              storageClass:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
              resources:
                type: object
                properties:
                  cpu:
                    type: string
                    pattern: '^[0-9]+m?$'
                  memory:
                    type: string
                    pattern: '^[0-9]+[MGT]i$'
          status:
            type: object
            properties:
              phase:
                type: string
                enum: [Pending, Creating, Ready, Failed, Deleting]
              replicas:
                type: integer
              readyReplicas:
                type: integer
```

**Application CRD example:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.app.example.com
spec:
  group: app.example.com
  names:
    kind: Application
    plural: applications
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [image, replicas]
            properties:
              image:
                type: string
                pattern: '^[a-z0-9\-\.\/]+:[a-z0-9\.\-]+$'
              replicas:
                type: integer
                minimum: 0
                maximum: 100
              environment:
                type: string
                enum: [development, staging, production]
                default: development
              ingress:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: false
                  hostname:
                    type: string
                    pattern: '^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$'
```

## Cleanup

```bash
kubectl delete database --all
kubectl delete crd databases.example.com
```

---
Next: Day 30 - Operator Reconciliation Loop Failures