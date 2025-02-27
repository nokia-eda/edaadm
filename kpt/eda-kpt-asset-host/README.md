# assetvm

## Description
sample description

## Usage

### Fetch the package
`kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] assetvm`
Details: https://kpt.dev/reference/cli/pkg/get/

### View package content
`kpt pkg tree assetvm`
Details: https://kpt.dev/reference/cli/pkg/tree/

### Apply the package
```
kpt live init assetvm
kpt live apply assetvm --reconcile-timeout=2m --output=table
```
Details: https://kpt.dev/reference/cli/live/

## Testing

### Artifacts

Fetch the current artifacts, these are flat files being served by the web service

```shell
$ curl -k --resolve artifacts.assets.local:443:192.168.102.61 https://artifacts.assets.local:443/artifacts/
<!DOCTYPE html>
<html lang="en">
 <head>
  <meta charset="UTF-8" />
  <title>403 Forbidden</title>
 </head>
 <body>
  <h1>403 Forbidden</h1>
 </body>
</html>
```

Load a sample file to the artifacts

```shell
$ echo "`date` jagi" > /tmp/jagi.txt
$ cat /tmp/jagi.txt
Tue Feb  4 23:04:18 PST 2025 jagi

$ kubectl cp /tmp/jagi.txt -c lighttpd -n asset-components `kubectl get pods -n asset-components | grep lighttpd- | awk '{print $1}'`:/var/www/html

```

Fetch that sample file

```shell
$ curl -k --resolve artifacts.assets.local:443:192.168.102.61 https://artifacts.assets.local:443/artifacts/jagi.txt
Tue Feb  4 23:04:18 PST 2025 jagi

```

### Registry

Fetch the current images

```shell
$ curl -k --resolve registry.assets.local:443:192.168.102.61 https://registry.assets.local:443/v2/_catalog?n=1000
{"repositories":[]}

$ curl -k https://192.168.102.61:443/v2/_catalog?n=1000
{"repositories":[]}

```

Upload new images to this registry and repeat test to make sure the new images are listed

### Git

#### UI access

`http://192.168.102.61/git`

The default credentials is `eda` / `eda`

#### Clone

The default git repositories can be cloned as below

```
git clone http://192.168.102.61/git/eda/kpt.git
git clone http://192.168.102.61/git/eda/connect-k8s-helm-charts.git
git clone http://192.168.102.61/git/eda/playground.git
git clone http://192.168.102.61/git/eda/catalog.git
```
