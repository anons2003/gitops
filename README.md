# GitOps Lab Với ArgoCD

Lab này demo cách vận hành Kubernetes theo GitOps:

- Git là nguồn sự thật duy nhất.
- ArgoCD đọc manifest từ Git và sync vào cluster.
- Sửa tay trong cluster sẽ bị ArgoCD kéo lại theo Git.
- CI validate manifest trước khi merge vào `main`.

## Cấu Trúc Repo

```text
.
├── .github/
│   └── workflows/
│       └── validate.yml
├── argocd/
│   ├── root.yaml
│   └── apps/
│       └── web.yaml
└── k8s/
    ├── namespace.yaml
    └── web.yaml
```

## Chuẩn Bị

Cần có sẵn:

- Docker Desktop
- `minikube`
- `kubectl`
- `git`
- GitHub repo: `https://github.com/anons2003/gitops`

## Lab 0: Tạo Cluster

Tạo local Kubernetes cluster bằng minikube:

```bash
minikube start -p gitops --driver=docker
kubectl config use-context gitops
kubectl get nodes
```

Kết quả mong đợi:

```text
NAME     STATUS   ROLES
gitops   Ready    control-plane
```

## Lab 1: Cài ArgoCD

Tạo namespace và cài ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Đợi các pod ArgoCD sẵn sàng:

```bash
kubectl -n argocd get pods
kubectl -n argocd rollout status deploy/argocd-server
```

Các pod quan trọng nên ở trạng thái `1/1 Running`:

```text
argocd-application-controller
argocd-repo-server
argocd-server
argocd-redis
```

Mở ArgoCD UI:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Lấy password admin:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Đăng nhập:

```text
URL: https://localhost:8080
Username: admin
Password: output từ lệnh trên
```

## Lab 2: Tạo ArgoCD Application

File `argocd/apps/web.yaml` tạo ArgoCD Application tên `web`.

Ý nghĩa chính:

- `repoURL`: repo GitHub chứa manifest.
- `targetRevision`: branch `main`.
- `path`: folder trong repo chứa Kubernetes manifest, ở đây là `k8s`.
- `destination.namespace`: deploy app vào namespace `demo`.
- `automated.prune`: xóa resource khỏi cluster nếu resource bị xóa khỏi Git.
- `automated.selfHeal`: sửa tay trong cluster sẽ bị kéo lại theo Git.
- `CreateNamespace=true`: tự tạo namespace nếu chưa có.

Apply Application lần đầu:

```bash
kubectl apply -f argocd/apps/web.yaml
```

Kiểm tra:

```bash
kubectl -n argocd get app web
kubectl -n demo get deploy,pod
```

Kết quả mong đợi:

```text
web   Synced   Healthy
```

## Lab 3: Sync Và Self-Heal

Đổi `replicas` trong `k8s/web.yaml`, ví dụ:

```yaml
replicas: 4
```

Commit và push:

```bash
git add k8s/web.yaml
git commit -m "scale web to 4 replicas"
git push
```

Kiểm tra ArgoCD sync:

```bash
kubectl -n argocd get app web
kubectl -n demo get deploy web
```

Thử sửa tay cluster:

```bash
kubectl -n demo scale deploy/web --replicas=9
kubectl -n demo get deploy web -w
```

Kết quả mong đợi:

- Ban đầu Deployment có thể lên `9`.
- Sau đó ArgoCD self-heal về số replicas trong Git.

Chốt: trong GitOps, sửa tay không sống lâu. Muốn đổi thật thì sửa Git.

## Lab 4: Rollback Bằng Git Revert

Rollback đúng cách GitOps:

```bash
git revert HEAD --no-edit
git push
```

Kiểm tra:

```bash
kubectl -n argocd get app web
kubectl -n demo get deploy,pod
```

Không nên dùng `kubectl rollout undo` làm rollback chính trong GitOps, vì Git vẫn giữ version cuối cùng. ArgoCD sẽ đưa cluster quay lại theo Git.

## Lab 5: App Of Apps

