# Test Plan (Manual) — Helm + kubectl cho `pinpoint-stack` (Pinpoint Collector 3.0.3 + HBase 2.2.6 trên HDFS + ZooKeeper)

> Mục tiêu: hoàn thiện test plan bám sát **hiện trạng** Helm chart `helm/pinpoint-stack` trong repo này.
>
> Hiện trạng quan trọng cần nắm:
> - Chart **chỉ triển khai Pinpoint Collector** (không có Pinpoint Web/UI, không có Pinpoint Agent sample).
> - HDFS **chỉ có 1 NameNode** (chưa có HA Active/Standby + JournalNode).
> - HBase Master **1 replica**, RegionServer **3 replicas**.
> - Có `Schema Init Job` chạy theo hook `pre-install/pre-upgrade` để init schema Pinpoint trên HBase.
>
> Tài liệu liên quan: `helm/pinpoint-stack/values.yaml`, `helm/pinpoint-stack/templates/NOTES.txt`, `helm/runbook.md`, `test/test_viewpoint_pinpoint.md`.


Tài liệu này tối ưu cho tình huống bạn không dùng CI, chỉ chạy lệnh trực tiếp. Tôi chia thành các suite theo thứ tự chạy thực tế, kèm lệnh, tiêu chí pass fail và artifact cần lưu.

## 1. Chuẩn bị

### 1.1 Phạm vi & giả định (theo hiện trạng chart)

**In-scope (đang được Helm chart deploy):**
- HDFS: `NameNode` (1), `DataNode` (3)
- ZooKeeper: (3)
- HBase: `Master` (1), `RegionServer` (3)
- `Schema Init Job` (Helm hook pre-install/pre-upgrade)
- Pinpoint: **Collector** (Deployment, 3 replicas)

**Out-of-scope (hiện chart chưa có):**
- Pinpoint Web/UI
- Pinpoint Agent sample / traffic generator
- HDFS NameNode HA (Active/Standby + JournalNode)
- HBase Master HA (Active/Standby)

### 1.2 Biến môi trường dùng chung

Đặt các biến sau để copy/paste lệnh. Khuyến nghị dùng đúng naming pattern của chart.

- Namespace: `pp-test`
- Helm release: `pp`
- Chart path: `./helm/pinpoint-stack`
- Values file: `./helm/pinpoint-stack/values.yaml` (hoặc file override của bạn, ví dụ `values-dev.yaml`)

Gợi ý export (Linux/macOS, shell bash/zsh):
- `export NS=pp-test`
- `export REL=pp`
- `export CHART=./helm/pinpoint-stack`
- `export VALUES=values-dev.yaml`

Trên Windows (PowerShell), dùng:
- `$env:NS     = "pp-test"`
- `$env:REL    = "pp"`
- `$env:CHART  = ".\\helm\\pinpoint-stack"`
- `$env:VALUES = ".\\helm\\pinpoint-stack\\values-dev.yaml"`

### 1.3 Artifact cần lưu cho mỗi lần chạy

Tạo thư mục log theo timestamp, ví dụ `logs/2026-02-21_0900/` và lưu toàn bộ output dưới đây.

Lưu version công cụ
`helm version`
`kubectl version --short`

Lưu values và manifest thực tế của release
`helm get values ${REL} -n ${NS} -a > logs/.../helm-values.txt`
`helm get manifest ${REL} -n ${NS} > logs/.../helm-manifest.yaml`
`helm history ${REL} -n ${NS} > logs/.../helm-history.txt`

Lưu trạng thái tài nguyên
`kubectl get all -n ${NS} -o wide > logs/.../k8s-all.txt`
`kubectl get pvc -n ${NS} -o wide > logs/.../k8s-pvc.txt`
`kubectl get events -n ${NS} --sort-by=.lastTimestamp > logs/.../k8s-events.txt`

---

## 2. Suite A kiểm tra tĩnh trước khi cài

### TC A01 Helm lint

Mức độ Critical
Mục tiêu chart không lỗi cấu trúc

Lệnh
`helm lint ${CHART}`

Pass
Không có error, warning nếu có phải có lý do chấp nhận

Artifact
Lưu output lint

