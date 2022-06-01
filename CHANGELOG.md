# Changelog

## ESP 7.22 / Viya 2021.2.4

* Support Kubernetes version 1.22.

## ESP 7.1

* an opensource postgres DB is used for client storage needs: Studio, Streamviewer, ESM and ESP Metering. A persitent volume is required for the storage needs of the postgres DB.
* the filebrowser and individual ESP projects require only a persistent volume if testing with csv files.
* each service and project no longer get a unique host via ingress for access. A single host of the form: `<tenant>.<domain>` is used to access all `<tenant>` services.

path based ingress:

```text
  Project X    --   <namespace>.sas.com/SASEventStreamProcessing/X
  Metering     --   <namespace>.sas.com/SASEventStreamProcessingMetering
  Studio       --   <namespace>.sas.com/SASEventStreamProcessingStudio
  Streamviewer --   <namespace>.sas.com/SASEventStreamProcessingStreamviewer
  ESM          --   <namespace>.sas.com/SASEventStreamManager
  FileBrowser  --   <namespace>.sas.com/files
```

* Multi user mode implemented with a external UAA server. The multiuser deployment comes with TLS enabled by default.
* Simple to use *uaatool* included for user management.

* Testing has been done with the following thord party components:

```text
ghcr.io/skolodzieski/postgres             12.5
ghcr.io/skolodzieski/uaa                  74.29.0
ghcr.io/skolodzieski/uaac                 3.2.0
ghcr.io/skolodzieski/busybox              1.33.0
ghcr.io/skolodzieski/filebrowser          v2.11.0
```

## ESP 6.2

ESP 6.2 was the initial release.

* the deployment files for version 6.2.x of ESP may be found under the v6.2.2 TAG.
