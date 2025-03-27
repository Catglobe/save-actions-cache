# Why?

Tired of waiting for the 'Set up job' step to finish? This action caches the actions cache for self-hosted runners.

# How?

Action runners already have the cache mechanism built-in. This action simply puts the files of the current actions script in that folder to avoid downloading it in the future.

# Usage

1. Give the project a star! :star:
2. Setup your self-hosted runner to use the cache. You must set env variable `ACTIONS_RUNNER_ACTION_ARCHIVE_CACHE` for the runner to a folder that is writeable.

Example, we use `AutoscalingRunnerSet`:
```yaml
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  labels:
    app.kubernetes.io/component: "autoscaling-runner-set"
    actions.github.com/scale-set-name: cg-linux
    actions.github.com/scale-set-namespace: {{ .Release.Namespace }}
    {{- include "labels" . | nindent 4 }}
  name: gha-linux
  namespace: {{ .Release.Namespace }}
spec:
  githubConfigSecret: github-auth
  githubConfigUrl: 'https://github.com/Catglobe'
  maxRunners: 6
  minRunners: 6
  runnerGroup: default
  runnerScaleSetName: cg-linux
  template:
    spec:
      containers:
      - name: runner
        image: catglobe/gha-runner-linux:205
        env:
        - name: NUGET_PACKAGES
          value: "/opt/hostedtoolcache/nuget"
        - name: npm_config_cache
          value: "/opt/hostedtoolcache/npm"
        - name: RUNNER_TOOL_CACHE
          value: "/opt/hostedtoolcache/tool"
        - name: ACTIONS_RUNNER_ACTION_ARCHIVE_CACHE
          value: "/opt/hostedtoolcache/actions"
        - name: REPO_CACHE
          value: "/opt/hostedtoolcache/repo-cache"
        #- name: STARTUP_DELAY_IN_SECONDS
        #  value: "600"
        volumeMounts:
        - mountPath: /actions-runner/_work
          name: work
        - mountPath: /opt/hostedtoolcache
          name: cache
      initContainers:
      - name: mkdir
        image: alpine:3@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c
        command:
        - /bin/sh
        - -c
        - |
          mkdir -p /opt/hostedtoolcache/repo-cache && 
          mkdir -p /opt/hostedtoolcache/tool && 
          mkdir -p /opt/hostedtoolcache/nuget && 
          mkdir -p /opt/hostedtoolcache/npm && 
          mkdir -p /opt/hostedtoolcache/actions && 
          chmod 777 /opt/hostedtoolcache && 
          chmod 777 /opt/hostedtoolcache/repo-cache && 
          chmod 777 /opt/hostedtoolcache/tool && 
          chmod 777 /opt/hostedtoolcache/nuget && 
          chmod 777 /opt/hostedtoolcache/npm && 
          chmod 777 /opt/hostedtoolcache/actions
        volumeMounts:
        - mountPath: /opt/hostedtoolcache
          name: cache
        - mountPath: /actions-runner/_work
          name: work
      nodeSelector:
        kubernetes.io/os: linux
        gha-runner: "true"
      securityContext:
        fsGroup: 121
        fsGroupChangePolicy: "OnRootMismatch"
      volumes:
      - name: cache
        persistentVolumeClaim:
          claimName: github-cache-local-ssd
      - emptyDir: {}
        name: work
```
1. 3. If you care about security, fork this repository. That prevents any attack vector from being introduced in the future.
4. Add this to all of your actions workflows:
```yaml
jobs:
  MyFirstJob:
    runs-on: cg-linux # or whatever you named your runner
    ...
    steps:
    - name: Sync actions cache
      uses: catglobe/sync-actions-cache@v1
  OtherJob:
    runs-on: cg-linux # or whatever you named your runner
    ...
    steps:
    - name: Sync actions cache
      uses: catglobe/sync-actions-cache@v1
  PublicRunnerJob:
    #No runs-on: ==> don't add the step
```
Replace `catglobe` with your own company name or GitHub username if you made a fork.

If your `work` folder is not in `/actions-runner/_work`, you can set `actionsFolder` to your path (don't forget to append `/actions`):

```yaml
    - name: Sync actions cache
      uses: catglobe/sync-actions-cache@v1
      with:
        actionsFolder: /my_custom_work_folder/actions
```

5. Profit! `Set up job` should take single digit seconds, not minutes.

## How to check

Run your job in debug mode and check the logs. You should see something like this:
```
Download action repository 'actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683' (SHA:11bd71901bbe5b1630ceea73d27597364c9af683)
##[debug]Copied action archive '/opt/hostedtoolcache/actions/actions_checkout/11bd71901bbe5b1630ceea73d27597364c9af683.tar.gz' to '/actions-runner/_work/_actions/_temp_42fdacac-a721-4309-8fce-fb5f1ecef8d5/c30cfe66-109b-46ed-b245-25a6f6f36b8a.tar.gz'
##[debug]Unwrap 'actions-checkout-11bd719' to '/actions-runner/_work/_actions/actions/checkout/11bd71901bbe5b1630ceea73d27597364c9af683'
##[debug]Archive '/actions-runner/_work/_actions/_temp_42fdacac-a721-4309-8fce-fb5f1ecef8d5/c30cfe66-109b-46ed-b245-25a6f6f36b8a.tar.gz' has been unzipped into '/actions-runner/_work/_actions/actions/checkout/11bd71901bbe5b1630ceea73d27597364c9af683'.
```

# License

MIT