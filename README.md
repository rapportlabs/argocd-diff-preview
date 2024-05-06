![Argo CD Diff Preview](./images/title.png)

Argo CD Diff Preview is a tool that renders the diff between two branches in a Git repository. It is designed to render manifests generated by Argo CD, providing a clear and concise view of the changes between two branches. It operates similarly to Atlantis for Terraform, creating a plan that outlines the proposed changes.

### 3 Example pull requests:
- [Helm Example | Internal Chart](https://github.com/dag-andersen/argocd-diff-preview/pull/16)
- [Helm example | External Chart: Nginx](https://github.com/dag-andersen/argocd-diff-preview/pull/15)
- [Kustomize Example](https://github.com/dag-andersen/argocd-diff-preview/pull/12)

![](./images/example.png)

## Why do we need this?

In the Kubernetes world, we often use templating tools like Kustomize and Helm to generate our Kubernetes manifests. These tools make maintaining and streamlining configuration easier across applications and environments. However, they also make it harder to visualize the application's actual configuration in the cluster.

Mentally parsing Helm templates and kustomize patches is hard without rendering the actual output. Thus, making mistakes while modifying an application's configuration is relatively easy.

In the field of GitOps and infrastructure as code, all configurations are checked into Git and modified through PRs. The code changes in the PR are reviewed by a human, who needs to understand the changes made to the configuration. This is hard when the configuration is generated through templating tools like Kustomize and Helm.

## Overview

![](./images/flow.png)

The safest way to render your Helm Charts and Kustomize Overlays is to let ArgoCD render them for you. This can be done by spinning up an ephemeral cluster in your automated pipelines.

The implementation is actually quite simple. It just follows the steps below:

#### 10 steps
1. Start a local cluster
2. Install ArgoCD
3. Add the required credentials (git credentials, image pull secrets, etc.)
4. Fetch all ArgoCD application files on your PR branch
   - Point their `targetRevision` to the Pull Request branch.
   - Remove the `syncPolicy` from the application (to avoid the application to sync locally)
1. Apply the modified applications to the cluster
2. Let ArgoCD do its magic
3. Extract the rendered manifests from the ArgoCD server.
4. Repeat steps 4–7 for the base branch (main branch)
5. Create a diff between the manifests rendered from each branch.
6. Display the diff in the PR

## Features

- Renders manifests generated by Argo CD
- Spins up a fully ephemeral local cluster. No need to access the actual cluster where the applications are deployed.
- Can be run locally before you open the pull request
- Provides a clear and concise view of the changes
- Render resources from external sources (e.g., Helm charts). For example, when you update the chart version of Nginx, you can get a render of the new output. For example, this is useful to spot changes in default values. [PR example](https://github.com/dag-andersen/argocd-diff-preview/pull/15). 

### Not supported
- Does not support ArgoCD CMP plugins
- Does not work [Cluster Generators](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/) in your ApplicationSets

## Try demo locally with 3 simple commands!

Steps:
1. Make sure Docker is running. E.g., run `docker ps` to see if it's running.
2. Run the following 3 commands:

```bash
git clone https://github.com/dag-andersen/argocd-diff-preview base-branch --depth 1 -q 
git clone https://github.com/dag-andersen/argocd-diff-preview target-branch --depth 1 -q -b helm-example-2
docker run \
   --network host \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -v $(pwd)/output:/output \
   -v $(pwd)/base-branch:/base-branch \
   -v $(pwd)/target-branch:/target-branch \
   -e TARGET_BRANCH=helm-example-3 \
   -e GIT_REPO="https://github.com/dag-andersen/argocd-diff-preview.git" \
   dagandersen/argocd-diff-preview:latest
```

and the output would be something like this:

```bash
...
🚀 Creating cluster...
🚀 Cluster created successfully
🦑 Installing Argo CD...
...
🌚 Getting resources for base-branch
🌚 Getting resources for target-branch
...
🔮 Generating diff between main and helm-example-3
🙏 Please check the ./output/diff.md file for differences
```

and the `./output/diff.md` would be something like [this](https://github.com/dag-andersen/argocd-diff-preview/pull/16)


## Installation and Usage

### Run as container

Pre-requisites:
- Install: [Docker](https://docs.docker.com/get-docker/)

```bash
docker run \
   --network host \                                   # This is required so the container can access the local cluster on the host's docker daemon.
   -v /var/run/docker.sock:/var/run/docker.sock \     # This is required to access the host's docker daemon.
   -v <path-to-main-branch>:/base-branch \            # This is the main branch's ArgoCD applications.
   -v <path-to-pr-branch>:/target-branch \
   -v $(pwd)/output:/output \
   -e base_branch=main \
   -e target_branch=<name-of-the-target-branch> \
   dag-andersen/argocd-diff
```

<details>
  <summary>Example in GitHub Actions workflow</summary>

```yaml
- uses: actions/checkout@v4
  with:
    path: pull-request

- uses: actions/checkout@v4
  with:
    ref: main
    path: main

- name: Diff
  run: |
    docker run \
      --network=host \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v $(pwd)/main:/base-branch \
      -v $(pwd)/pull-request:/target-branch\
      -v $(pwd)/output:/output \
      -e TARGET_BRANCH=${{ github.head_ref }} \
      -e GIT_REPO=<your-repo> \
      dagandersen/argocd-diff-preview:latest

- name: post comment 
  run: |
    gh pr comment ${{ github.event.number }} --body-file output/diff.md
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

</details>

### Run as binary

Pre-requisites:
- Install: [Git](https://git-scm.com/downloads), [Docker](https://docs.docker.com/get-docker/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) OR [minikube](https://minikube.sigs.k8s.io/docs/start/), [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

Check the [releases](https://github.com/dag-andersen/argocd-diff-preview/releases) and find the correct binary for your operating system.

Example for downloading and running on macOS:

```bash
curl -LJO https://github.com/dag-andersen/argocd-diff-preview/releases/download/v0.0.4/argocd-diff-preview-Darwin-x86_64.tar.gz
tar -xvf argocd-diff-preview-Darwin-x86_64.tar.gz
./argocd-diff-preview --help
```

### Run from source

Pre-requisites:
- Install: [Git](https://git-scm.com/downloads), [Docker](https://docs.docker.com/get-docker/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) OR [minikube](https://minikube.sigs.k8s.io/docs/start/), [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/), [Rust](https://www.rust-lang.org/tools/install)

```bash
git clone https://github.com/dag-andersen/argocd-diff-preview
cd argocd-diff-preview
cargo run -- --help
```

### Handling credentials

In the simple code example above, I do not provide the cluster with any credentials, which only works if the image registry and the git repository are public. Since your Git repository might not be public you need to provide the tool with the necessary read-access credentials for the repository. This can be done by placing the Argo CD repo secrets in folder mounted at `/secrets`. When the tool starts, it will simply run `kubectl apply -f /secrets` to apply every resource to the cluster, before starting the rendering process.

<details>
  <summary>Credentials example - Username + Password </summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argoproj-https-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: https://github.com/argoproj
  type: helm
  password: my-password
  username: my-username
```
</details>

<details>
  <summary>Credentials example - GitHub App </summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: https://github.com/argoproj
  type: helm
  githubAppID: 1
  githubAppInstallationID: 2
  githubAppPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```
</details>

For more info, see the [Argo CD docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/argocd-repo-creds-yaml/)

### Scalability and performance

Rendering the manifests generated by all applications in the repository on each pull request is slow. This can be annoying, especially if only a few applications have changed. The tool supports grepping applications with regex. Setting the environment variable `FILE_REGEX` only allows the tool to run on manifests that match a particular regex.

For example, If someone in your organization from Team A changes to one of their applications, the tool can be run with `FILE_REGEX=/Team-A/` so it only renders changes in folders matching `*/Team-A/*`. This speeds up the process significantly.

## Options
```
USAGE:
    argocd-diff-preview [FLAGS] [OPTIONS] --git-repo <git-repository> --target-branch <target-branch>

FLAGS:
    -d, --debug      Activate debug mode
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -b, --base-branch <base-branch>         Base branch name [env: BASE_BRANCH=]  [default: main]
        --base-branch-folder <folder>       Base branch folder [env: BASE_BRANCH_FOLDER=]  [default: base-branch]
    -i, --diff-ignore <diff-ignore>         Ignore lines in diff. Example: use 'v[1,9]+.[1,9]+.[1,9]+' for ignoring changes caused by version changes following semver [env: DIFF_IGNORE=]
    -r, --file-regex <file-regex>           Regex to filter files. Example: "/apps_.*\.yaml" [env: FILE_REGEX=]
    -g, --git-repo <git-repository>         Git repository URL [env: GIT_REPO=]
        --local-cluster-tool <tool>         Local cluster tool. Options: kind, minikube [env: LOCAL_CLUSTER_TOOL=] [default: auto]
    -o, --output-folder <output-folder>     Output folder where the diff will be saved [env: OUTPUT_FOLDER=]  [default: ./output]
    -s, --secrets-folder <secrets-folder>   Secrets folder where the secrets are read from [env: SECRETS_FOLDER=]  [default: ./secrets]
    -t, --target-branch <target-branch>     Target branch name [env: TARGET_BRANCH=]
        --target-branch-folder <folder>     Target branch folder [env: TARGET_BRANCH_FOLDER=]  [default: target-branch]
        --timeout <timeout>                 Set timeout [env: TIMEOUT=]  [default: 180]
```

## Roadmap
- Make a dedicated GitHub Action that wraps the Docker container, so the tool becomes more user-friendly.  
- Let the user specify Argo CD version. Currently it always uses the newest version available.
- Delete Argo CD Applications, when they have been parsed by the tool, so Argo CD can focus on the remaining applications, which hopefully speeds up the process. 

## Do you experience any issues?

If you experience any issues, please open an issue in this repository. We will do our best to address it as soon as possible. I only have limited codebases available to me, so I can't test it on all possible codebases. 

## Questions, issues, or suggestions

If you have any questions, issues, or suggestions, please open an issue in this repository.
Any contributions are **greatly appreciated**. 
