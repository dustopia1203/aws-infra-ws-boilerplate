<!-- 
  HƯỚNG DẪN SỬ DỤNG TEMPLATE NÀY:
  1. Copy file này thành sumary-terraform.md
  2. Cung cấp file terraform-plan output cho AI
  3. Yêu cầu AI: "Đọc file terraform-plan và điền vào file sumary-terraform.md theo format mẫu"
  4. AI sẽ phân tích terraform plan và điền nội dung chi tiết vào từng section
  
  QUY ƯỚC:
  - [Thuộc phạm vi release]: Thay đổi liên quan trực tiếp đến release
  - [Không thuộc phạm vi release nhưng không ảnh hưởng]: Thay đổi tự động/phụ thuộc, không ảnh hưởng hệ thống
  - Nội dung "known after apply" cần ghi dự kiến thay đổi + note kiểm tra khi release

  QUAN TRỌNG - CHI TIẾT ATTRIBUTE-LEVEL:
  - Mỗi section PHẢI có phần "Chi tiết thay đổi" ghi rõ attribute nào thay đổi, giá trị cũ → mới
  - Không được viết chung chung như "cập nhật configuration" hay "thay đổi expression mới"
  - Phải ghi cụ thể: expression cũ là gì, mới là gì; threshold bao nhiêu; period bao nhiêu
  - Nếu nhiều resource cùng pattern, show chi tiết 1 resource rồi ghi "Tương tự cho N resource còn lại"
-->

## Terraform Plan & Pull Request

### {TÊN_PROJECT} – {REGION}
- **Pull Request**: [{PR_URL}]({PR_URL})
- **Terraform Plan**: [{TÊN_FILE_PLAN}]

<!-- Thêm các project/environment khác nếu có -->

---

## Tổng quan thay đổi Terraform plan

> ⚠️ **Dưới đây là tổng hợp thay đổi terraform. Có một số thay đổi không thuộc phạm vi release nhưng không ảnh hưởng hệ thống chúng tôi đã ghi chú phía dưới**

## {TÊN_PROJECT} – {REGION}
- **Thêm mới (Add):** {X} resources
- **Thay đổi (Change):** {Y} resources
- **Xóa (Destroy):** {Z} resources

<!-- Nếu có destroy do recreate, thêm note giải thích -->
> Lưu ý: {N} resources bị destroy là do {LÝ_DO, vd: ECS Task Definitions phải tạo lại khi thay đổi container_definitions}. Thực tế không mất service.

---

## Chi tiết thay đổi

<!-- ============================================================ -->
<!-- MẪU 1: Thay đổi AMI / Launch Template (thường không thuộc release) -->
<!-- ============================================================ -->

### 1. Cập nhật AMI cho Launch Templates – {N} resources

Cập nhật AMI mới cho các Auto Scaling Group:

| Resource | Tên | AMI cũ | AMI mới |
|---|---|---|---|
| `{resource_address}` | {tên_launch_template} | {ami_cũ} | **{ami_mới}** |

➡️ [Không thuộc phạm vi release nhưng không ảnh hưởng]

Do AMI ECS optimize ra version mới nên terraform tự động update. Khi deploy chỉ áp dụng với instance mới, không ảnh hưởng instance đang hoạt động.

---

<!-- ============================================================ -->
<!-- MẪU 2: Thay đổi Tags (thường không thuộc release) -->
<!-- ============================================================ -->

### 2. Cập nhật Tags – {N} resources

Bổ sung tag `{Key} = "{Value}"`:
- `{resource_address_1}` ({mô_tả})
- `{resource_address_2}` ({mô_tả})

➡️ [Không thuộc phạm vi release nhưng không ảnh hưởng] Terraform thêm tag.

---

<!-- ============================================================ -->
<!-- MẪU 3: ECS Task Definition recreate (thường thuộc release) -->
<!-- ============================================================ -->

### 3. Cập nhật ECS Task Definitions – {N} resources (destroy and recreate)

Các ECS Task Definition được tạo lại do {LÝ_DO, vd: thêm secret/env mới}:

| # | Resource | Task Definition | Thay đổi |
|---|---|---|---|
| 1 | `{resource_address}` | {task_definition_name} | Thêm secret `{SECRET_NAME}` |

**Chi tiết thay đổi:**
- Container `{container_name}`: thêm secret `{SECRET_NAME}` với valueFrom = `{arn_secret}`
- Image: `{old_image}` → `{new_image}` _(nếu có thay đổi)_
- Environment: thêm `{ENV_KEY} = "{ENV_VALUE}"` _(nếu có)_

➡️ [Thuộc phạm vi release] {Mô_tả_ngắn_gọn}.

