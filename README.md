# CI-CD_github_actions — Learning Lab

This repository is used for learning and experimenting with GitHub Actions, Go builds, Docker, and Kubernetes. I created and modified CI workflows while practicing common CI/CD patterns and troubleshooting issues.

> Purpose: This repo and its workflows were modified for learning purposes. The contents of this README summarize what I learned and useful commands to reproduce or verify locally.

What I learned
- GitHub Actions triggers and event filters
  - `pull_request` without `types` runs on many events (opened, synchronize, reopened, edited). Use `types: [opened]` if you only want the workflow to run once when the PR is created.
  - Use `github.head_ref` (PR source branch) and `github.ref_name` for branch names. Example: `BRANCH="${GITHUB_HEAD_REF:-${GITHUB_REF_NAME}}"`.
- Checkout & auth
  - `actions/checkout` needs a token when workflows perform pushes: `with: token: ${{ secrets.GH_TOKEN }}` (or `GITHUB_TOKEN`).
  - GitHub Actions can check out a detached HEAD; fetch and checkout the branch before making commits: `git fetch --all && git checkout "$BRANCH"`.
- Avoid detached HEAD and push safely
  - If you modify files in the job, checkout the correct branch first. Order: fetch -> checkout -> modify -> commit -> push.
  - Workflows do not merge PRs automatically. Pushing to `main` via `git push origin HEAD:main` updates main, but doesn't merge a PR. Be cautious with `-f` (force push).
- Go tooling in CI
  - `go mod download` to fetch dependencies.
  - `go build -o product-catalog-service main.go` to build the binary.
  - `go test ./...` to run unit tests.
- golangci-lint installation
  - Use a valid module version tag (e.g., `go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.57.2`).
  - Installed binaries land in `$(go env GOPATH)/bin` (commonly `$HOME/go/bin`) — add that to PATH in the workflow: `export PATH=$PATH:$(go env GOPATH)/bin`.
  - Run the linter from within the module directory (e.g., `cd src/product-catalog && golangci-lint run ./...`).
- Docker build context and Dockerfile path
  - `context` sets the build context. The `file` path is relative to the runner working directory; specify `file: src/product-catalog/Dockerfile` or use `file: ./Dockerfile` when the context is `src/product-catalog`.
  - Example (local): `docker build -f src/product-catalog/Dockerfile -t username/product-catalog:TAG src/product-catalog`.
- sed usage in workflows
  - If your replacement contains `/`, use a different delimiter like `#` or `|` to avoid escaping issues: `sed -i 's#image: .*#image: username/product-catalog:TAG#' file`.
- Common CI pitfalls I fixed
  - Chaining shell commands correctly with `&&` or using a multi-line `run: |` block.
  - Ensuring `actions/checkout` uses a token when pushing changes.
  - Checking out the correct branch and fetching all remote refs to avoid `pathspec` errors.
  - Running linters from module directory to avoid "directory prefix does not contain main module" errors.
  - Matching golangci-lint version to the Go toolchain to avoid buildir/import errors.

Commands I used (local / workflow equivalents)

- Build binary locally

```bash
cd src/product-catalog
go mod download
go build -o product-catalog-service main.go
```

- Run tests

```bash
cd src/product-catalog
go test ./...
```

- Install golangci-lint and run

```bash
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.57.2
export PATH=$PATH:$(go env GOPATH)/bin
cd src/product-catalog
golangci-lint run ./...
```

- Docker build (local)

```bash
docker build -f src/product-catalog/Dockerfile -t <username>/product-catalog:<tag> src/product-catalog
```

- Update kubernetes manifest safely with sed (CI-friendly)

```bash
# safe delimiter is #
sed -i 's#image: .*#image: <username>/product-catalog:<tag>#' kubernetes/productcatalog/deploy.yaml
```

- Create ArgoCD namespace (cluster)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Notes & recommended workflow changes
- Prefer updating the Kubernetes manifest in the same branch as the PR rather than force-pushing to `main` from CI.
- Use protected branch rules on `main` to avoid CI or automation force-pushes.
- Consider using the GitHub API or `gh` CLI to create/update a PR if you want automation to propose changes instead of pushing directly to `main`.
- Use `actions/checkout@v4` with `fetch-depth: 0` if you need full history or branch refs.

If you want, I can:
- Add a short CONTRIBUTING.md describing how to run CI locally.
- Open a PR that contains only the updated Kubernetes manifest instead of pushing to main directly.

---

This README was added automatically as part of a learning exercise. If you'd like the content reworded, shortened, or expanded (e.g., include code snippets for common fixes), tell me what to focus on.
