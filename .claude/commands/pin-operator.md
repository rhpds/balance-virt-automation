Add operator version pinning to the operator named: $ARGUMENTS

## What this does

Configures an operator workload in the workloads library to use OLM version pinning via `installPlanApproval: Manual`, `startingCSV`, and the `helper-status-checker` sub-chart.

## Steps

### 1. Gather information

Ask the user for:
- **Operator name** (the OLM package name, e.g. `kubevirt-hyperconverged`)
- **startingCSV** (the exact CSV version string, e.g. `kubevirt-hyperconverged-operator.v4.21.0`)
- **Operator namespace** (e.g. `openshift-cnv`)
- **OLM channel** (e.g. `stable`)

Identify the operator's workloads library path. The infra chart should be at `workloads_library/infra/<operator-name>/` and the Application template at `cluster/infra/bootstrap/templates/workloads_library/application-<operator-name>.yaml`.

### 2. Verify ArgoCD health check is enabled

Check that `cluster/infra/bootstrap/values.yaml` has the ArgoCD resourceHealthChecks enabled:

```yaml
argocd:
  resourceHealthChecks:
    enabled: true
```

And that `cluster/infra/bootstrap/templates/argo-healthchecks.yaml` exists with the Lua Subscription health check that treats Manual install plans as Healthy. If either is missing, create/enable them.

### 3. Update the operator's Chart.yaml

Add `helper-status-checker` and `tpl` as dependencies if not already present:

```yaml
dependencies:
  - name: helper-status-checker
    version: ~4.0.0
    repository: https://charts.stderr.at/
    condition: helper-status-checker.enabled
  - name: tpl
    version: ~1.0.0
    repository: https://charts.stderr.at/
```

### 4. Update the operator's values.yaml

Set `installPlanApproval: Manual` in the operator values. The `startingCSV` is set in the Application template (step 6), not here:

```yaml
operator:
  channel: <channel>
  installPlanApproval: Manual
```

### 5. Update the operator's subscription.yaml template

Ensure the subscription template renders `installPlanApproval` and `startingCSV` from values. Use the kubevirt operator as a reference (`workloads_library/infra/kubevirt-operator/templates/subscription.yaml`):

```yaml
spec:
  channel: {{ .Values.operator.channel }}
  installPlanApproval: {{ .Values.operator.installPlanApproval }}
  name: <operator-name>
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  {{- if .Values.operator.startingCSV }}
  startingCSV: {{ .Values.operator.startingCSV }}
  {{- end }}
```

### 6. Update the ArgoCD Application template

In the Application template at `cluster/infra/bootstrap/templates/workloads_library/application-<name>.yaml`, add inline helm values for version pinning. Use `{{ .Values.<operatorKey>.operator.startingCSV }}` to pull the startingCSV from bootstrap values. Reference the descheduler as an example (`cluster/infra/bootstrap/templates/workloads_library/application-descheduler-operator.yaml`):

```yaml
helm:
  values: |
    operator:
      startingCSV: {{ .Values.<operatorKey>.operator.startingCSV }}
      installPlanApproval: Manual
    helper-status-checker:
      enabled: true
      approver: true
      checks:
        - operatorName: <operator-name>
          namespace:
            name: <operator-namespace>
          syncwave: '<sync-wave>'
          serviceAccount:
            name: "<operator-name>-status-checker"
```

### 7. Update the infra bootstrap values.yaml

In `cluster/infra/bootstrap/values.yaml`, add the `operator.startingCSV` value under the operator's values block:

```yaml
<operatorKey>:
  enabled: false
  git:
    path: workloads_library/infra/<operator-name>
    <<: *git_defaults
  operator:
    startingCSV: <startingCSV>
```

### 8. Verify

Run `helm template` on the operator chart to confirm the subscription renders correctly with the pinning values:

```
helm template workloads_library/infra/<operator-name>/ --set operator.startingCSV=<startingCSV> --set operator.installPlanApproval=Manual
```
