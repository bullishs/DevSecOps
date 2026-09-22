# k9s Cheat Sheet
 
k9s คือ terminal UI สำหรับ Kubernetes — จุดแข็งคือดูสถานะแบบ realtime, ดู log, exec, port-forward ได้ในไม่กี่ปุ่ม
 
**หลักคิด:** คีย์ตัวเดียว = คำสั่ง · `:` = ไปที่ resource · `/` = กรอง · `Esc` = ถอยกลับ · `?` = help
 
## สารบัญ
 
- [เปิดใช้งาน (CLI flags)](#เปิดใช้งาน-cli-flags)
- [คีย์พื้นฐาน / Navigation](#คีย์พื้นฐาน--navigation)
- [Filter & ค้นหา](#filter--ค้นหา)
- [ทำงานกับ Pod](#ทำงานกับ-pod)
- [Log view](#log-view)
- [เครื่องมือวินิจฉัย](#เครื่องมือวินิจฉัย)
- [Config & Skin](#config--skin)
---
 
## เปิดใช้งาน (CLI flags)
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `k9s` | เปิดด้วย context ปัจจุบันของ kubeconfig |
| `k9s -n payment` | เปิดที่ namespace ที่ต้องการ (`-n all` = ทุก namespace) |
| `k9s --context prod` | เลือก cluster ตอนเปิด |
| `k9s -c deploy` | เปิดมาที่ view ของ resource ที่ระบุเลย (`-c po`, `-c svc`, `-c ctx`) |
| `k9s --readonly` | **โหมดอ่านอย่างเดียว** — ควรใช้เสมอเวลาเข้าดู production |
| `k9s --headless --logoless` | ซ่อนแถบหัว/โลโก้ ได้พื้นที่จอมากขึ้น (กด `Ctrl-E` สลับได้ตอนรัน) |
| `k9s --kubeconfig ~/.kube/eks-prod` | ชี้ kubeconfig อีกไฟล์ |
| `k9s --refresh 5` | ปรับรอบ refresh เป็นวินาที (cluster ใหญ่ตั้งให้ช้าลงจะเบา API server) |
| `k9s info` | บอก path ของ config / log / skin ที่ k9s ใช้จริง |
 
---
 
## คีย์พื้นฐาน / Navigation
 
| คีย์ | ทำอะไร |
|---|---|
| `?` | เปิดหน้า help คีย์ทั้งหมด (จำอันนี้อันเดียวก็รอด) |
| `:` | เข้า command mode แล้วพิมพ์ชื่อ resource |
| `Esc` | ยกเลิก / ถอยกลับ view ก่อนหน้า |
| `Enter` | เจาะลึกเข้าไป (deployment → pod → container) |
| `d` | describe resource ที่เลือก |
| `y` | ดู YAML ฉบับเต็มของ resource |
| `e` | edit resource ด้วย `$EDITOR` |
| `Ctrl-D` | ลบ resource (มีหน้ายืนยัน) |
| `Ctrl-K` | kill resource ทันทีแบบไม่รอ grace period |
| `0`–`9` | สลับ namespace เร็ว ๆ (`0` = all namespaces) |
| `Shift-<C>` | เรียงตาม column — ตัวใหญ่ + อักษรแรกของ column เช่น `Shift-C` CPU, `Shift-M` MEM, `Shift-R` Restarts, `Shift-A` Age, `Shift-N` Name |
| `Ctrl-W` | สลับโชว์ทุก column (wide) แบบ `-o wide` |
| `Ctrl-A` | ลิสต์ alias/resource ทั้งหมดที่ k9s รู้จัก (รวม CRD) |
| `Ctrl-S` | save view ปัจจุบันเป็นไฟล์ในโฟลเดอร์ screen dump |
| `:q` / `Ctrl-C` | ออกจาก k9s |
 
### Resource ที่พิมพ์ต่อจาก `:`
 
| คำสั่ง | resource |
|---|---|
| `:po` | pods |
| `:deploy` | deployments |
| `:svc` | services |
| `:ing` | ingresses |
| `:no` | nodes |
| `:cm` | configmaps |
| `:secret` | secrets |
| `:pvc` | persistent volume claims |
| `:sts` | statefulsets |
| `:ds` | daemonsets |
| `:job` / `:cj` | jobs / cronjobs |
| `:ev` | events |
| `:hpa` | horizontal pod autoscalers |
| `:crd` | custom resource definitions |
| `:ctx` | เลือก/สลับ cluster |
| `:ns` | เลือก namespace |
| `:pf` | รายการ port-forward ที่เปิดไว้ |
| `:rbac` / `:sa` / `:role` | ตรวจ RBAC จาก UI |
 
> พิมพ์ namespace ต่อท้ายได้เลย เช่น `:po payment` หรือระบุ context ตรง ๆ `:ctx prod`
 
---
 
## Filter & ค้นหา
 
| คำสั่ง | ทำอะไร |
|---|---|
| `/api` | กรองรายการด้วยข้อความ |
| `/-l app=api` | กรองด้วย **label selector** |
| `/!Running` | กรองแบบ inverse — ซ่อนรายการที่ตรง (ใช้หา pod ที่ไม่ Running) |
| `/-f error` | fuzzy filter |
 
---
 
## ทำงานกับ Pod
 
| คีย์ | ทำอะไร |
|---|---|
| `l` | ดู log ของ pod/container ที่เลือก |
| `p` | ดู log ของ container **รอบก่อนหน้า** (= `logs --previous`) สำหรับ CrashLoopBackOff |
| `s` | เปิด shell เข้า container (= `exec -it`) |
| `a` | attach เข้า process หลักของ container |
| `Shift-F` | สร้าง port-forward จาก pod/container นี้ |
| `Shift-Z` | เปิด **sanitize** — โชว์ pod ที่มีปัญหา / ไม่มีเจ้าของ |
 
### ที่ node view
 
| คีย์ | ทำอะไร |
|---|---|
| `k` | cordon node |
| `u` | uncordon node |
| `r` | drain node |
 
---
 
## Log view
 
| คีย์ | ทำอะไร |
|---|---|
| `0` / `1` / `2` / `3` / `4` / `5` | เลือกช่วงเวลา log: all / 1m / 5m / 15m / 30m / 1h |
| `f` | full screen (ซ่อนส่วนอื่นของ UI) |
| `w` | toggle line wrap |
| `s` | toggle autoscroll (หยุดไหลเพื่ออ่านย้อน) |
| `t` | toggle timestamp |
| `/text` | ค้นในกอง log (`n` ถัดไป, `Shift-N` ก่อนหน้า) |
| `Ctrl-S` | บันทึก log ที่เห็นลงไฟล์ |
| `Shift-C` | สลับเรียงเก่า→ใหม่ / ใหม่→เก่า |
| `Ctrl-A` | ดู log ของทุก container ใน pod รวมกัน |
 
---
 
## เครื่องมือวินิจฉัย
 
| คำสั่ง | ทำอะไร |
|---|---|
| `:pulses` (`:pu`) | **Pulses** — dashboard สุขภาพ cluster (deployment / pod / event / node ผ่านไม่ผ่าน) ดูก่อนเริ่มงานทุกเช้า |
| `:popeye` | **Popeye** — สแกนหา misconfiguration: ไม่ได้ตั้ง requests/limits, ไม่มี probe, image ใช้ tag `latest`, secret ที่ไม่มีใครใช้ |
| `:xray deploy` | **XRay** — แผนผังต้นไม้ของความสัมพันธ์ (deploy → rs → pod → container) เห็น dependency ทีเดียว |
| `:xray svc all` | ไล่ดูว่า service ผูกกับ pod ตัวไหนจริง ๆ (ตัวช่วยเวลา endpoint ว่าง) |
| `:bench` | ดูผล benchmark — กด `Ctrl-B` ที่ service/pod เพื่อยิง HTTP load (ผ่าน hey) |
| `:ev` | ดู event ทั้ง cluster เรียงตามเวลา |
| `:dir ~/manifests` | เปิดโฟลเดอร์ manifest แล้ว apply จาก k9s ได้ |
| `:screendump` | เปิดดูไฟล์ dump ที่เคย save ไว้ |
 
---
 
## Config & Skin
 
| Path | ใช้ทำอะไร |
|---|---|
| `~/.config/k9s/config.yaml` | ตั้งค่าหลัก: `refreshRate`, `readOnly`, `logger.tail`, `maxConnRetry`, default namespace/view (ถ้าตั้ง `K9S_CONFIG_DIR` ก็ใช้ path นั้น) |
| `~/.config/k9s/skins/<name>.yaml` | ธีมสี — ระบุชื่อ skin ใน config เพื่อใช้ **สีต่างกันต่อ cluster** (แดง = prod) กันพลาด |
| `~/.config/k9s/aliases.yaml` | ตั้ง alias เอง เช่น `dep: apps/v1/deployments` แล้วพิมพ์ `:dep` |
| `~/.config/k9s/hotkeys.yaml` | ผูกคีย์ลัดไป view ที่ใช้บ่อย (เช่น `Shift-1` → pods ใน namespace ของทีม) |
| `~/.config/k9s/plugins.yaml` | เพิ่มคำสั่งของตัวเองเข้า UI (เช่น กดปุ่มเดียวเพื่อ `rollout restart`) |
| `K9S_LOGS_DIR` / `k9s info` | หา log ของ k9s เองตอน UI มีปัญหา |
 
ตัวอย่าง `config.yaml` แบบปลอดภัยสำหรับ prod:
 
```yaml
k9s:
  refreshRate: 5
  maxConnRetry: 5
  readOnly: true
  noExitOnCtrlC: true
  ui:
    skin: prod-red
    logoless: true
  logger:
    tail: 200
    buffer: 5000
    sinceSeconds: 300
```
 
ตัวอย่าง `hotkeys.yaml`:
 
```yaml
hotKeys:
  shift-1:
    shortCut: Shift-1
    description: Payment pods
    command: pods payment
  shift-2:
    shortCut: Shift-2
    description: Nodes
    command: nodes
```
 
> :bulb: **ตั้งครั้งเดียวคุ้ม**
> ทำ skin สีแดงให้ context ของ prod + เปิด `--readonly` เวลาเข้าดู prod
> บน EKS ให้รัน `aws eks update-kubeconfig --alias prod` ก่อน แล้วสลับใน k9s ด้วย `:ctx`
 
---
 
[← กลับหน้าสารบัญ](./README.md)