### TC A02 Render manifest theo values và kiểm nhanh các điểm rủi ro

Mức độ Critical
Mục tiêu tránh deploy sai image, sai selector, sai storage

Lệnh
`helm template ${REL} ${CHART} -n ${NS} -f ${VALUES} > logs/.../render.yaml`

Kiểm bằng grep các điểm sau
Image tag không phải latest
`grep -R "image:" logs/.../render.yaml`
Selector ổn định, không đổi linh tinh
`grep -R "selector:" -n logs/.../render.yaml`
PVC và storage class đúng
`grep -R "kind: PersistentVolumeClaim" -n logs/.../render.yaml`

Pass
Không thấy latest, PVC có StorageClassName phù hợp, selector không “trôi”

Artifact
render.yaml

---

## 3. Suite B cài mới và smoke

### TC B01 Tạo namespace sạch

Mức độ Critical

Lệnh
`kubectl delete ns ${NS} --wait=true` nếu bạn muốn làm sạch hoàn toàn
`kubectl create ns ${NS}`

Pass
Namespace tồn tại, không còn resource cũ

Artifact
kubectl output

### TC B02 Helm install clean

Mức độ Critical

Lệnh
`helm install ${REL} ${CHART} -n ${NS} -f ${VALUES} --wait --timeout 30m`

Nếu bạn không muốn wait vì chart lâu, dùng
`helm install ${REL} ${CHART} -n ${NS} -f ${VALUES}`

Pass
Không lỗi install, toàn bộ pod quan trọng lên Ready, job init Completed

Artifact
helm install log, events, describe các pod lỗi

### TC B03 Theo dõi readiness liveness và restart count

Mức độ High

Lệnh
`kubectl get pods -n ${NS} -w`
`kubectl get pods -n ${NS} -o custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount,STATUS:.status.phase`

Pass
Không CrashLoopBackOff kéo dài, restart không tăng liên tục

Artifact
snapshot output và events

### TC B04 Kiểm tra PVC Bound và dung lượng

Mức độ Critical

Lệnh
`kubectl get pvc -n ${NS} -o wide`
`kubectl describe pvc <pvc-name> -n ${NS}`

Pass
Trạng thái Bound, storage class đúng, access mode đúng

Artifact
k8s-pvc.txt và describe pvc

---

## 4. Suite C xác nhận HBase trên HDFS hoạt động bền vững

Phần này bắt buộc vì rủi ro lớn nhất là HDFS permission, WAL, lease, compaction.

### TC C01 Xác nhận HBase đọc ghi cơ bản

Mức độ Critical
Mục tiêu chứng minh HBase cluster thật sự usable, không chỉ “pod Ready”

Cách làm
Chạy lệnh test bằng hbase shell trực tiếp từ pod master.

Bước
1) Tìm pod hbase-master:
   `export MASTER_POD=$(kubectl get pod -l app.kubernetes.io/component=hbase-master -n ${NS} -o jsonpath='{.items[0].metadata.name}')`
2) Truy cập hbase shell:
   `kubectl exec -it ${MASTER_POD} -n ${NS} -- hbase shell`
3) Trong hbase shell, chạy các lệnh:
   - `create 'test_table', 'cf'`
   - `put 'test_table', 'row1', 'cf:qual1', 'value1'`
   - `get 'test_table', 'row1'`
   - `scan 'test_table'`
   - `disable 'test_table'`
   - `drop 'test_table'`

Pass
Không lỗi khi create, put, get, scan, drop.
Log regionserver không có lỗi fatal liên quan HDFS.

Artifact
Log thao tác hbase shell, log regionserver trong khoảng thời gian test.

### TC C02 Restart RegionServer, dữ liệu vẫn còn

Mức độ Critical

Bước
Ghi dữ liệu test như TC C01
Xóa một pod regionserver
`kubectl delete pod <regionserver-pod> -n ${NS}`
Chờ pod lên lại Ready
Đọc lại dữ liệu

Pass
Dữ liệu đọc đúng sau restart
Không xuất hiện lỗi kiểu WAL sync fail, lease recovery kẹt lâu

Artifact
kubectl events, logs regionserver trước và sau restart

