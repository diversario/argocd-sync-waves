1. Get a kind cluster up
   1. set `networking.apiServerAddress` to laptop IP
   2. make sure `kubectl` current context is this cluster
      1. ArgoCD container will have access to your entire `kubeconfig.yaml`, making it very easy to accidentally do stuff to a real cluster.
      Remove real clusters from your kubeconfig, or override the default kubeconfig location via the `KUBECONFIG` env var and export kind config into that file (see below in the diff)
2. Go through the [toolchain guide](https://argo-cd.readthedocs.io/en/stable/developer-guide/toolchain-guide) up to and including the [Generate API glue code and other assets section](https://argo-cd.readthedocs.io/en/stable/developer-guide/toolchain-guide/#generate-api-glue-code-and-other-assets)
   1. We need a newer `git` in the `test-tools-image` image.
      1. After the `ENV DEBIAN_FRONTEND=noninteractive` line, add:
      ```
      RUN  apt-get update && apt-get install --fix-missing -y \
          software-properties-common

      RUN add-apt-repository ppa:git-core/ppa -y
      ```
3. Go through the [developer guide](https://argo-cd.readthedocs.io/en/stable/developer-guide/running-locally/#start-local-services-virtualized-toolchain-inside-docker) up to and including the [Start local services (virtualized toolchain inside Docker) section](https://argo-cd.readthedocs.io/en/stable/developer-guide/running-locally/#start-local-services-virtualized-toolchain-inside-docker)
   1. Apply these changes to the `Makefile`:
    ```diff
    diff --git a/Makefile b/Makefile
    index 96275f9bf..adeaff130 100644
    --- a/Makefile
    +++ b/Makefile
    @@ -100,11 +100,14 @@ define run-in-test-server
                    -e ARGOCD_GPG_DATA_PATH=${ARGOCD_GPG_DATA_PATH:-/tmp/argocd-local/gpg/source} \
                    -e ARGOCD_APPLICATION_NAMESPACES \
                    -e GITHUB_TOKEN \
    +               -e KUBECONFIG=/home/user/.kube/kind-kubeconfig.yaml \
    +               -e ARGOCD_ASK_PASS_SOCK=/home/user/reposerver-ask-pass.sock \
                    -v ${DOCKER_SRC_MOUNT} \
                    -v ${GOPATH}/pkg/mod:/go/pkg/mod${VOLUME_MOUNT} \
                    -v ${GOCACHE}:/tmp/go-build-cache${VOLUME_MOUNT} \
                    -v ${HOME}/.kube:/home/user/.kube${VOLUME_MOUNT} \
    -               -v /tmp:/tmp${VOLUME_MOUNT} \
    +               -v ${HOME}/tmp:/tmp${VOLUME_MOUNT} \
                    -w ${DOCKER_WORKDIR} \
                    -p ${ARGOCD_E2E_APISERVER_PORT}:8080 \
                    -p 4000:4000 \
    @@ -125,6 +128,8 @@ define run-in-test-client
                    -e GITHUB_TOKEN \
                    -e GOCACHE=/tmp/go-build-cache \
                    -e ARGOCD_LINT_GOGC=$(ARGOCD_LINT_GOGC) \
    +               -e KUBECONFIG=/home/user/.kube/kind-kubeconfig.yaml \
    +               -e ARGOCD_API_URL=http://10.7.0.197:8080 \ # this needs to match your laptop IP
                    -v ${DOCKER_SRC_MOUNT} \
                    -v ${GOPATH}/pkg/mod:/go/pkg/mod${VOLUME_MOUNT} \
                    -v ${GOCACHE}:/tmp/go-build-cache${VOLUME_MOUNT} \
    @@ -482,7 +487,7 @@ clean: clean-debug
    .PHONY: start
    start: test-tools-image
            $(DOCKER) version
    -       $(call run-in-test-server,make ARGOCD_PROCFILE=test/container/Procfile start-local ARGOCD_START=${ARGOCD_START})
    +       $(call run-in-test-server,git config --global --add safe.directory '*' && make ARGOCD_PROCFILE=test/container/Procfile start-local ARGOCD_START=${ARGOCD_START})

    # Starts a local instance of ArgoCD
    .PHONY: start-local
    ```
4. Run `make start`. If it fails with
   ```
   fatal: detected dubious ownership in repository at '/go/src/github.com/argoproj/argo-cd'
   ```
   then change the `git config` invocation in `start` target from `*` to that specific directory and rerun.
5. Go to the UI at http://localhost:4000/settings/repos and add a public GH repo (public so no auth config required)
6. Go to http://localhost:4000/applications and add a new app that uses the repo from the previous step. You will likely see this in the repo server logs:
    ```
    09:44:04               repo-server | ERRO[0218] finished unary call with code Unknown         error="failed to initialize repository resources: rpc error: code = Internal desc = Failed to fetch default: `git fetch origin --tags --force --prune` failed exit status 128: fatal: detected dubious ownership in repository at '<path to cached source> add an exception for this directory, call:\n\n\tgit config --global --add safe.directory <path to cached source>" grpc.code=Unknown grpc.method=GenerateManifest grpc.service=repository.RepoServerService grpc.start_time="2024-09-11T09:44:04Z" grpc.time_ms=245.089 span.kind=server system=grpc
    ```
    Exec into the container and run
    ```sh
    chown -R user /tmp/_argocd-repo/d0135c11-65b9-4d3e-ad3f-0739eddd509c # use the correct path
    ```
    Stop and start the container again.