---

<!-- ============================================================ -->
<!-- MẪU 4: ECS Service update (thường đi kèm Task Definition) -->
<!-- ============================================================ -->

### 4. Cập nhật ECS Services – {N} resources

| Resource | Service Name | Thay đổi |
|---|---|---|
| `{resource_address}` | {service_name} | `task_definition` → revision mới |

➡️ [Thuộc phạm vi release] ECS Service cập nhật để sử dụng Task Definition revision mới.

---

<!-- ============================================================ -->
<!-- MẪU 5: Tạo mới SQS Queue -->
<!-- ============================================================ -->

### 5. Tạo mới SQS Queue – {N} resources (add)

Tạo mới SQS queue và các resource liên quan:

| # | Resource | Tên |
|---|---|---|
| 1 | `{module}.aws_sqs_queue.queue` | {queue_name} |
| 2 | `{module}.aws_sqs_queue.dead_queue` | {queue_name}-dead |
| 3 | `{module}.aws_sqs_queue_policy.queue` | Queue policy |
| 4 | `{module}.aws_sqs_queue_policy.dead_queue` | Dead queue policy |
| 5 | `{module}.aws_cloudwatch_metric_alarm.dead_queue[0]` | Dead queue alarm |

Cấu hình queue:
- `visibility_timeout_seconds`: {value}
- `message_retention_seconds`: {value}
- `sqs_managed_sse_enabled`: {true/false}

➡️ [Thuộc phạm vi release] {Mô_tả_mục_đích_queue}.

---

<!-- ============================================================ -->
<!-- MẪU 6: Cập nhật SQS Visibility Timeout -->
<!-- ============================================================ -->

### 6. Cập nhật SQS Visibility Timeout – {N} resources

| Resource | Queue Name | Timeout cũ | Timeout mới |
|---|---|---|---|
| `{resource_address}` | {queue_name} | {old}s | **{new}s** |

**Lý do thay đổi:** {Giải thích tại sao cần tăng/giảm timeout, vd: worker cần thêm thời gian xử lý message lớn}

➡️ [Thuộc phạm vi release] {Lý_do_thay_đổi}.

---

<!-- ============================================================ -->
<!-- MẪU 7: Cập nhật IAM Policy -->
<!-- ============================================================ -->

### 7. Cập nhật IAM Policy – {N} resources

**7.1.** `{resource_address}` ({policy_name})

**Policy cũ:**
```json
{
  "Effect": "Allow",
  "Action": ["{old_actions}"],
  "Resource": ["{old_resource_arns}"]
}
```

**Policy mới (thay đổi):**
```json
{
  "Effect": "Allow",
  "Action": ["{new_actions}"],
  "Resource": ["{new_resource_arns}"]
}
```

**Khác biệt:** Thêm action `{new_action}` / Thêm resource `{new_resource_arn}` / Xóa `{removed_item}`

> 📝 **Lưu ý:** Nếu nội dung policy là "known after apply", ghi rõ giá trị dự kiến dựa trên code diff và note cần kiểm tra khi release.

➡️ [Thuộc phạm vi release] {Mô_tả_ngắn_gọn}.

---

<!-- ============================================================ -->
<!-- MẪU 8: Cập nhật S3 Bucket ACL -->
<!-- ============================================================ -->

### 8. Cập nhật S3 Bucket ACL – {N} resources

`{resource_address}` (bucket: {bucket_name})
- Thêm grant {PERMISSION} cho {GRANTEE_TYPE}: `{grantee_id}`

➡️ [{Phạm_vi}] {Mô_tả_ngắn_gọn}.

---

<!-- ============================================================ -->
<!-- MẪU 9: Cập nhật CloudWatch Dashboard -->
<!-- ============================================================ -->

### 9. Cập nhật CloudWatch Dashboards – {N} resources

**9.1.** `{resource_address}` (Dashboard: {dashboard_name})

{Mô_tả_thay_đổi, vd: Thêm queue/service mới vào các widget monitoring}.

➡️ [{Phạm_vi}] {Mô_tả_ngắn_gọn}.

---

<!-- ============================================================ -->
<!-- MẪU 10: Thay đổi do AWS Provider version (thường không thuộc release) -->
<!-- ============================================================ -->

### 10. Thay đổi do AWS Provider – {N} resources

- `{resource_address_1}` ({mô_tả})
- `{resource_address_2}` ({mô_tả})

Thêm attribute `{attribute_name}` (known after apply).

➡️ [Không thuộc phạm vi release nhưng không ảnh hưởng] Đây là thay đổi do AWS provider version mới, không ảnh hưởng hoạt động.
