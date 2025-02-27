# eda-kpt-asset-ingress

## Description
sample description

## Usage

### Fetch the package
`kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] eda-kpt-asset-ingress`
Details: https://kpt.dev/reference/cli/pkg/get/

### View package content
`kpt pkg tree eda-kpt-asset-ingress`
Details: https://kpt.dev/reference/cli/pkg/tree/

### Apply the package
```
kpt live init eda-kpt-asset-ingress
kpt live apply eda-kpt-asset-ingress --reconcile-timeout=2m --output=table
```
Details: https://kpt.dev/reference/cli/live/
