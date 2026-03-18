# OLM v1 Adoption FAQ

This FAQ is compiled from discussions in the #olm-v1-adoption-for-your-operator Slack channel.

## Table of Contents

- [Getting Started & Setup](#getting-started--setup)
- [Install Modes & Namespace Configuration](#install-modes--namespace-configuration)
- [Configuration & Customization](#configuration--customization)
- [Migration from OLMv0](#migration-from-olmv0)
- [Webhooks & Security](#webhooks--security)
- [Operator Behavior Changes](#operator-behavior-changes)
- [Known Limitations](#known-limitations)
- [Testing & Verification](#testing--verification)
- [Timeline & Support](#timeline--support)

---

## Getting Started & Setup

### How do I enable OLMv1 on my cluster?

OLMv1 has had feature-driven GA levels.  Compare your needed feature against the following matrix.

| Feature                               | OCP rel# | Availability |
|---------------------------------------|----------|--------------|
| ClusterExtension API                  | 4.18     | G            |
| ClusterCatalog API                    | 4.18     | G            |
| ClusterExtension Webhooks             | 4.21     | G            |
| ClusterExtension Single/Own-Namespace | 4.21     | T            |

'G' indicates this feature defaults to enabled and is considered Generally Available.
'T' indicates that this feature defaults to disabled and will be enabled in Tech Preview No Upgrades configuration.

### How do I create an OLM v1 package from scratch?

**Bundle format**: OLM v1 uses the **same bundle format** as OLM v0, so you can reuse existing bundles. The bundle image structure is identical.

**Catalog format**: OLM v1 requires **File-Based Catalogs (FBC)**, not sqlite-based catalogs. The catalog image contains FBC which lists the operator bundles available.

**Getting started guide**: https://operator-framework.github.io/operator-controller/getting-started/olmv1_getting_started/#install-a-cluster-extension

**Creating a catalog**: https://operator-framework.github.io/operator-controller/getting-started/olmv1_getting_started/#create-a-catalog

**Key differences from OLM v0**:
- You deploy using `ClusterExtension` instead of `Subscription`
- You must create a `ServiceAccount` with appropriate RBAC permissions before installation
- The catalog must use FBC format (not sqlite-based catalogs)


### Why am I getting "no bundles found for package" errors?

Common causes:
1. **Wrong catalog version**: OLMv1 in 4.22 points to `registry.redhat.io/redhat/redhat-operator-index:v4.21` and the desired bundle version exists in the 4.22 catalog
2. **Operator not in catalog**: If your operator isn't GA for the catalog version, create a custom ClusterCatalog pointing to the correct version
3. **Version/channel mismatch**: Verify the version and channel exist in the catalog (or remove those filters from your ClusterExtension)

Example custom ClusterCatalog:
```yaml
apiVersion: olm.operatorframework.io/v1
kind: ClusterCatalog
metadata:
  name: my-custom-catalog
spec:
  source:
    type: Image
    image:
      ref: registry.redhat.io/redhat/redhat-operator-index:v4.21
      pollIntervalMinutes: 10
```

### How do I add custom registry certificates to OLMv1?

Add certificates to the cluster's trust store via:
- `proxy/cluster` resource
- `image.config.openshift.io/cluster` resource

Note: Some users reported this didn't work in early versions. Verify your OCP version supports this.

### Why does OLMv1 select the wrong channel (e.g., dev-preview instead of stable)?

OLMv1 does not support the `operators.operatorframework.io.bundle.channel.default.v1` annotation. If you require a specific channel to be used, always explicitly specify the channel in your ClusterExtension:

```yaml
spec:
  source:
    catalog:
      channels:
        - stable
```

---

## Install Modes & Namespace Configuration

### Does my operator need to support AllNamespaces install mode?

**Phase 2 (OCP 4.21)**: SingleNamespace and OwnNamespace are supported in **Tech Preview (TPNU)** via the `config.inline.watchNamespace` field in a limited way.

**Future**: AllNamespaces is the recommended design pattern, but not mandatory yet.

## Background

**Note**: Single-/Own-namespace  installmode support is currently **Tech Preview (TPNU)** and will remain so for the foreseeable future.  Design discussions are ongoing about the best approach for hangling operators that rely on non-AllNamespaces installmode.

In OLMv0, use-cases for the Single-/Own-namespace installmodes were to facilitate multiple operator installations (potentially with different versions).
OLMv1 is not going to support these use-cases because APIs/CRDs are cluster-scoped and OLMv1 enforces a single-owner rule on resources. See [OLMv1 Design Decisions](https://operator-framework.github.io/operator-controller/project/olmv1_design_decisions/) and [OLMv1 Single Owner Objects](https://operator-framework.github.io/operator-controller/concepts/single-owner-objects/).

OLMv1 has _limited support_ for Single-/Own-namespace installmodes using ClusterExtension configuration. If your ClusterExtension
- accepts being installed once on the cluster, 
- doesn't expect that API visibility is restricted to `.spec.namespace` or `.spec.config.inline.watchNamespace` namespaces, 
- don't rely on being able to watch the designated namespace for CRs, and 
- isn't reliant on OLMv0's role aggregation mechanisms
then your operator using SingleNamespace or OwnNamespace InstallMode may be installed by OLMv1 without changes.

### What install modes can be used in OLMv1?

| Install Mode | OLMv0 | OLMv1 Phase 2 |
|--------------|-------|---------------|
| AllNamespaces | Supported | Supported |
| OwnNamespace | Supported | TechPreview (with config) |
| SingleNamespace | Supported | TechPreview (with config) |
| MultiNamespace | Supported | Not Supported |

### How do I configure SingleNamespace/OwnNamespace mode?

Use the inline config with `watchNamespace`:

```yaml
apiVersion: olm.operatorframework.io/v1
kind: ClusterExtension
metadata:
  name: my-operator
spec:
  namespace: my-namespace
  serviceAccount:
    name: my-operator-installer
  config:
    configType: Inline
    inline:
      watchNamespace: my-namespace
  source:
    sourceType: Catalog
    catalog:
      packageName: my-operator
```

### Can I install the same operator in multiple namespaces?

**No**. ClusterExtension is cluster-scoped. You cannot install multiple instances of the same operator in different namespaces, even with OwnNamespace mode. This is a design change from OLMv0. Read more in [OLMv1 Design Decisions](https://operator-framework.github.io/operator-controller/project/olmv1_design_decisions/). 

### What if my operator requires an environment variable to run as a spoke cluster?

In OLMv0, you would patch the Subscription:
```bash
oc patch sub/my-subscription --patch '{"spec":{"config":{"env":[{"name": "KMM_MANAGED", "value": "1"}]}}}' --type=merge
```

**OLMv1 equivalent**: This feature is being discussed but is not available yet in Phase 2 trials. Check the current ClusterExtension CRD schema.

---

## Configuration & Customization

### What's the equivalent of Subscription's spec.config.env in OLMv1?

As of the discussions, there is **no direct equivalent yet**. The `config.inline` field supports `watchNamespace` but not arbitrary environment variables. This is a known gap being addressed.

### How do I configure operator-specific settings?

Currently limited to:
- `watchNamespace` via `config.inline.watchNamespace`
- Service account via `spec.serviceAccount.name`

For other configurations, you may need to use operator-specific CRs after installation or wait for additional config support.

### What ServiceAccount and RBAC permissions do I need to create before installing?

OLM v1 requires you to create a `ServiceAccount` with appropriate RBAC permissions **before** installing a ClusterExtension. This ServiceAccount is used by OLM to install and manage the operator.

**The ServiceAccount is only for installation**. OLM will still create all the other RBACs that are part of the bundle for the operator's runtime permissions.

**Deriving minimal RBAC**: https://github.com/operator-framework/operator-controller/blob/main/docs/howto/derive-service-account.md

**Prototype tools to generate minimal RBAC**: https://github.com/operator-framework/operator-controller/tree/main/hack/tools/catalogs

The installer ServiceAccount needs permissions to:
- Create and manage the extension's CustomResourceDefinitions
- Create and manage resources packaged in the bundle
- Grant the extension controller's service account the permissions it requires
- Create and manage the extension controller's service account
- Create and manage Roles, RoleBindings, ClusterRoles, and ClusterRoleBindings
- Create and manage the extension controller's deployment
- Update finalizers on the ClusterExtension (for clusters using OwnerReferencesPermissionEnforcement)

---

## Migration from OLMv0

### Are bundles and catalogs compatible between OLMv0 and OLMv1?

**Bundle images**: Yes, OLM v1 uses the **exact same bundle format** as OLM v0. You can reuse your existing bundle images without changes (assuming they don't use unsupported features like dependencies).

**Catalog images**: Yes, **if** they use File-Based Catalogs (FBC). OLM v1 requires FBC format and does not support sqlite-based catalogs.

- If your OLM v0 catalog is already using FBC, it will work with OLM v1
- If your OLM v0 catalog uses sqlite format, you must migrate to FBC for OLM v1

### How do I upgrade an operator from OLMv0 to OLMv1?

**Short answer**: Migration/upgrade path is still being designed.

Current understanding:
- OLMv0 and OLMv1 can coexist on the same cluster
- You cannot install the same operator via OLMv0 and via OLMv1 simultaneously
- Direct upgrade/migration tooling is not yet available

### Can I run OLMv0 and OLMv1 side-by-side?

Yes, with limitations.  OLMv1 enforces a [Single-Owner Model](https://operator-framework.github.io/operator-controller/concepts/single-owner-objects/) for all resources.  OLMv0 does not.  This means that OLMv1 will refuse to install an operator (containing a CRD) in a cluster where OLMv0 has already installed it.   The reverse is NOT true, i.e. OLMv0 will be happy to install a CSV for a ClusterExtension installed by OLMv1.  This can lead to unexpected outcomes and should be avoided.

### When will OLMv0 be deprecated?

Based on discussions:
- OLMv0 is in **maintenance mode** and viewed as feature complete
- Deprecation is expected to be announced in OCP 5.Y timeframe, but removal will not occur until OCP 6.0 (earliest)
- Operators should continue supporting OLMv0 for OCP 4.Y, OCP 5.Y

---

## Webhooks & Security

### Are webhooks supported in OLMv1 prior to OCP 4.21?

No, and the failure message looks like
```
message: unsupported bundle: webhookDefinitions are not supported
```


### Why are my privileged pods failing with pod security errors?

OLMv1 does **not automatically add pod security labels** to namespaces like OLMv0 does.

OLMv0 adds:
```yaml
metadata:
  annotations:
    security.openshift.io/scc.podSecurityLabelSync: "true"
    openshift.io/sa.scc.mcs: s0:c28,c17
    openshift.io/sa.scc.supplemental-groups: 1000790000/10000
    openshift.io/sa.scc.uid-range: 1000790000/10000
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

**Workaround**: Manually annotate the namespace:
```bash
oc annotate namespace my-namespace security.openshift.io/scc.podSecurityLabelSync=true
```

### Why am I getting webhook certificate errors?

Example error:
```
x509: certificate is valid for webhook-service.metallb-system.svc, webhook-service.metallb-system.svc.cluster.local,
not metallb-operator-webhook-server-service.metallb-system.svc
```

This may be related to how OLMv1 manages certificates. Check if your operator uses cert-manager or service-ca for webhook certificates.

---

## Operator Behavior Changes

### Does OLMv1 reconcile operator deployments like OLMv0?

**Yes**, but using different APIs:
- OLMv0 reconciles based on CSV
- OLMv1 reconciles based on the bundle spec in ClusterExtension

### Can I modify operator deployments at runtime?

**No**. OLMv1 reverts deployment changes back to the bundle specification.

One user reported an infinite loop:
1. Operator creates PVC
2. Operator modifies deployment to mount PVC
3. ClusterExtension reverts deployment to bundle spec
4. Loop repeats

**Implication**: Deployments should be considered **immutable** under OLMv1. Use operator CRs to configure behavior instead of modifying deployments.

### What happened to CSV manipulation for testing?

In OLMv0, QE teams could patch the CSV to:
- Set environment variables
- Scale deployments down
- Test different scenarios

**OLMv1**: The CSV doesn't exist in the same way. ClusterExtension is the primary resource, but it doesn't support the same level of manipulation.

**Alternative**: Modify the bundle and rebuild, or use operator-specific CRs for configuration.

### What happened to OperatorCondition CR?

OLMv1 does **not create** OperatorCondition CRs or set the `OPERATOR_CONDITION_NAME` environment variable.

**Use case affected**: Blocking upgrades via OperatorCondition

**Status**: Alternative mechanism under active development.

**Planned replacement**: A declarative approval API on an upcoming `ClusterExtensionRevision` API that would enable a set of approvals to be required before proceeding with the rollout of a revision. The operator would be able to register itself as an approver of future revisions.

**Note**: There is no design document yet, as prerequisite work is still ongoing. The primary use case from OLMv0 was the `Upgradeable` condition to block operator upgrades.

**Reference**: OLMv0 OperatorConditions documentation: https://github.com/operator-framework/olm-docs/blob/master/content/en/docs/advanced-tasks/communicating-operator-conditions-to-olm.md

### Why am I getting "cannot set blockOwnerDeletion" errors?

Example:
```
configmaps "kieconfigs-7.13.5" is forbidden: cannot set blockOwnerDeletion if an ownerReference
refers to a resource you can't set finalizers on
```

This is related to OLMv1 design decisions around ownership. The operator may not have permissions to set certain owner references.

**Reference**: https://operator-framework.github.io/operator-controller/project/olmv1_design_decisions/

### Does OLMv1 respect the console.openshift.io/disable-operand-delete annotation?

This is a console feature, not an OLM feature.  It's unclear if the OpenShift console will support this or an equivalent feature when interacting with OLMv1.

---

## Known Limitations

### What features are NOT supported in OLMv1 Phase 2?

1. **Dependency management**: OLMv0's dependency APIs are not supported
   - **Operators with dependencies will FAIL to install** if dependencies are declared in the bundle
   - You must **remove dependency declarations** from your bundle before using OLM v1
   - No automatic installation of required operators (e.g., cert-manager)
   - Users must manually install dependencies
   - If your operator uses dependencies, reach out in #olmv1-dev to schedule time with the OLM team and Operator Program to identify alternative solutions
   - Unsupported dependency properties:
     - `olm.gvk.required`
     - `olm.package.required`
     - `olm.constraint`

2. **Arbitrary environment variables**: No equivalent to Subscription's `spec.config.env` (yet)

3. **MultiNamespace install mode**: Not supported

4. **Multiple instances**: Cannot install the same CRD multiple times in different namespaces

5. **UI integration**: _Partially_ complete
   - ClusterExtensions may not show in "Installed Operators" UI
   - OperatorHub shows operators but installation flow differs

6. **Upgrade support**: Still being developed

7. **must-gather integration**: Some operators' must-gather may only look for OLMv0 APIs

### Can I use OLMv0 dependency features?

No. If your operator declares dependencies via OLMv0's dependency APIs, those will be ignored in OLMv1.

**Workaround**: Document dependencies and require manual installation.

### Is there a replacement for operatorframework.io/bundle-unpack-min-retry-interval?

**Status unclear**. May be obsolete in OLMv1 or handled differently.

---

## Testing & Verification

### What do I need to verify for my operator?

1. **Installation**: Operator installs successfully via ClusterExtension
2. **Functionality**: All operator features work as expected
3. **Operands**: Operator can create and manage its operands
4. **Upgrades**: If supported, test upgrade paths (Phase 2 may not support this yet)

### Are there Prow CI job examples for OLMv1?

**Status**: OLMv0 had https://docs.ci.openshift.org/docs/how-tos/testing-operator-sdk-operators/

OLMv1 examples are being developed. Check the openshift/release repo for updates.

### How do I test in CI?

Options:
1. Use clusterbot with TechPreview enabled
2. Create custom CI jobs based on OCP 4.22+
3. Wait for official OLMv1 CI job templates

### My operator passed Phase 2 testing. Does it support OLMv1?

**Probably, but verify**:
- Installation works
- Functionality works
- You've addressed any known limitations (dependencies, pod security, etc.)

Even if it works, you may need to document changes or limitations for users.

### Why isn't my operator visible in OperatorHub on 4.22?

1. **Catalog mismatch**: OCP 4.22 uses v4.21 catalog by default; your operator may only be in v4.21
2. **Not GA yet**: Operator hasn't been released for 4.22
3. **Create custom ClusterCatalog**: Point to the catalog index image containing your operator

---

## Timeline & Support

### When is OLMv1 available?

- **OCP 4.18-4.22**: Basic feature support initially, with incremental feature addition (see matrix at top).
- **Future**: TBD when it becomes GA

### When is OLMv0 deprecated?

- **Currently**: Maintenance mode
- **Expected deprecation**: Not until OCP5.Y
- **Expected removal**: Not until OCP6.0
- **Recommendation**: Support both OLMv0 (for 4.22 and earlier) and OLMv1 (for 4.18+)

### What's required for operators in OCP 4.22?

- **Must work on OLMv0**: For customers not using TechPreview
- **Should be verified on OLMv1**: To ensure future compatibility
- **Document limitations**: If OLMv1 has different behavior/features

### Do I need to migrate to OLMv1 immediately?

**No**. OLMv0 will be supported through at least 2027. However:
- Start testing with OLMv1 now
- Plan for eventual migration
- Consider AllNamespaces design pattern for new operator versions

### Is OLMv1 meant to be an option or a replacement in OCP 4.21?

**Option** for now. Operators should continue supporting OLMv0 as the primary installation method for 4.21.

### What if my operator isn't listed in the tracking `Phase 2` spreadsheet?

It may not be required for Phase 2 verification, but you should still test it if:
- It will be used on OCP 4.21+
- It uses webhooks
- It has complex namespace requirements
- It has dependencies

---

## Additional Resources

- **Verification Guide**: https://docs.google.com/document/d/1UpBLZFGONj-e9aDTvWFdlGKLnaIhTZQr-wOWcs6lGCI/edit
- **Design Decisions**: https://operator-framework.github.io/operator-controller/project/olmv1_design_decisions/
- **Limitations**: https://operator-framework.github.io/operator-controller/project/olmv1_limitations/
- **Tracking Spreadsheet**: https://docs.google.com/spreadsheets/d/1Nb5BUNyfMmgzBQdXL4MRJKr1MJBU6H8W3JOmzdSANi8/edit

---

## Common Issues & Solutions

### Issue: "supported install modes OwnNamespace SingleNamespace do not support targeting all namespaces"

**Solution**: Add the `watchNamespace` config:
```yaml
spec:
  config:
    configType: Inline
    inline:
      watchNamespace: your-namespace
```

### Issue: "CustomResourceDefinition already exists in namespace and cannot be managed by operator-controller"

**Cause**: Trying to install the same operator twice (CRDs are cluster-scoped)

**Solution**: Uninstall the previous instance or use a different operator

### Issue: "required field watchNamespace is missing"

**Solution**: Your bundle's registryv1 configuration requires watchNamespace. Add it to your ClusterExtension:
```yaml
spec:
  config:
    configType: Inline
    inline:
      watchNamespace: your-namespace
```

### Issue: Operator installed but pods not starting due to security constraints

**Solution**: Add pod security labels to the namespace (see [Webhooks & Security](#webhooks--security) section)

### Issue: "would violate PodSecurity restricted:latest: privileged containers"

**Solution**: Set namespace to privileged pod security level:
```bash
oc label namespace your-namespace pod-security.kubernetes.io/enforce=privileged
```

---

## Contributing to This FAQ

This FAQ is based on discussions from the #olm-v1-adoption-for-your-operator Slack channel. If you have updates or corrections, please contribute them back to the community by pushing a PR to this document.
