# Pinning Operator Versions

Pin an operator to a specific version by setting `installPlanApproval: Manual` and `startingCSV` on the OLM Subscription. The `helper-status-checker` sub-chart handles approving the InstallPlan.

## ArgoCD Health Check

A custom Lua health check on the ArgoCD CR prevents Manual subscriptions from being stuck as "Progressing". This is already configured at:

```
cluster/infra/argocd-Subscription-health-check/argo-healthchecks.yaml
```

This must be applied before deploying any operator with Manual approval.

## Working Example: KubeVirt Operator

### Chart values (`workloads_library/infra/kubevirt-operator/values.yaml`)

```yaml
operator:
  channel: stable
  startingCSV: kubevirt-hyperconverged-operator.v4.21.0
  installPlanApproval: Manual

helper-status-checker:
  enabled: true
  approver: false
  checks:
    - operatorName: kubevirt-hyperconverged
      namespace:
        name: openshift-cnv
      syncwave: "-5"
      serviceAccount:
        name: "kubevirt-status-checker"
```

### Chart dependencies (`workloads_library/infra/kubevirt-operator/Chart.yaml`)

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

### Application template (`cluster/infra/bootstrap/templates/workloads_library/application-kubevirt-operator.yaml`)

The Application overrides chart defaults with inline Helm values. `startingCSV` here is the source of truth for the pinned version:

```yaml
helm:
  values: |
    operator:
      startingCSV: kubevirt-hyperconverged-operator.v4.21.0
      installPlanApproval: Manual
    helper-status-checker:
      enabled: true
      approver: true
      checks:
        - operatorName: kubevirt-hyperconverged
          namespace:
            name: openshift-cnv
          syncwave: '-5'
          serviceAccount:
            name: "kubevirt-status-checker"
```

### Updating the pinned version

Update `startingCSV` in the Application template, commit, and push. ArgoCD syncs the new Subscription and `helper-status-checker` approves the InstallPlan.