### TC C03 Restart thành phần HDFS client side trong HBase và kiểm tra hồi phục

Mức độ High

Bước
Chọn một pod HBase master hoặc regionserver
Xóa pod đó
Chờ lên lại
Chạy lại một vòng put get nhỏ

Pass
Hồi phục bình thường, không stuck ở initializing region

Artifact
describe pod, logs

### TC C04 Test tình huống HDFS tạm thời không truy cập được

Mức độ High
Mục tiêu kiểm chứng hệ thống degrade có kiểm soát, không corrupt

Cách làm thủ công dễ nhất
Tạm thời chặn egress từ namespace tới HDFS endpoint bằng NetworkPolicy nếu cluster hỗ trợ, hoặc tạm dừng service HDFS trong môi trường test nếu bạn kiểm soát được.

Kịch bản
Trong lúc chặn, thử put get
Sau đó mở lại kết nối, thử lại

Pass
Trong lúc chặn có lỗi rõ ràng nhưng không làm pod crash hàng loạt
Sau khi mở lại, HBase phục hồi và tiếp tục đọc ghi

Artifact
events, logs regionserver, logs hdfs client

---

## 5. Suite D kiểm chứng Pinpoint end-to-end (theo hiện trạng: **Collector only**)

> Lưu ý: chart hiện tại **không deploy Pinpoint Web/UI**, nên tiêu chí kiểm chứng end-to-end tập trung vào:
> - Collector nhận được request (gRPC ports 9991/9992/9993)
> - Collector ghi được xuống HBase (xác minh qua logs + HBase tables/rows)
> - Hệ thống chịu được restart/scale collector khi có traffic

### TC D01 Ingest smoke vào Collector và xác nhận ghi xuống HBase

Mức độ: Critical

Tiền điều kiện
- Có Pinpoint Agent chạy ở ngoài cluster (VM/baremetal) **hoặc** một app sample có gắn agent.
- Agent trỏ endpoint tới service collector của release:
  - gRPC Agent: `${REL}-collector.${NS}.svc:9991`
  - gRPC Stat: `${REL}-collector.${NS}.svc:9992`
  - gRPC Span: `${REL}-collector.${NS}.svc:9993`

Bước
1) Port-forward (tùy chọn) để debug nhanh:
   - `kubectl port-forward -n ${NS} svc/${REL}-collector 9991:9991 9992:9992 9993:9993`
2) Tạo traffic đủ để agent gửi span/stat (ví dụ gọi API/HTTP loop 2–5 phút).
3) Kiểm tra logs collector không lỗi HBase/ZK:
   - `kubectl logs -n ${NS} deploy/${REL}-collector --since=15m`
4) Kiểm tra HBase có các bảng schema của Pinpoint (được tạo bởi schema init job) và có dữ liệu tăng lên:
   - Exec vào HBase master pod rồi dùng `hbase shell` (xem Suite C để lấy pod name).
   - `list` để thấy các table `Pinpoint*` (tên table tùy schema).
   - `count '<table>'` hoặc scan hạn chế để thấy row tăng (chọn 1–2 bảng liên quan span/stat).

Pass
- Collector không có lỗi kiểu: `NoSuchTableException`, `Failed to connect to Zookeeper`, `Call timeout`, `RpcThrottling`, `WAL`.
- HBase có schema Pinpoint và có row được ghi (count tăng sau khi tạo traffic).

Artifact
- Logs collector trong khoảng test
- Output `kubectl get svc -n ${NS}` (để lưu endpoints/ports)
- Trích output hbase shell: `list`, `count`/`scan`

### TC D02 Degrade test: scale down / restart 1 collector pod khi đang ingest

Mức độ: High

Bước
1) Đảm bảo đang có ingest (agent đang gửi data).
2) Xóa 1 pod collector:
   - `kubectl delete pod -n ${NS} -l app.kubernetes.io/name=collector`
   (hoặc scale):
   - `kubectl scale deploy/${REL}-collector -n ${NS} --replicas=1`
3) Quan sát rollout/availability:
   - `kubectl get pod -n ${NS} -l app.kubernetes.io/name=collector -w`
