# Runbook DEV — Chuẩn bị môi trường Windows để chạy `test_plan_detail.md` cho `pinpoint-stack` trên AKS

Tài liệu này hướng dẫn **cấu hình máy dev Windows** và **cách chạy test manual/automation** theo `test/test_plan_detail.md` cho Helm chart `helm/pinpoint-stack` trên **AKS**.

Lưu ý: trong repo hiện tại, **thư mục `test` và thư mục `helm` không nằm chung một thư mục root** (ví dụ: `C:\Sk\hbaseCluster\test` và `C:\Sk\pinpoint-helm\helm`). Vì vậy, khi chạy `.ps1` hoặc `.py` bạn **cần đặt đường dẫn tuyệt đối hoặc chỉnh lại biến môi trường cho phù hợp**. Chi tiết ở mục 5 và 11.

---

## 1. Bối cảnh & phạm vi

- Hệ điều hành: **Windows 10/11** (64-bit).
- Kubernetes: **AKS** đã được tạo sẵn (cluster do team infra cung cấp).
- Ứng dụng: `pinpoint-stack` gồm HDFS + ZooKeeper + HBase + Pinpoint Collector.
- Helm chart: thư mục `helm/pinpoint-stack` (có thể nằm ở repo khác/thư mục khác so với `test`).
- Tài liệu test: `test/test_plan_detail.md` (Suite A–G).

Mục tiêu:
- Chuẩn bị đầy đủ **tooling** trên Windows (az cli, kubectl, helm, git).
- Kết nối từ Windows vào AKS (kubeconfig, context).
- Chuẩn bị **values file**, **biến môi trường** đúng như test plan, dù `helm/` và `test/` ở hai chỗ khác nhau.
- Hướng dẫn chạy **manual test** và **automation script (.ps1, .py)** đúng thư mục.

---

## 2. Cài đặt công cụ bắt buộc trên Windows

### 2.1 Azure CLI

1) Tải file `.msi` từ:  
   https://learn.microsoft.com/cli/azure/install-azure-cli-windows

2) Cài đặt theo wizard (Next → Next → Finish).

3) Kiểm tra:

```powershell
az --version
```

### 2.2 kubectl

**Cách khuyến nghị (qua Azure CLI):**

```powershell
az aks install-cli
kubectl version --client
```

**Hoặc dùng Chocolatey (nếu có):**

```powershell
choco install kubernetes-cli -y
kubectl version --client
```

### 2.3 Helm 3

Nếu dùng Chocolatey:

```powershell
choco install kubernetes-helm -y
helm version
```

