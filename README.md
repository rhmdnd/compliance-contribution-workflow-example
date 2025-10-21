# Compliance Contribution Workflow Example

This repository shows how to write compliance rules, organize them into
profiles, and build them into a container image that the Compliance Operator
can parse and make available for scanning OpenShift clusters.

The goal is to enable various domain experts to write and maintain compliance
content in a systematic way.

## Repository Structure

The following project structure helps keep the compliance content organized:

* `rules/` - Contains any new `Rule` resources.
* `custom-rules/` - Contains any new `CustomRule` resources.
* `profiles/` - Contains `Profile` resources that use any default `Rule` resources or `Rule` resources created from `rules/`.
* `tailored-profiles/` - Contains `TailoredProfile` resources that use any `Rule` or `CustomRule` resources.
* `rbac/` - Contains a single file named `rbac.yaml` that includes any permission changes necessary for the `api-resource-collector` to fetch resources for the rules.
* `Containerfile` - A container file that copies relevant resources from each directory into a final container image that the Compliance Operator can use.

Note that the usage of `custom-rules` or `tailored-profiles` is not required.
They are simply there to give content authors the flexibility to create those
resources dynamically.

This structure allows content authors to write:

- Rules that support checking OpenShift add-on features, like OpenShift Virtualization
- Organize rules into one or more profiles
- Produce a consistent container image for distributing their compliance content

For example, this process can be used to write checks that are specific to
PCI-DSS guidance, and then group into a `profiles/pci-dss.yaml` file. It can
also be used to write rules specific to a deployment architecture, and
organized into a `rosa-pci-dss.yaml` profile.

## Workflow

Add rules to `rules/` and organize them into profiles in `profiles/`.

```bash
$ ls rules profiles 
profiles:
openshift-virtualization-benchmark.yaml

rules:
kubevirt-nonroot-feature-gate-is-enabled.yaml
```

Deploy changes for testing against a cluster with the Compliance Operator
already installed using Kustomize. This can be particularly useful for testing
and developing changes but it isn't required to ship compliance content:

```bash 
oc apply -k kustomization.yaml
```

Once you're ready to ship your content, bundle all the rules and profiles into
a container image:

```bash
podman build -t openshift-virtualization-benchmark:1.0.0 -f Containerfile .
```

Load the content into the Compliance Operator using a `ProfileBundle`

```yaml
---
apiVersion: compliance.openshift.io/v1alpha1
kind: ProfileBundle
metadata:
  name: openshift-virtualization-benchmark
  namespace: openshift-compliance
spec:
  contentImage: registry.example.io/compliance/openshift-virtualization-benchmark@1.0.0
```

Apply the `ProfileBundle` to the Compliance Operator namespace.

```bash
oc apply -f profile-bundle.yaml
```

This Compliance Operator will fetch the image, and apply all the manifests from
the various directories (`rules/`, `custom-rules/`, `profiles/`, `rbac/`,
`tailored-profiles/`). This will make your content available to scan the
cluster using a `ScanSettingBinding`:

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: openshift-virt-binding
  namespace: openshift-compliance
profiles:
  - apiGroup: compliance.openshift.io/v1alpha1
    kind: Profile
    name: openshift-virtualization-benchmark
settingsRef:
  apiGroup: compliance.openshift.io/v1alpha1
  kind: ScanSetting
  name: default
```

Apply that binding to start the scan:

```bash
oc apply -f scan-setting-binding.yaml
```

You can now monitor the status of the scan and the rules you created using the following command:

```bash
watch oc get scansettingbinding,suites,scans,compliancecheckresults
```
