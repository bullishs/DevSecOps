# AWS CLI v2 Cheat Sheet
 
คำสั่งที่ใช้จริงในงาน operate + คำสั่งที่ช่วยให้เห็นภาพ service ตอนอ่านสอบ SAA-C03
 
## สารบัญ
 
- [Config & Identity](#config--identity)
- [Output & --query (JMESPath)](#output--query-jmespath)
- [EC2](#ec2)
- [S3](#s3)
- [IAM & Organizations](#iam--organizations)
- [VPC & Networking](#vpc--networking)
- [EKS / ECR / Container](#eks--ecr--container)
- [RDS & Database](#rds--database)
- [CloudWatch, Logs & Cost](#cloudwatch-logs--cost)
- [SSM, Secrets & Serverless](#ssm-secrets--serverless)
- [สูตรที่ใช้บ่อย](#สูตรที่ใช้บ่อย)
---
 
## Config & Identity
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws configure` | ตั้งค่า access key / region / output แบบ interactive เก็บที่ `~/.aws/credentials` และ `~/.aws/config` |
| `aws configure list` | ดูว่าตอนนี้ค่าแต่ละตัวมาจากไหน (env / profile / IAM role) — ใช้ debug เวลาสงสัยว่าใช้ credential ผิดตัว |
| `aws configure sso` | ตั้งค่า profile แบบ **IAM Identity Center** (SSO) — วิธีที่แนะนำสำหรับ multi-account |
| `aws sso login --profile prod` | เปิด browser ขอ token ใหม่ให้ profile นั้น (หมดอายุแล้วต้อง login ซ้ำ) |
| `aws sts get-caller-identity` | **คำสั่งแรกที่ควรพิมพ์เสมอ** — ตอบว่าตอนนี้เราเป็นใคร account ไหน role อะไร |
| `aws configure get region --profile prod` | อ่านค่าเดี่ยว ๆ จาก profile (ใช้ใน shell script ได้) |
| `export AWS_PROFILE=prod AWS_REGION=ap-southeast-1` | ตั้ง profile/region ทั้ง session แทนการพิมพ์ `--profile` ทุกคำสั่ง |
| `aws ec2 describe-regions --query 'Regions[].RegionName' --output text` | ลิสต์ region ทั้งหมดที่ account เปิดใช้ |
| `aws ec2 describe-availability-zones --region ap-southeast-1` | ดู AZ ใน region — ชื่อ AZ (`ap-southeast-1a`) map คนละ physical AZ ในแต่ละ account |
 
> :bulb: **credential precedence**: CLI flag → environment variable → profile ใน `~/.aws` → container / EC2 instance role
> ถ้าโค้ดรันบน EC2/EKS ให้ใช้ **IAM Role** อย่าฝัง access key (ข้อสอบชอบถาม)
 
---
 
## Output & --query (JMESPath)
 
| คำสั่ง / flag | คำอธิบาย |
|---|---|
| `--output table \| json \| text \| yaml` | เลือกรูปแบบผลลัพธ์ — `table` อ่านคนง่าย, `text` เอาไปต่อ awk/xargs ได้ |
| `--no-cli-pager` | ปิด pager (ไม่ต้องกด q ออก) ตั้งถาวรได้ด้วย `export AWS_PAGER=""` |
| `--query 'Reservations[].Instances[].InstanceId'` | คัดเฉพาะ field ที่อยากได้ — กรองที่ฝั่ง client ด้วย JMESPath |
| ``--query 'Instances[?State.Name==`running`]'`` | filter ด้วยเงื่อนไข — สังเกตว่าใช้ backtick ครอบค่าที่เทียบ |
| `--query 'Instances[].[InstanceId,InstanceType,PrivateIpAddress]' --output table` | เลือกหลาย column ออกมาเป็นตาราง |
| `--query 'sort_by(Volumes,&Size)[-3:].[VolumeId,Size]'` | เรียงแล้วเอา 3 ตัวท้าย — JMESPath มี `sort_by` / `length` / `max_by` / `join` ให้ใช้ |
| `--filters "Name=tag:Env,Values=prod"` | filter ที่ฝั่ง **server** (เร็วกว่า `--query` เพราะกรองก่อนส่งกลับ) |
| `--max-items 20 --page-size 20` | คุมจำนวนผลลัพธ์ / ขนาดหน้า เวลา resource เยอะมาก |
| `--dry-run` | ลองยิงดูว่า **สิทธิ์พอไหม** โดยไม่สร้างของจริง (EC2 API) — เจอ `DryRunOperation` = ผ่าน |
| `aws ec2 describe-instances --generate-cli-skeleton input > in.json` | สร้างโครง JSON input เปล่า ๆ แล้วเรียกด้วย `--cli-input-json file://in.json` |
| `aws ec2 wait instance-running --instance-ids i-abc` | บล็อกรอจนสถานะเปลี่ยน — ใช้ใน automation script แทน `sleep` |
 
---
 
## EC2
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws ec2 describe-instances` | ดู instance ทั้งหมด (ดู [สูตรที่ใช้บ่อย](#สูตรที่ใช้บ่อย) สำหรับ `--query` แบบสรุปตาราง) |
| `aws ec2 start-instances --instance-ids i-abc i-def` | เปิดเครื่อง — `stop` / `reboot` / `terminate-instances` ใช้รูปแบบเดียวกัน |
| `aws ec2 describe-instance-status --include-all-instances` | ดู system status check / instance status check (สำหรับ CloudWatch alarm recover) |
| `aws ec2 create-tags --resources i-abc --tags Key=Env,Value=prod` | ติด tag — พื้นฐานของ cost allocation และ ABAC |
| `aws ec2 describe-security-groups --group-ids sg-abc` | ดู inbound/outbound rule ของ SG |
| `aws ec2 authorize-security-group-ingress --group-id sg-abc --protocol tcp --port 443 --cidr 0.0.0.0/0` | เปิด port ใน SG (`revoke-security-group-ingress` = ปิด) |
| `aws ec2 create-snapshot --volume-id vol-abc --description "before patch"` | snapshot EBS → เก็บใน S3 (incremental) ข้าม AZ/region ได้ด้วย `copy-snapshot` |
| `aws ec2 create-image --instance-id i-abc --name golden-2026 --no-reboot` | สร้าง AMI จาก instance ที่รันอยู่ — ใช้ทำ golden image ให้ ASG |
| `aws ec2 describe-spot-price-history --instance-types t3.medium --product-descriptions "Linux/UNIX" --max-items 5` | เช็คราคา spot ก่อนตัดสินใจใช้ Spot Instance |
| `aws ec2 modify-instance-attribute --instance-id i-abc --instance-type t3.large` | เปลี่ยนขนาดเครื่อง (**ต้อง stop ก่อน**) = vertical scaling |
 
---
 
## S3
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws s3 ls s3://bucket/prefix/ --human-readable --summarize` | ลิสต์ object พร้อมขนาดรวม |
| `aws s3 cp file.zip s3://bucket/path/ --storage-class STANDARD_IA` | อัปโหลดพร้อมเลือก storage class (CLI ทำ multipart ให้อัตโนมัติถ้าไฟล์ใหญ่) |
| `aws s3 sync ./dist s3://bucket --delete --exclude "*.map"` | sync โฟลเดอร์ขึ้น bucket ลบของที่ไม่มีต้นทางออก — คำสั่ง deploy static site |
| `aws s3 rm s3://bucket/path --recursive --dryrun` | ลบทั้ง prefix — ใส่ `--dryrun` ดูก่อนเสมอ |
| `aws s3 presign s3://bucket/key --expires-in 3600` | สร้าง **pre-signed URL** ให้คนนอกดาวน์โหลดชั่วคราวโดยไม่ต้องเปิด bucket เป็น public |
| `aws s3api put-bucket-versioning --bucket b --versioning-configuration Status=Enabled` | เปิด versioning — เงื่อนไขก่อนใช้ MFA Delete / Replication / กัน overwrite |
| `aws s3api put-public-access-block --bucket b --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true` | ปิดทางเป็น public ทั้งหมด (ค่า default ของ bucket ใหม่) |
| `aws s3api put-bucket-lifecycle-configuration --bucket b --lifecycle-configuration file://lc.json` | ตั้ง lifecycle ย้ายไป IA / Glacier / ลบทิ้งตามอายุ = ตัวลดค่าใช้จ่ายหลัก |
| `aws s3api list-object-versions --bucket b --prefix p/` | ดูทุก version + delete marker (ตอนกู้ไฟล์ที่ถูกลบ) |
| `aws s3api restore-object --bucket b --key k --restore-request Days=3,GlacierJobParameters={Tier=Expedited}` | ขอ restore object จาก Glacier — tier: Expedited (1–5 นาที) / Standard (3–5 ชม.) / Bulk (5–12 ชม.) |
| `aws s3 cp s3://a s3://b --recursive --source-region us-east-1` | คัดลอกข้าม bucket/region โดยไม่ผ่านเครื่องเรา |
 
บังคับ SSE-KMS เป็น default ของ bucket:
 
```bash
aws s3api put-bucket-encryption --bucket <bucket> \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"}}]}'
```
 
---
 
## IAM & Organizations
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws iam list-attached-role-policies --role-name MyRole` | ดู managed policy ที่แปะกับ role |
| `aws iam list-role-policies --role-name MyRole` | ดู inline policy ของ role (คนละชุดกับ managed) |
| `aws iam get-role --role-name MyRole --query 'Role.AssumeRolePolicyDocument'` | อ่าน **trust policy** — ใครสวม role นี้ได้ (คนละอันกับ permission policy) |
| `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names s3:PutObject --resource-arns <arn>` | จำลองว่า policy จะ allow หรือ deny ก่อนลองของจริง |
| `aws iam get-account-password-policy` | เช็ค password policy ของ account |
| `aws iam create-service-linked-role --aws-service-name eks.amazonaws.com` | สร้าง service-linked role ที่ AWS service ต้องใช้ |
| `aws organizations list-accounts --query 'Accounts[].[Id,Name,Status]' --output table` | ลิสต์ account ใน Organization |
| `aws organizations list-policies --filter SERVICE_CONTROL_POLICY` | ดู SCP ทั้งหมด — SCP เป็น **เพดานสิทธิ์** ไม่ได้ให้สิทธิ์เอง |
 
สวมบทบาทข้าม account:
 
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<account-id>:role/<role-name> \
  --role-session-name boyd
```
 
รายงาน credential ทุก user (key เก่า / ไม่เปิด MFA) สำหรับงาน audit:
 
```bash
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 -d
```
 
---
 
## VPC & Networking
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws ec2 describe-vpcs --query 'Vpcs[].[VpcId,CidrBlock,IsDefault]' --output table` | ดู VPC + CIDR |
| `aws ec2 describe-subnets --query 'Subnets[].[SubnetId,AvailabilityZone,CidrBlock,MapPublicIpOnLaunch]' --output table` | ดู subnet เรียงตาม AZ — `MapPublicIpOnLaunch=true` คือ public subnet |
| `aws ec2 describe-route-tables --query 'RouteTables[].{RT:RouteTableId,R:Routes[].GatewayId}'` | ตรวจ route — เจอ `igw-` = public, เจอ `nat-` = private |
| `aws ec2 describe-nat-gateways --filter Name=state,Values=available` | NAT Gateway ที่รันอยู่ (ตัวกินเงินเงียบ ๆ — 1 ตัวต่อ AZ ถ้าต้องการ HA) |
| `aws ec2 describe-vpc-endpoints --query 'VpcEndpoints[].[ServiceName,VpcEndpointType]' --output table` | VPC Endpoint: **Gateway** (S3, DynamoDB — ฟรี) vs **Interface/PrivateLink** (ENI, คิดเงิน) |
| `aws ec2 describe-vpc-peering-connections` | peering ทั้งหมด — จำว่า peering **ไม่ transitive** และ CIDR ห้ามซ้อนกัน |
| `aws ec2 describe-network-acls --query 'NetworkAcls[].Entries'` | NACL เป็น **stateless** ต้องเปิด ephemeral port ขากลับ (1024–65535) เองด้วย |
| `aws ec2 describe-transit-gateways` | Transit Gateway = hub เชื่อม VPC จำนวนมาก (แก้ปัญหา peering mesh) |
| `aws route53 list-hosted-zones` | ดู hosted zone ทั้งหมด |
| `aws route53 list-resource-record-sets --hosted-zone-id <zone-id>` | ดู record ทั้งหมดใน zone |
 
เปิด VPC Flow Logs เก็บเฉพาะ REJECT (ตอนไล่ปัญหา SG/NACL):
 
```bash
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids <vpc-id> \
  --traffic-type REJECT \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow \
  --deliver-logs-permission-arn arn:aws:iam::<account-id>:role/FlowLogs
```
 
---
 
## EKS / ECR / Container
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws eks update-kubeconfig --name <cluster> --region ap-southeast-1 --alias prod` | **คำสั่งที่ใช้บ่อยที่สุด** — เขียน context ลง `~/.kube/config` ให้ kubectl/k9s ใช้ได้เลย |
| `aws eks list-clusters` | ดู cluster ทั้งหมด |
| `aws eks describe-cluster --name <c> --query 'cluster.{V:version,EP:endpoint,St:status}'` | ดู version / endpoint / สถานะ |
| `aws eks list-nodegroups --cluster-name <c>` | ดู managed node group |
| `aws eks describe-nodegroup --cluster-name <c> --nodegroup-name <ng>` | ดู scaling config / instance type ของ node group |
| `aws eks update-nodegroup-config --cluster-name <c> --nodegroup-name <ng> --scaling-config minSize=2,maxSize=6,desiredSize=3` | ปรับจำนวน node |
| `aws eks list-addons --cluster-name <c>` | ดู add-on (vpc-cni, coredns, kube-proxy, ebs-csi-driver) |
| `aws ecr describe-images --repository-name app --query 'sort_by(imageDetails,&imagePushedAt)[-5:].[imageTags[0],imagePushedAt]' --output table` | ดู 5 image ล่าสุดที่ push |
| `aws ecr start-image-scan --repository-name app --image-id imageTag=latest` | สั่ง scan ช่องโหว่ image (ต่อกับ Inspector ได้) |
| `aws ecs update-service --cluster <c> --service <s> --force-new-deployment` | deploy ECS ใหม่โดยไม่เปลี่ยน task definition |
 
login ECR ก่อน push image:
 
```bash
aws ecr get-login-password --region ap-southeast-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com
```
 
---
 
## RDS & Database
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,Engine,DBInstanceClass,MultiAZ,DBInstanceStatus]' --output table` | สรุป instance ทั้งหมด + สถานะ Multi-AZ |
| `aws rds create-db-snapshot --db-instance-identifier db1 --db-snapshot-identifier db1-manual` | manual snapshot (ไม่หายเองเหมือน automated backup) |
| `aws rds create-db-instance-read-replica --db-instance-identifier r1 --source-db-instance-identifier db1` | Read Replica = กระจาย read (async) ≠ Multi-AZ ที่ทำเพื่อ HA (sync) |
| `aws rds failover-db-cluster --db-cluster-identifier aurora1` | ทดสอบ failover Aurora |
| `aws dynamodb describe-table --table-name t --query 'Table.{Keys:KeySchema,Mode:BillingModeSummary}'` | ดู partition/sort key และ billing mode (On-demand vs Provisioned) |
| `aws elasticache describe-cache-clusters --show-cache-node-info` | ดู ElastiCache node (Redis สำหรับ session/leaderboard, Memcached สำหรับ simple cache) |
 
Point-in-time recovery — กู้ย้อนเวลาจาก automated backup (สร้าง instance ใหม่เสมอ):
 
```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier db1 \
  --target-db-instance-identifier db1-pitr \
  --restore-time 2026-09-20T10:00:00Z
```
 
query DynamoDB ด้วย partition key (ถูกและเร็วกว่า scan มาก):
 
```bash
aws dynamodb query --table-name t \
  --key-condition-expression "pk = :p" \
  --expression-attribute-values '{":p":{"S":"USER#1"}}'
```
 
---
 
## CloudWatch, Logs & Cost
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws logs tail /aws/lambda/fn --follow --since 10m --format short` | **tail log แบบ realtime** — ตัวช่วยที่ดีที่สุดใน CLI v2 |
| `aws logs filter-log-events --log-group-name /app --filter-pattern "ERROR"` | ค้น log ย้อนหลังตาม pattern |
| `aws logs put-retention-policy --log-group-name /app --retention-in-days 30` | ตั้งอายุ log — default คือเก็บตลอดไป (= เสียเงินเรื่อย ๆ) |
| `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances` | CloudTrail = **ใครทำอะไร** (audit API call) ต่างจาก CloudWatch ที่ดู metric/log |
| `aws support describe-trusted-advisor-checks --language en` | Trusted Advisor (ต้อง Business/Enterprise support ขึ้นไป) |
 
CloudWatch Logs Insights — query ภาษา SQL-ish (ตามด้วย `get-query-results`):
 
```bash
aws logs start-query --log-group-name /app \
  --query-string 'fields @timestamp,@message | filter @message like /5\d\d/ | sort @timestamp desc | limit 50' \
  --start-time <epoch-ms> --end-time <epoch-ms>
```
 
ดึง metric ย้อนหลัง:
 
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --start-time 2026-09-21T00:00:00Z --end-time 2026-09-22T00:00:00Z \
  --period 300 --statistics Average
```
 
สร้าง alarm → ยิง SNS / trigger ASG / EC2 recover:
 
```bash
aws cloudwatch put-metric-alarm --alarm-name cpu-high \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 --period 300 --statistics Average \
  --alarm-actions arn:aws:sns:<region>:<account-id>:<topic>
```
 
ดูค่าใช้จ่ายแยกตาม service:
 
```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-09-01,End=2026-09-30 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```
 
---
 
## SSM, Secrets & Serverless
 
| คำสั่ง | คำอธิบาย |
|---|---|
| `aws ssm start-session --target i-abc` | **SSH เข้าเครื่องแบบไม่ต้องเปิด port 22 / ไม่ต้องมี bastion** — คำตอบข้อสอบเรื่อง secure access |
| `aws ssm send-command --document-name AWS-RunShellScript --instance-ids i-abc --parameters 'commands=["yum -y update"]'` | สั่งรันคำสั่งพร้อมกันหลายเครื่อง (patching) |
| `aws ssm get-parameter --name /app/db/url --with-decryption --query Parameter.Value --output text` | Parameter Store — เก็บ config/secret (ฟรีสำหรับ standard tier) |
| `aws ssm put-parameter --name /app/key --value xxx --type SecureString --overwrite` | เขียนค่าแบบเข้ารหัสด้วย KMS |
| `aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text` | Secrets Manager — จุดต่างคือ **rotate อัตโนมัติ** ได้ (แลกกับค่าใช้จ่าย) |
| `aws kms encrypt --key-id alias/app --plaintext fileb://f.txt --output text --query CiphertextBlob` | เข้ารหัสด้วย KMS CMK |
| `aws lambda invoke --function-name fn --payload '{"k":"v"}' --cli-binary-format raw-in-base64-out out.json` | เรียก Lambda จาก CLI (v2 ต้องใส่ `--cli-binary-format`) |
| `aws lambda update-function-configuration --function-name fn --memory-size 1024 --timeout 30` | ปรับ memory (มาก = CPU มากขึ้นด้วย) และ timeout สูงสุด 15 นาที |
| `aws sqs get-queue-attributes --queue-url <url> --attribute-names All` | ดูความยาวคิว / visibility timeout / DLQ config |
| `aws sns publish --topic-arn <arn> --message "deploy done"` | ส่ง notification ทดสอบ fan-out |
| `aws cloudformation deploy --template-file t.yaml --stack-name s --capabilities CAPABILITY_IAM` | deploy IaC — ใช้ `describe-stack-events` เวลา rollback เพื่อดูว่าพังที่ resource ไหน |
 
> :warning: **กับดักข้อสอบ**
> - "เข้า EC2 ใน private subnet โดยไม่เปิด inbound port และไม่มี bastion" → **SSM Session Manager**
> - "หมุนรหัสฐานข้อมูลอัตโนมัติ" → **Secrets Manager** ไม่ใช่ Parameter Store
 
---
 
## สูตรที่ใช้บ่อย
 
สรุป EC2 instance ทั้งหมดเป็นตาราง:
 
```bash
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].{ID:InstanceId,T:InstanceType,S:State.Name,IP:PrivateIpAddress}' \
  --output table
```
 
หา instance ที่ยังรันอยู่และมี tag `Env=prod`:
 
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Env,Values=prod" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]' \
  --output text
```
 
ตรวจว่า node ของ EKS ใช้ AMI/version อะไรอยู่:
 
```bash
aws eks describe-nodegroup --cluster-name <c> --nodegroup-name <ng> \
  --query 'nodegroup.{AMI:amiType,Ver:version,Rel:releaseVersion,Scale:scalingConfig}'
```
 
หา log group ที่ยังไม่ตั้ง retention (ตัวกินเงิน):
 
```bash
aws logs describe-log-groups \
  --query 'logGroups[?!not_null(retentionInDays)].logGroupName' \
  --output text
```
 
---
 
[← กลับหน้าสารบัญ](./README.md)
