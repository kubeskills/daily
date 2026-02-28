## Day 31: Helm Charts - Template Rendering Failures

## THE IDEA:
Install a Helm chart with syntax errors in templates and watch the installation 
fail with cryptic Go template errors. Learn to debug template rendering and 
value overrides.

## THE SETUP:
Create a chart with intentional template bugs, attempt installation, and decode 
errors like "nil pointer" and "unexpected <EOF>". Test value precedence.

## WHAT I LEARNED:
- Templates use Go text/template syntax
- helm template renders locally (dry run)
- Values file precedence: defaults < values.yaml < --set flags
- Missing required values cause nil pointer errors
- Syntax errors fail at render time, not install time

## WHY IT MATTERS:
Helm template issues cause:
- Failed deployments in CI/CD pipelines
- Production rollouts blocked by typos
- Configuration drift from incorrect value overrides

## TRY IT:
🧪 Interactive Lab: https://killercoda.com/chadmcrowell/course/kubeskills-daily/day-031

Tomorrow: GitOps sync failures with ArgoCD.

---


# Killercoda Lab Instructions


## Step 1: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## Step 2: Create basic Helm chart

```bash
helm create myapp
cd myapp

# Check generated structure
tree .
```

## Step 3: Examine default templates

```bash
cat templates/deployment.yaml | head -30
cat values.yaml
```

## Step 4: Test template rendering

```bash
helm template myapp . --debug
```

Shows rendered YAML output.

## Step 5: Introduce template syntax error

```bash
cat > templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  {{- if .Values.annotations }}  # Opening if
  annotations:
    {{- range $key, $val := .Values.annotations }}
    {{ $key }}: {{ $val }}
    {{- end }}
  # Missing {{- end }} for if!
data:
  app.conf: |
    server:
      port: {{ .Values.service.port }}
EOF
```

**Try rendering:**
```bash
helm template myapp . 2>&1 | grep -i error
```

**Error:**
```
Error: template: myapp/templates/configmap.yaml:11:1: unexpected EOF
```

## Step 6: Fix template syntax

```bash
cat > templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  {{- if .Values.annotations }}
  annotations:
    {{- range $key, $val := .Values.annotations }}
    {{ $key }}: {{ $val }}
    {{- end }}
  {{- end }}  # Fixed!
data:
  app.conf: |
    server:
      port: {{ .Values.service.port }}
EOF
```

**Test again:**
```bash
helm template myapp .
```

Success!

## Step 7: Test nil pointer dereference

```bash
cat > templates/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  # .Values.database doesn't exist!
  password: {{ .Values.database.password | b64enc }}
EOF
```

**Try rendering:**
```bash
helm template myapp . 2>&1
```

**Error:**
```
Error: template: myapp/templates/secret.yaml:7:21: executing "myapp/templates/secret.yaml" 
at <.Values.database.password>: nil pointer evaluating interface {}.password
```

## Step 8: Fix with default value

```bash
cat > templates/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
{{- if .Values.database }}
data:
  password: {{ .Values.database.password | default "changeme" | b64enc }}
{{- end }}
EOF
```

**Or add to values.yaml:**
```bash
cat >> values.yaml << 'EOF'

database:
  password: "defaultpass"
EOF
```

## Step 9: Test required values

```bash
cat > templates/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  password: {{ required "database.password is required!" .Values.database.password | b64enc }}
EOF
```

**Delete database from values.yaml:**
```bash
# Remove database section
sed -i '/^database:/,+1d' values.yaml
```

**Try rendering:**
```bash
helm template myapp . 2>&1 | grep -i required
```

**Error:**
```
Error: execution error at (myapp/templates/secret.yaml:7:16): database.password is required!
```

## Step 10: Test value precedence

```bash
# Add database back
cat >> values.yaml << 'EOF'
database:
  password: "from-values-file"
EOF
```

**Render with --set:**
```bash
helm template myapp . --set database.password="from-flag" | grep password
```

--set flag wins!

## Step 11: Test values file override

```bash
cat > override.yaml << 'EOF'
database:
  password: "from-override-file"
EOF

helm template myapp . -f override.yaml | grep password
```

Override file wins over values.yaml!

## Step 12: Test template functions

