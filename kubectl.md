# kubectl Cheat Sheet
 
คำสั่งเรียงตามงานจริง: ดู → ไล่ปัญหา → แก้ → ดูแล node
 
**ตัวย่อที่ควรจำ:** `po` pods · `deploy` deployments · `svc` services · `ns` namespaces · `ing` ingresses · `cm` configmaps · `sts` statefulsets · `ds` daemonsets · `rs` replicasets · `ep` endpoints · `no` nodes · `cj` cronjobs
 
## สารบัญ
 
- [Context & Namespace](#context--namespace)
- [ดู Resource](#ดู-resource)
- [Logs & Debug](#logs--debug)
- [Apply, Edit & Delete](#apply-edit--delete)
- [Deployment, Rollout & Scale](#deployment-rollout--scale)
- [Node & Maintenance](#node--maintenance)
- [Config, Secret & Storage](#config-secret--storage)
- [Service, Ingress & Job](#service-ingress--job)
- [RBAC & Auth](#rbac--auth)
- [JSONPath & Output](#jsonpath--output)
- [Alias & Plugin](#alias--plugin)
---
 
## Context & Namespace
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl config get-contexts` | ดู cluster ทั้งหมดที่มีใน kubeconfig (`*` คือตัวที่ใช้อยู่) |
| `kubectl config use-context prod` | สลับ cluster |
| `kubectl config set-context --current --namespace=payment` | ตั้ง default namespace ของ context นี้ — ไม่ต้องพิมพ์ `-n` ทุกครั้ง |
| `kubectl config current-context` | เช็คว่าอยู่ cluster ไหน (**เช็คก่อนพิมพ์คำสั่งอันตราย**) |
| `kubectl cluster-info` | endpoint ของ control plane |
| `kubectl config view --minify --flatten` | ดึง config ของ context ปัจจุบันออกมาแบบสมบูรณ์ (แชร์ให้ CI ได้) |
| `kubectl api-resources \| grep -i ingress` | ดูว่า resource นั้นชื่อย่ออะไร, namespaced ไหม, apiVersion อะไร |
| `kubectl explain deployment.spec.strategy --recursive` | อ่าน schema ของ field จาก cluster เอง — ไม่ต้องเปิดเว็บ |
| `kubectl version --short` | เทียบ version client / server |
| `kubectl get nodes -o wide` | ดู version ของ kubelet แต่ละ node |
 
---
 
## ดู Resource
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl get pods -A -o wide` | pod ทุก namespace + node/IP ที่รันอยู่ |
| `kubectl get po --sort-by=.status.containerStatuses[0].restartCount` | เรียงตามจำนวน restart — หา pod ที่ crash วนอยู่ |
| `kubectl get po --field-selector status.phase!=Running -A` | เอาแต่ pod ที่ไม่ปกติ (Pending / Failed) |
| `kubectl get po -l app=api,tier=backend --show-labels` | filter ด้วย label + โชว์ label ทั้งหมด |
| `kubectl get all -n payment` | ดู workload หลักใน namespace (ไม่รวม cm/secret/ing/pvc) |
| `kubectl describe po <pod>` | **คำสั่งไล่ปัญหาอันดับหนึ่ง** — ดู Events ท้ายสุดก่อนเลย (ImagePullBackOff, FailedScheduling, OOMKilled) |
| `kubectl get events --sort-by=.lastTimestamp -A \| tail -30` | ไทม์ไลน์เหตุการณ์ล่าสุดของ cluster |
| `kubectl get po -w` | watch การเปลี่ยนสถานะสด ๆ ตอน deploy |
| `kubectl get deploy -o yaml > backup.yaml` | export manifest ไว้ก่อนแก้อะไร |
| `kubectl top po --sort-by=memory` | RAM จริงที่ใช้ (ต้องมี metrics-server) — เทียบกับ requests/limits |
| `kubectl top no` | ภาระของแต่ละ node |
 
เลือก column เอง (ใช้ทำรายงานเวอร์ชัน image):
 
```bash
kubectl get po -o custom-columns='N:.metadata.name,NODE:.spec.nodeName,IMG:.spec.containers[*].image'
```
 
---
 
## Logs & Debug
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl logs -f <pod> -c app --tail=200 --timestamps` | tail log ต่อเนื่อง ระบุ container ด้วย `-c` เมื่อมีหลาย container |
| `kubectl logs <pod> --previous` | **log ของ container รอบที่ตายไปแล้ว** — ตัวช่วยหลักตอน CrashLoopBackOff |
| `kubectl logs -l app=api --all-containers --max-log-requests=10 -f` | รวม log จากทุก pod ที่ตรง label |
| `kubectl logs deploy/api --since=15m` | ดู log ผ่าน deployment ตรง ๆ ไม่ต้องหาชื่อ pod |
| `kubectl exec -it <pod> -- sh` | เข้า shell ใน container (ใช้ `bash` ถ้า image มี) |
| `kubectl exec <pod> -- env \| sort` | ตรวจ env var จริงที่ container เห็น |
| `kubectl debug <pod> -it --image=nicolaka/netshoot --target=app` | **ephemeral container** — แนบเครื่องมือ network เข้า pod ที่ image ไม่มี shell/curl |
| `kubectl run tmp --rm -it --image=nicolaka/netshoot --restart=Never -- sh` | เปิด pod ชั่วคราวเพื่อ curl/dig ทดสอบภายใน cluster |
| `kubectl port-forward svc/api 8080:80` | ต่อ service มาที่เครื่องเรา (`pod/` หรือ `deploy/` ก็ได้) — ทดสอบโดยไม่ต้องเปิด public |
| `kubectl cp <pod>:/app/log.txt ./log.txt` | คัดลอกไฟล์เข้า/ออกจาก container |
| `kubectl attach -it <pod>` | เกาะ stdout/stdin ของ process หลัก |
 
> :bulb: **ลำดับไล่ปัญหา**
> `get po` (สถานะ) → `describe po` (Events) → `logs --previous` (ทำไมตาย) → `exec` / `debug` (เข้าไปดูข้างใน) → `get events` (ภาพรวม cluster)
 
---
 
## Apply, Edit & Delete
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl apply -f ./k8s/ --recursive` | apply ทุกไฟล์ในโฟลเดอร์ (declarative — ใช้เป็นหลัก) |
| `kubectl apply -k ./overlays/prod` | apply ผ่าน Kustomize (มีในตัว ไม่ต้องลงเพิ่ม) |
| `kubectl diff -f deploy.yaml` | **ดูก่อนว่าจะเปลี่ยนอะไร** เทียบกับของที่รันอยู่ — ทำทุกครั้งก่อน apply บน prod |
| `kubectl apply -f x.yaml --dry-run=server` | ให้ API server ตรวจ (ผ่าน webhook/validation) แต่ไม่บันทึก |
| `kubectl create deploy api --image=nginx --dry-run=client -o yaml > deploy.yaml` | สูตรสร้าง manifest เร็ว ๆ แทนการพิมพ์ YAML เปล่า |
| `kubectl edit deploy api` | แก้สด ๆ ใน editor (แก้ฉุกเฉิน — จำไว้ว่าจะหลุดจาก Git) |
| `kubectl patch deploy api -p '{"spec":{"replicas":5}}'` | strategic merge patch — แก้เฉพาะ field ที่ต้องการ |
| `kubectl set image deploy/api app=repo/app:v2` | เปลี่ยน image = trigger rolling update |
| `kubectl delete po <pod> --grace-period=0 --force` | ลบ pod ค้าง Terminating (ใช้เมื่อจำเป็นจริง ๆ) |
| `kubectl delete -f x.yaml` | ลบตามไฟล์ |
| `kubectl delete po -l app=api` | ลบตาม label |
| `kubectl replace --force -f x.yaml` | ลบแล้วสร้างใหม่ทั้งก้อน (สำหรับ field ที่ immutable) |
 
---
 
## Deployment, Rollout & Scale
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl rollout status deploy/api --timeout=180s` | รอจน rollout เสร็จ — ใส่ใน CI pipeline เพื่อให้ job fail ถ้า deploy ไม่ขึ้น |
| `kubectl rollout history deploy/api` | ดูประวัติ revision |
| `kubectl rollout undo deploy/api --to-revision=3` | **rollback** — ไม่ระบุ revision = ย้อนไปอันก่อนหน้า |
| `kubectl rollout restart deploy/api` | restart pod ทั้งชุดแบบ rolling (ใช้ตอนเปลี่ยน secret/configmap แล้วต้องโหลดใหม่) |
| `kubectl rollout pause deploy/api` | หยุด rollout กลางทางเพื่อดูผล = canary แบบมือ |
| `kubectl rollout resume deploy/api` | ไปต่อ |
| `kubectl scale deploy/api --replicas=4` | ปรับจำนวน replica ตรง ๆ |
| `kubectl autoscale deploy/api --min=2 --max=10 --cpu-percent=70` | สร้าง HPA เร็ว ๆ |
| `kubectl get hpa -w` | ดู HPA ตัดสินใจ scale ตาม metric |
| `kubectl get rs --sort-by=.metadata.creationTimestamp` | ดู ReplicaSet เก่า/ใหม่ ตอนสงสัยว่า rollout ค้างที่ไหน |
 
---
 
## Node & Maintenance
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl get no -L topology.kubernetes.io/zone -L node.kubernetes.io/instance-type` | ดู node แยกตาม AZ และ instance type (สำคัญบน EKS) |
| `kubectl describe no <node> \| grep -A10 Allocated` | ดูว่า node จอง resource ไปเท่าไร (แก้ปัญหา pod Pending) |
| `kubectl cordon <node>` | ห้าม schedule pod ใหม่ลง node นี้ (pod เดิมยังอยู่) |
| `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | ย้าย pod ออกก่อน patch/ปลด node — เคารพ PodDisruptionBudget |
| `kubectl uncordon <node>` | เปิดให้ schedule ได้อีกครั้ง |
| `kubectl taint no <node> workload=batch:NoSchedule` | กัน node ไว้ให้ workload เฉพาะ (คู่กับ tolerations) — ใส่ `-` ท้ายเพื่อลบ |
| `kubectl label no <node> disktype=ssd` | ติด label ให้ nodeSelector / nodeAffinity ใช้ |
| `kubectl get pdb -A` | ดู PodDisruptionBudget — ตัวที่ทำให้ drain ค้างเมื่อ replica เหลือน้อย |
| `kubectl get po -A -o wide --field-selector spec.nodeName=<node>` | ดูว่ามี pod อะไรอยู่บน node นั้น |
 
---
 
## Config, Secret & Storage
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl create cm app-cfg --from-file=./app.conf --from-literal=LOG=debug` | สร้าง ConfigMap จากไฟล์และค่าเดี่ยว |
| `kubectl create secret generic db --from-literal=pass=s3cr3t` | สร้าง Secret (เป็นแค่ base64 ไม่ใช่การเข้ารหัส — ต้องเปิด encryption at rest หรือใช้ external secret) |
| `kubectl get secret db -o jsonpath='{.data.pass}' \| base64 -d` | อ่านค่า secret ออกมา |
| `kubectl get pvc,pv -A` | ดู volume claim และ volume จริง — เช็ค StorageClass / ReclaimPolicy |
| `kubectl get sc` | StorageClass ที่มี (บน EKS มักเป็น gp3 ผ่าน ebs-csi-driver) |
 
imagePullSecret สำหรับ private registry (ECR):
 
```bash
kubectl create secret docker-registry ecr \
  --docker-server=<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region ap-southeast-1)"
```
 
สูตร **update ConfigMap ที่มีอยู่แล้ว** โดยไม่ต้องลบก่อน:
 
```bash
kubectl create cm app-cfg --from-file=app.conf \
  --dry-run=client -o yaml | kubectl apply -f -
```
 
---
 
## Service, Ingress & Job
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl get svc -o wide` | ดู type / cluster IP / external IP / selector |
| `kubectl get ep api` | **Endpoints ว่าง = selector ไม่ตรง label ของ pod** — สาเหตุอันดับหนึ่งที่ service ไม่ทำงาน |
| `kubectl expose deploy api --port=80 --target-port=8080 --type=ClusterIP` | สร้าง service ให้ deployment เร็ว ๆ |
| `kubectl get ing -A` | ดู ingress ทั้งหมด (บน EKS มักผ่าน AWS Load Balancer Controller → ALB) |
| `kubectl describe ing web \| tail -20` | อ่าน event ของ ingress controller ตอน ALB ไม่ถูกสร้าง |
| `kubectl create job --from=cronjob/nightly manual-run` | สั่งรัน CronJob ทันทีโดยไม่ต้องรอเวลา |
| `kubectl delete job --field-selector status.successful=1` | เก็บกวาด job ที่เสร็จแล้ว |
| `kubectl get po -n kube-system -l k8s-app=kube-dns` | เช็ค CoreDNS เวลา service discovery ภายในมีปัญหา |
 
---
 
## RBAC & Auth
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `kubectl auth can-i delete po -n payment` | เช็คสิทธิ์ตัวเอง |
| `kubectl auth can-i get po --as=system:serviceaccount:payment:api` | เช็คสิทธิ์แทน ServiceAccount อื่น |
| `kubectl auth can-i --list -n payment` | ลิสต์ทุกอย่างที่เราทำได้ใน namespace นั้น |
| `kubectl get sa,role,rolebinding -n payment` | ตรวจ RBAC ของ namespace |
| `kubectl get clusterrole,clusterrolebinding` | ตรวจสิทธิ์ระดับ cluster |
 
---
 
## JSONPath & Output
 
ทำรายงาน deployment → image ทั้ง namespace:
 
```bash
kubectl get deploy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].image}{"\n"}{end}'
```
 
หา pod ที่ค้างสถานะ Pending:
 
```bash
kubectl get po -o jsonpath='{.items[?(@.status.phase=="Pending")].metadata.name}'
```
 
หา container ที่ไม่ได้ตั้ง resource limits (ของเช็คก่อน production review):
 
```bash
kubectl get po -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{@.name}{" "}{@.resources.limits}{"\n"}{end}{end}' | grep -v 'map\['
```
 
---
 
## Alias & Plugin
 
```bash
# ~/.bashrc หรือ ~/.zshrc
alias k=kubectl
alias kgp='kubectl get po -o wide'
alias kd='kubectl describe'
alias kl='kubectl logs -f'
alias kx='kubectl exec -it'
 
# ให้ tab-complete ทำงานกับ alias ด้วย
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```
 
plugin ที่คุ้มค่าลง (ผ่าน [krew](https://krew.sigs.k8s.io/)):
 
```bash
kubectl krew install ctx ns stern
```
 
| plugin | ทำอะไร |
|---|---|
| `kubectx` | สลับ cluster เร็ว ๆ |
| `kubens` | สลับ namespace เร็ว ๆ |
| `stern` | tail log หลาย pod พร้อมกันแบบมีสี |
 
> :warning: **ระวัง**
> - `kubectl delete` ไม่มี undo และไม่ถาม — เช็ค `kubectl config current-context` ก่อนเสมอ และเลี่ยง `--all` บน prod
> - `kubectl edit` ทำให้ cluster ต่างจาก Git (configuration drift) ควรกลับไปแก้ที่ repo แล้ว apply ใหม่
> - `kubectl scale --replicas=0` แล้วปรับกลับ = มี downtime · ถ้าอยาก restart ให้ใช้ `kubectl rollout restart`
 
---
 
[← กลับหน้าสารบัญ](./README.md)
