# OKE-Debug-Pod-Node-Doctor-Runbook

This runbook shows how to:

1. Create a debug pod on an OKE worker node
2. Enter the host namespace
3. Run `node-doctor.sh`
4. Copy the resulting bundle or log to your local machine or OCI Object Storage
5. Clean up the debug pod

## Prerequisites

* `kubectl` configured for the target cluster
* Access to the OKE worker node
* A valid debug image, for example:
  `docker.io/library/ubuntu:latest`

## 1. Create a debug pod 

Use the node IP or node name. In this example, 10.0.10.42 is being used as an example private IP for the node. Default setting is to expire the pod after 30 minutes. 

```bash
kubectl debug node/10.0.10.42 \
  --profile=general \
  --image=docker.io/library/ubuntu:latest \
  -- /bin/sh -c 'trap : TERM INT; sleep 1800 & wait'
```

## 2. Attach to the debug pod

First find the pod name, then, exec into it.

```bash
kubectl get pods | grep node-debugger-10.0.10.42
kubectl exec -it node-debugger-10.0.10.42-xxxxx -- /bin/sh
```

## 3. Enter the host filesystem

Inside the debug container:

```sh
cd /host
chroot /host
pwd
ls /
```

You are now operating on the worker node host.

## 4. OPTIONAL: Fix the Python venv prerequisite if needed

If `node-doctor.sh` fails with `ensurepip is not available` or `No module named pip`, install the matching venv package on the host.

```sh
python3 --version
apt update
apt install -y python3.10-venv
rm -rf /venv
```

If the node uses a different Python version, use the matching package instead:

```sh
apt install -y python3-venv
# or python3.11-venv / python3.12-venv, depending on the node image
```

## 5. Run node doctor

Run the check script from the host shell first to check if node doctor is working.

```sh
/usr/local/bin/node-doctor.sh --check
```

Run the bundle generation script from the host shell first to generate the log bundle.

```sh
/usr/local/bin/node-doctor.sh --generate
```

The bundle output is usually saved under:

```text
/tmp/oke-support-bundle-<date-time>
```

If the bundle path is not obvious, locate it:

```sh
find / -maxdepth 4 \
  \( -iname '*oke-node-doctor*' -o -iname '*.tar' -o -iname '*.tar.gz' \) \
  2>/dev/null
```

## 6. Copy the bundle or log to your local machine

First switch to a new terminal on your local machine. We will confirm where the file exists. The output may be in the pod filesystem or on the host filesystem mounted at `/host`.

```bash
kubectl exec -it node-debugger-10.0.10.42-xxxxx -- sh -c 'ls -l /tmp /var/log/oke-node-doctor /host/tmp /host/var/log/oke-node-doctor'
```

### Copy the file to your local filesystem

```bash
kubectl cp node-debugger-10.0.10.42-xxxxx:/host/tmp/oke-support-bundle-date-time.tar ~/path/to/folder/oke-support-bundle-date-time.tar
```

## 7. Upload the bundle to OCI Object Storage

You can upload directly from the debug pod if OCI CLI is installed there and the pod has valid credentials
If the bundle is on the host filesystem through the mount:

```
kubectl exec -it node-debugger-10.0.10.42-xxxxx -- sh -c '
  oci os object put \
    -bn <bucket-name> \
    --name oke-support-bundle-<date-time>.tar \
    --file /host/tmp/oke-support-bundle-<date-time>.tar
```

Fallback: If the debug pod does not have OCI CLI or credentials, copy the file locally first and then upload it from your machine.

## 8. Clean up

When finished, exit the host shell and delete the debug pod.

```sh
exit
```

```bash
kubectl delete pod node-debugger-10.0.10.42-xxxxx
```

## Notes

* If you use a fully interactive `kubectl debug -it ... -- /bin/sh` session instead, it only stays alive while that shell remains open.
* If `kubectl debug` appears to hang after printing `Creating debugging pod...`, that usually means the pod was created successfully and the command is waiting because the container is intentionally kept running.