```bash
cat > templates/test-funcs.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-test
data:
  # String manipulation
  upper: {{ .Values.appName | upper | quote }}
  lower: {{ .Values.appName | lower | quote }}
  title: {{ .Values.appName | title | quote }}
  
  # Defaults
  missing: {{ .Values.missing | default "default-value" | quote }}
  
  # Math (wrong - using string)
  bad-math: {{ add .Values.replicas "5" }}
EOF
```

**Add to values.yaml:**
```bash
cat >> values.yaml << 'EOF'
appName: "MyApplication"
EOF
```

**Render:**
```bash
helm template myapp . 2>&1 | grep -A 20 test-funcs
```

**Error on bad-math:**
```
error calling add: unable to cast "5" of type string to int64
```

## Step 13: Fix function types

```bash
cat > templates/test-funcs.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-test
data:
  upper: {{ .Values.appName | upper | quote }}
  good-math: {{ add .Values.replicaCount 5 | quote }}
  list: {{ list "a" "b" "c" | join "," | quote }}
  dict: {{ dict "key1" "val1" "key2" "val2" | toJson }}
EOF
```

**Render:**
```bash
helm template myapp .
```

## Step 14: Test conditional blocks

```bash
cat > templates/conditional.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-conditional
data:
  {{- if eq .Values.environment "production" }}
  log-level: "error"
  {{- else if eq .Values.environment "staging" }}
  log-level: "warn"
  {{- else }}
  log-level: "debug"
  {{- end }}
  
  {{- with .Values.monitoring }}
  monitoring-enabled: "true"
  metrics-port: {{ .port | quote }}
  {{- end }}
EOF
```

**Add to values.yaml:**
```bash
cat >> values.yaml << 'EOF'
environment: "staging"
monitoring:
  port: 9090
EOF
```

**Render:**
```bash
helm template myapp .
```

## Step 15: Test range loops

```bash
cat > templates/loop.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-loop
data:
  {{- range .Values.environments }}
  env-{{ . }}: "enabled"
  {{- end }}
  
  {{- range $key, $val := .Values.config }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
EOF
```

**Add to values.yaml:**
```bash
cat >> values.yaml << 'EOF'
environments:
  - dev
  - staging
  - prod
  
config:
  timeout: "30s"
  retries: "3"
EOF
```

**Render:**
```bash
helm template myapp .
```

## Step 16: Test helm lint

```bash
# Lint the chart
helm lint .

# With values
helm lint . -f override.yaml
```

## Step 17: Dry-run installation

```bash
cd ..
helm install myapp-release ./myapp --dry-run --debug | head -50
```

Shows what would be installed.

## Step 18: Actual installation

```bash
helm install myapp-release ./myapp --set replicaCount=2
```

**Check resources:**
```bash
helm list
kubectl get all -l app.kubernetes.io/instance=myapp-release
```

## Step 19: Test upgrade

```bash
# Modify values
helm upgrade myapp-release ./myapp --set replicaCount=3

# Check revision history
helm history myapp-release
```

## Step 20: Test rollback

```bash
# Rollback to previous revision
helm rollback myapp-release 1

# Verify
helm history myapp-release
kubectl get deployment -l app.kubernetes.io/instance=myapp-release -o jsonpath='{.items[0].spec.replicas}'
```

## Key Observations

✅ **Go templates** - text/template syntax
✅ **helm template** - renders locally for debugging
✅ **Value precedence** - --set > -f override.yaml > values.yaml
✅ **required** - enforces mandatory values
✅ **Nil pointer** - accessing undefined values
✅ **Type errors** - function argument type mismatches

## Production Patterns

**Safe value access:**
```yaml
{{- if .Values.feature }}
{{- if .Values.feature.enabled }}
enabled: "true"
{{- end }}
{{- end }}

# Or with default
enabled: {{ .Values.feature.enabled | default false }}
```

**Required values with good errors:**
```yaml
{{- $_ := required "A valid database.host is required!" .Values.database.host }}
{{- $_ := required "A valid database.password is required!" .Values.database.password }}
```

**Named templates (helpers):**
```yaml
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
```

**Resource limits with defaults:**
```yaml
resources:
  {{- if .Values.resources }}
  {{- toYaml .Values.resources | nindent 2 }}
  {{- else }}
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
  {{- end }}
```

## Cleanup

```bash
helm uninstall myapp-release
cd ..
rm -rf myapp override.yaml
```

---
Next: Day 32 - GitOps Sync Failures with ArgoCD