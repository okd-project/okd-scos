# OKD/SCOS release procedure

## Validate
- Pick a nightly from https://origin-release.ci.openshift.org/#4.12.0-0.okd-scos
- Run the install with cluster-bot and ensure it's ready to be promoted; log in and check branding:
  ```
  launch registry.ci.openshift.org/origin/release-scos:<NIGHTLY>
  ```
- Export RELEASE env:
  ```
  export RELEASE="4.12.0-0.okd-scos-2022-10-22-232744"
  ```
- Check signatures:
  ```
  oc adm release info registry.ci.openshift.org/origin/release-scos:${RELEASE}
  ```
  Copy `Digest` from output
  ```
  ...
  Digest:    sha256:0132da68b7bc49ae27202585392ad1f6ee4951e0255627c15017775c6c9e33a6
  ...
  ```
  Check signature:
  ```
  export DIGEST=<DIGEST>
  curl -Lvs https://storage.googleapis.com/openshift-ci-release/releases/signatures/openshift/release/sha256\=${DIGEST}/signature-1 | gpg -d
  ```
  The output should look like:
  ```
  {
      "critical": {
      "type": "atomic container signature",
      "image": {
          "docker-manifest-digest": "sha256:874a0c414befe51f4e9040ebc139355122491eb76d15ccb964ccf3630e6ded1e"
      },
      "identity": {
          "docker-reference": "registry.ci.openshift.org/origin/release-scos:4.12.0-0.okd-scos-2022-10-22-232744"
      }
      },
      "optional": {
      "creator": "openshift release-controller",
      "timestamp": 1666481349
      }
  }
  ```

## Mirror
- Login to quay
  ```
  podman login quay.io
  ```
- Mirror the relase to quay:
  ```
  oc adm release new --from-release registry.ci.openshift.org/origin/release-scos:${RELEASE} --mirror quay.io/okd/scos-content --to-image quay.io/okd/scos-release:${RELEASE} --name=${RELEASE}
  ```
- Test the release via cluster-bot
  ```
  launch quay.io/okd/scos-release:${RELEASE}
  ```

## Update `4-scos-stable` channel
- Login to https://console-openshift-console.apps.ci.l2s4.p1.openshiftapps.com/dashboards
- Switch to origin project
  ```
  oc project origin
  ```
- Tag quay image to the imagetag
  ```
  oc tag quay.io/okd/scos-release:${RELEASE} release-scos:${RELEASE}
  ```

## Update GitHub releases
- Export tools
  ```
  oc adm release extract \
    --command-os='*' \
    --tools \
    --to=okd-scos-${RELEASE} \
    quay.io/okd/scos-release:${RELEASE}
  ```
- Sign binaries
  ```
  gpg --default-key maintainers@okd.io --armor --detach-sign okd-scos-${RELEASE}/sha256sum.txt
  ```
  Verify the signature
  ```
  gpg --verify okd-scos-${RELEASE}/sha256sum.txt.asc okd-scos-${RELEASE}/sha256sum.txt
  ```
- Create new release on https://github.com/okd-project/okd-scos/
  - Upload contents of the `okd-scos-${RELEASE}` directory
  - Add the following description:
    ````
    Client tools for OKD/SCOS
    -------------------------

    These archives contain the client tooling for [OKD on CentOS Stream CoreOS](https://docs.okd.io).

    To verify the contents of this directory, use the 'gpg' and 'shasum' tools to
    ensure the archives you have downloaded match those published from this location.

    The openshift-install binary has been preconfigured to install the following release:

    ```
    <Output of `oc adm release info --pullspecs quay.io/okd/scos-release:${RELEASE}`>
    ```
    ````

- For now, manually replace the line starting with `machine-os` under `Component Versions:`
  specifying the OS version in the output above which currently still points to the FCOS machine-os-content
  that is also present in the payload (although unused), with the `$PRETTY_NAME` of SCOS' `/etc/os-release`:
    
    ```
    $ podman run -it --rm --entrypoint bash $(oc adm release info --image-for=centos-stream-coreos-9 quay.io/okd/scos-release:${RELEASE})
    bash-5.1# source /etc/os-release 
    bash-5.1# echo $PRETTY_NAME
    CentOS Stream CoreOS 412.9.202210211543-0
    ```


## Notify community
- Include links to GitHub release and OKD/SCOS release status page
- Send email to okd-wg Google group
- Send message to `#openshift-users` on kubernetes.slack.com