4) Kiểm tra logs collector sau disruption.
5) Scale up lại 3 replicas (nếu đã scale xuống):
   - `kubectl scale deploy/${REL}-collector -n ${NS} --replicas=3`

Pass
- Không xảy ra downtime toàn bộ (vẫn còn ít nhất 1 collector nhận dữ liệu).
- Collector phục hồi sau khi pod bị xóa/scale và tiếp tục ghi xuống HBase.

Artifact
- Events + logs collector trước/sau disruption
- Snapshot replica/pod status

---

## 6. Suite E upgrade và rollback an toàn

### TC E01 Upgrade thay đổi nhỏ, không phá PVC

Mức độ Critical

Chuẩn bị
Tạo một file patch values, ví dụ đổi một tham số cấu hình hoặc tài nguyên.

Lệnh
`helm upgrade ${REL} ${CHART} -n ${NS} -f ${VALUES} -f patch.yaml --wait --timeout 30m`

Kiểm tra
`kubectl get pvc -n ${NS} -o wide` trước và sau

Pass
PVC giữ nguyên
Pod rollout thành công
Pinpoint ingest smoke vẫn chạy
HBase test put get vẫn ok

Artifact
helm upgrade log, pvc snapshot, logs

### TC E02 Rollback sau upgrade lỗi có kiểm soát

Mức độ Critical

Bước
Cố ý upgrade với config sai để một component không lên
Xác nhận lỗi xảy ra
Xem revision
`helm history ${REL} -n ${NS}`
Rollback về revision trước
`helm rollback ${REL} <REV> -n ${NS} --wait --timeout 30m`

Pass
Tất cả pod quan trọng trở lại Ready
Dữ liệu test HBase còn
Ingest Pinpoint trở lại bình thường

Artifact
helm history, rollback log, logs component bị lỗi

---

## 7. Suite F resilience theo Kubernetes

### TC F01 Drain node đang chạy RegionServer

Mức độ High
Chỉ chạy nếu bạn có quyền drain

Bước
Xác định pod regionserver nằm trên node nào
`kubectl get pod -n ${NS} -o wide | grep hbase-rs`
Drain node đó theo chuẩn vận hành của bạn
Quan sát reschedule

Pass
Pod reschedule, HBase vẫn đọc ghi sau khi ổn định
Pinpoint không sập toàn hệ

Artifact
output drain, events, logs

### TC F02 PDB hoạt động đúng

Mức độ Medium

Bước
`kubectl get pdb -n ${NS}`
Thực hiện thao tác gây disruption và quan sát eviction

Pass
Không cho phép phá vượt ngưỡng cho nhóm quan trọng

Artifact
describe pdb, events

---

## 8. Suite G security tối thiểu và hygiene

### TC G01 Kiểm tra RBAC tối thiểu

Mức độ High

Bước
Xem serviceAccount và rolebinding
`kubectl get sa,role,rolebinding -n ${NS}`
Tìm lỗi forbidden trong logs

Pass
Không có forbidden khi runtime
Không cần cluster-admin

Artifact
rbac manifest và logs

### TC G02 Secret không lộ trong log, rotation cơ bản

Mức độ Medium

Bước
Thay secret theo quy trình
Upgrade để reload
Quét log
`kubectl logs <pod> -n ${NS} --since=30m | grep -i "password\|secret\|token"`

Pass
Không thấy plaintext secret
Workload vẫn chạy

Artifact
log scan

---

## 9. Checklist go live tối thiểu

HBase put get scan pass sau restart regionserver.
Pinpoint ingest end to end pass, UI có dữ liệu.
Upgrade nhỏ và rollback pass, PVC không đổi.
Drain một node không làm sập hệ và hệ tự hồi phục.
Có logs và metrics tối thiểu để debug khi lỗi.

---

## 10. Mẫu nhật ký chạy test thủ công

Bạn copy khung này làm report.

Thông tin run: ngày giờ, cluster, namespace, chart version, values commit hash
Kết quả: danh sách TC, pass fail, link artifact
Issue: mô tả, bước tái hiện, log trích, mức độ, đề xuất fix
Kết luận: go no go và điều kiện kèm theo
