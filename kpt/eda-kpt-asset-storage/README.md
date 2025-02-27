# eda-kpt-asset-storage

## Description
sample description

## Usage

### Fetch the package
`kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] eda-kpt-asset-storage`
Details: https://kpt.dev/reference/cli/pkg/get/

### View package content
`kpt pkg tree eda-kpt-asset-storage`
Details: https://kpt.dev/reference/cli/pkg/tree/

### Apply the package
```
kpt live init eda-kpt-asset-storage
kpt live apply eda-kpt-asset-storage --reconcile-timeout=2m --output=table
```
Details: https://kpt.dev/reference/cli/live/