`argocd/root.yaml` là Application cha. Nó trỏ vào folder:

```text
argocd/apps
```

Folder này chứa các Application con, ví dụ:

```text
argocd/apps/web.yaml
```

Apply root một lần:

```bash
kubectl apply -f argocd/root.yaml
```

Kiểm tra:

```bash
kubectl -n argocd get applications
```

Kết quả mong đợi:

```text
root   Synced   Healthy
web    Synced   Healthy
```

Từ lúc này, thêm app mới bằng cách thêm file Application vào `argocd/apps/` rồi push Git. Không cần `kubectl apply` từng Application nữa.

## Lab 6: Sync Waves

Sync waves ép ArgoCD apply resource theo thứ tự.

Thứ tự trong lab:

```text
Namespace   wave -1
ConfigMap   wave 0
Deployment  wave 1
Service     wave 2
```

Annotation dùng để khai báo wave:

```yaml
argocd.argoproj.io/sync-wave: "0"
```

Kiểm tra resource:

```bash
kubectl -n demo get configmap
kubectl -n demo get all
```

Kết quả mong đợi:

```text
configmap/web-config
deployment.apps/web
service/web
pod/web-...
```

Lưu ý:

- `web-config` là ConfigMap do GitOps quản lý.
- `kube-root-ca.crt` là ConfigMap Kubernetes tự tạo trong namespace.
- ReplicaSet hiện tại có pod running; ReplicaSet cũ có thể còn tồn tại với `0` replica để giữ rollout history.

## Lab 7: CI Validation Workflow

Workflow nằm tại:

```text
.github/workflows/validate.yml
```

Workflow chạy khi:

- Push lên `main`
- Mở pull request
- Bấm `Run workflow` thủ công trong GitHub Actions

Nó cài `kubeconform` và validate manifest trong `k8s/`:

```bash
kubeconform -strict -summary -ignore-missing-schemas k8s/
```

Commit workflow:

```bash
git add .github/workflows/validate.yml
git commit -m "add test validation workflow"
git push
```

## Branch Protection / Ruleset

Trong GitHub:

```text
Settings -> Rules -> Rulesets -> New branch ruleset
```

Cấu hình khuyến nghị:

```text
Ruleset name: protect-main
Enforcement status: Active
Target branch: main
Require a pull request before merging: enabled
Require status checks to pass: enabled
Required check: validate
Block force pushes: enabled
Restrict deletions: enabled
```

Với lab cá nhân:

```text
Required approvals: 0
Require branches to be up to date before merging: disabled
```

## Test Branch Protection

Tạo branch test:

```bash
git checkout -b test-ci
```

Sửa một file trong `k8s/`, ví dụ đổi `replicas` trong `k8s/web.yaml`.

Commit và push:

```bash
git add k8s/web.yaml
git commit -m "test ci by changing replicas"
git push -u origin test-ci
```

Lên GitHub tạo Pull Request từ `test-ci` vào `main`.

Kết quả mong đợi:

- PR hiện check `validate`.
- PR chỉ merge được khi `validate` pass.

## Lệnh Hữu Ích

Xem app trong ArgoCD:

```bash
kubectl -n argocd get applications
kubectl -n argocd get app web
```

Xem resource app:

```bash
kubectl -n demo get all
kubectl -n demo get configmap
```

Xem rollout history:

```bash
kubectl -n demo rollout history deployment/web
```

Refresh ArgoCD Application:

```bash
kubectl -n argocd annotate app web argocd.argoproj.io/refresh=hard --overwrite
```

Manual sync bằng `kubectl patch`:

```bash
kubectl -n argocd patch app web --type merge \
  -p '{"operation":{"sync":{}}}'
```

## Ý Chính Cần Nhớ

- Git là nguồn sự thật duy nhất.
- ArgoCD reconcile cluster về đúng Git.
- Self-heal giúp sửa tay không tồn tại lâu.
- App-of-apps giúp quản lý nhiều Application bằng một root.
- Sync waves giúp apply resource đúng thứ tự.
- CI validate manifest trước khi merge vào `main`.