Hoặc tải từ GitHub Releases (https://github.com/helm/helm/releases), giải nén `helm.exe` và thêm vào `PATH`.

### 2.4 Git

```powershell
choco install git -y
git --version
```

---

## 3. Lấy source & chuẩn bị thư mục log

### 3.1 Cấu trúc thư mục thực tế (quan trọng)

Trong môi trường hiện tại, bạn có:

- Thư mục chứa test: ví dụ `C:\Sk\hbaseCluster\test`.
- Thư mục chứa chart Helm: ví dụ `C:\Sk\hbaseCluster\helm\pinpoint-stack`.

Hai thư mục này **có thể** không nằm chung một `git repo` hoặc không chung root. Vì vậy:

- Khi chạy script trong `test\` (ví dụ `run.ps1`, automation `.py`), **không thể dùng đường dẫn tương đối kiểu `./helm/pinpoint-stack` nếu chart nằm ở repo khác**.
- Thay vào đó, phải dùng **đường dẫn tuyệt đối** (absolute path) hoặc **đường dẫn tương đối từ vị trí script test sang thư mục helm**.

Ví dụ đường dẫn tuyệt đối cho chart (trường hợp giống môi trường hiện tại):

```powershell
# Ví dụ: chart nằm ở C:\Sk\hbaseCluster\helm\pinpoint-stack
$env:CHART = "C:\\Sk\\hbaseCluster\\helm\\pinpoint-stack"
$env:VALUES = "C:\\Sk\\hbaseCluster\\helm\\pinpoint-stack\\values.yaml"
```

### 3.2 Chuẩn bị thư mục log theo timestamp

Từ thư mục `C:\Sk\hbaseCluster\test` (nơi chứa file runbook và script):

```powershell
$timestamp = Get-Date -Format "yyyy-MM-dd_HHmm"
$logDir    = "logs\$timestamp"
New-Item -ItemType Directory -Force -Path $logDir | Out-Null
Write-Host "Log dir: $logDir"
```

Tất cả lệnh lưu artifact bên dưới sẽ dùng biến `$logDir` này.

---

## 4. Kết nối Windows vào AKS

### 4.1 Đăng nhập Azure & chọn subscription

Trong môi trường hiện tại, bạn dùng subscription:
- Tên: `FSO-akaOps`
- ID: `920cd9be-adb4-44b0-bed0-25bfef1ee2d9`

```powershell
az login
# Chọn subscription FSO-akaOps khi được hỏi
az account set --subscription 920cd9be-adb4-44b0-bed0-25bfef1ee2d9
az account list -o table
```

### 4.2 Lấy kubeconfig từ AKS

Ví dụ của bạn:
- Resource group: `TuanLN2-RG`
- AKS cluster: `pinpoint`

```powershell
az aks get-credentials `
  --resource-group TuanLN2-RG `
  --name pinpoint `
  --overwrite-existing

kubectl config get-contexts
kubectl get nodes
```

`kubectl get nodes` phải hiển thị các node ở trạng thái `Ready`.

---

## 5. Biến môi trường dùng chung cho test (PowerShell, có phân biệt đường dẫn Helm)

`test_plan_detail.md` dùng các biến:
- `NS` (namespace)
- `REL` (helm release name)
- `CHART` (đường dẫn chart)
- `VALUES` (file values dùng khi install/upgrade)

### 5.1 Trường hợp `helm/` và `test/` chung một repo

Nếu thư mục `test` và thư mục `helm` cùng nằm dưới một root (ví dụ `C:\Sk\hbaseCluster\helm` và `C:\Sk\hbaseCluster\test`), từ root repo bạn có thể dùng:

```powershell
$env:NS     = "pp-test"                          # namespace test
$env:REL    = "pp"                               # release name
$env:CHART  = ".\helm\pinpoint-stack"          # path chart tương đối từ root repo
$env:VALUES = ".\helm\pinpoint-stack\values-dev.yaml"  # file values override (nếu có)
```

### 5.2 Trường hợp thực tế hiện tại: chart nằm trong `C:\Sk\hbaseCluster\helm\pinpoint-stack`

Ví dụ:
- `C:\Sk\hbaseCluster\test`  (chạy .ps1, .py ở đây)
- `C:\Sk\hbaseCluster\helm\pinpoint-stack` (chứa chart)

Khi đó, trong PowerShell (ở thư mục `C:\Sk\hbaseCluster\test`), bạn có thể đặt:

```powershell
$env:NS     = "pp-test"                                           # namespace test
$env:REL    = "pp"                                                # release name
$env:CHART  = "C:\Sk\hbaseCluster\helm\pinpoint-stack"        # đường dẫn tuyệt đối tới chart
$env:VALUES = "C:\Sk\hbaseCluster\helm\pinpoint-stack\values.yaml"
```

Lưu ý:
- Tất cả lệnh `helm`/`kubectl` trong runbook và trong automation `.ps1`, `.py` cần sử dụng `$env:CHART` và `$env:VALUES` dạng phù hợp với cấu trúc thư mục thực tế.

---

## 11. Automation: `run.ps1` và pytest `.py` khi `test` và `helm` tách thư mục

### 11.1 Chạy `run.ps1` (Suite A+B automation cơ bản)

File: `C:\Sk\hbaseCluster\test\run.ps1`

Trong môi trường hiện tại, `run.ps1` đã được cấu hình sẵn giá trị mặc định:

```powershell
param(
  [string]$NS = "pp-test",
  [string]$REL = "pp",
  [string]$CHART = "C:\\Sk\\hbaseCluster\\helm\\pinpoint-stack",
  [string]$VALUES = "C:\\Sk\\hbaseCluster\\helm\\pinpoint-stack\\values.yaml",
  [string]$AKS_RG = "TuanLN2-RG",
  [string]$AKS_NAME = "pinpoint",
  [switch]$CleanNamespace,
  [int]$TimeoutMinutes = 30
)
```

**Bước chuẩn bị bắt buộc trước khi chạy script:**

Tạo namespace test (nếu chưa tồn tại):

```powershell
kubectl create namespace pp-test
kubectl get ns pp-test
```

Sau đó chạy script:

```powershell
cd C:\Sk\hbaseCluster\test

.un.ps1 `
  -NS "pp-test" `
  -REL "pp" `
  -CleanNamespace `
  -TimeoutMinutes 30
```

Script sẽ tự động:
- Chạy `helm version`, `kubectl version --client`.
- Chạy `helm lint`, `helm template` và lưu artifact (Suite A).
- Xóa/tạo lại namespace (nếu dùng `-CleanNamespace`).
- Chạy `helm install ... --wait --timeout ...` và lưu artifact (một phần Suite B).

### 11.2 Chạy pytest automation (`test/automation/test_suites.py`)

File: `C:\Sk\hbaseCluster\test\automation\test_suites.py`

`conftest.py` có thể sử dụng các biến môi trường sau:

```python
namespace = env("NS", "pp-test")
release   = env("REL", "pp")
chart     = env("CHART", r"C:\\Sk\\hbaseCluster\\helm\\pinpoint-stack")
values    = env("VALUES", r"C:\\Sk\\hbaseCluster\\helm\\pinpoint-stack\\values.yaml")
```

Bạn có thể chạy trực tiếp (không cần set env nếu dùng đúng mặc định):

```powershell
cd C:\Sk\hbaseCluster\test
pytest .\automation\test_suites.py -q
```

Hoặc nếu muốn override (ví dụ đổi namespace/release):

```powershell
cd C:\Sk\hbaseCluster\test

$env:NS     = "pp-test"
$env:REL    = "pp"
$env:CHART  = "C:\Sk\hbaseCluster\helm\pinpoint-stack"
$env:VALUES = "C:\Sk\hbaseCluster\helm\pinpoint-stack\values.yaml"

pytest .\automation\test_suites.py -q
```

Mapping giữa automation và Test Plan:
- Suite A:
  - `TC A01` → `test_a01_helm_lint`
  - `TC A02` → `test_a02_helm_template_check`
- Suite B:
  - `TC B01` → `test_b01_prepare_namespace`
  - `TC B02` → `test_b02_helm_install`
  - `TC B03` → `test_b03_verify_pods_ready`
  - `TC B04` → `test_b04_verify_pvc_bound`
- Suite C:
  - `TC C01` → `test_c01_hbase_basic_rw`
- Suite E:
  - `TC E01` → `test_e01_upgrade_safe`
  - `TC E02` → `test_e02_rollback`

Các Suite D, F, G hiện chưa có automation → vẫn chạy manual theo `test/test_plan_detail.md`.
