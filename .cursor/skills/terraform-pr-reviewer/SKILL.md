---
name: terraform-pr-reviewer
description: >
  Review Terraform PR code changes and terraform plan output. Cross-check git diff against plan,
  review code quality, security, and impact assessment, then generate a structured summary report
  in Vietnamese and Japanese. Use this skill whenever the user mentions reviewing a terraform PR,
  checking terraform plan output, cross-checking terraform changes, comparing code diff with plan,
  or wants to generate a terraform change summary report. Also trigger when the user provides
  branch names and terraform plan files for review, asks about the impact/risk of terraform
  changes before applying, says things like "check xem plan này có đúng không", "review PR terraform",
  "so sánh code với plan", or wants to know if a terraform apply is safe.
---

# Terraform PR Reviewer

Review Terraform pull requests by cross-checking git diff against terraform plan output, then producing two deliverables:
1. A detailed review report (code quality, security, impact assessment)
2. A structured summary of all terraform plan changes (Vietnamese + Japanese)

This skill exists because reviewing terraform changes manually is error-prone — it's easy to miss discrepancies between what the code says and what terraform actually plans to do. Automating the cross-check catches issues like wrong diff direction, state drift, and unintended destroys before they hit production.

### Core principle: Attribute-level detail

The output files are the ONLY documents that reviewers and the Japanese team will read. They will NOT have access to the raw diff or plan output. So every change must be documented at the **attribute level** — showing the specific values that changed (old → new), the actual configuration of new resources, and concrete code snippets from the diff.

A summary that just says "cập nhật expression mới" or "thay đổi configuration" is nearly useless. The reader needs to see the actual expression, the actual threshold value, the actual scaling adjustment — the specific values that terraform will apply. Think of it this way: if the reader cannot verify correctness just by reading your output, the output is not detailed enough.

---

## Step 1: Collect inputs from the user

### 1a. Auto-detect PR branch

If the user doesn't specify the PR branch, auto-detect it by running:
```bash
git branch --show-current
```
Use the result as the default PR branch. Tell the user: "Tôi detect branch hiện tại là `{branch}`, sẽ dùng làm PR branch nhé."

### 1b. Gather remaining inputs

Check what info the user has already provided. For any missing required fields, reply with a ready-to-copy prompt template that the user can fill in and send back directly. This saves the user from having to compose the message themselves.

**Example reply when info is missing:**

> Tôi cần thêm một số thông tin. Bạn copy đoạn dưới, điền vào rồi gửi lại nhé:
>
> ```
> Target branch:                          ← bắt buộc điền, không có giá trị mặc định
> Terraform folder: terraform/envs/stg
> Plan file path: terraform/envs/stg/terraform-plan-TICKET-123
> Ticket ID: TICKET-123
> Project name:                           ← bắt buộc điền, không có giá trị mặc định
> Region: ap-northeast-1
> PR URL (optional):
> Output directory (để trống = dùng folder hiện tại: terraform/envs/stg):
> ```

**Pre-fill rules — follow these strictly:**
- `Target branch`: ALWAYS leave blank. Never pre-fill with `develop`, `main`, or any guess. The user must explicitly provide this.
- `Project name`: ALWAYS leave blank. Never guess from folder names or branch names.
- `Terraform folder`: pre-fill if inferable from context (e.g., open files in editor)
- `Plan file path`: pre-fill if a plan file is open in the editor
- `Ticket ID`: extract from branch name if pattern matches (e.g., `feature/TOFAS-14187-xxx` → `TOFAS-14187`)
- `Region`: pre-fill only if explicitly visible in open files or prior conversation — otherwise leave blank
- `Output directory`: always show this line with the inferred default in parentheses so the user can override if needed

### Required inputs

| Input | Example | Required | Auto-detect |
|---|---|---|---|
| PR branch (source) | `feature/TICKET-123-update-scaling` | Yes | Current git branch |
| Target branch (base) | `develop` | Yes | No |
| Terraform folder to diff | `terraform/envs/stg` | Yes | No |
| Terraform plan output file path | `terraform/envs/stg/terraform-plan-TICKET-123` | Yes | No |
| Ticket ID (for output filenames) | `TICKET-123` | Yes | Extract from branch name |
| Project name | `MyProject STG` | Yes | No |
| Region | `ap-northeast-1` | Yes | No |
| PR URL (optional) | `https://github.com/org/repo/pull/42` | No | No |
| Output directory (optional) | `terraform/envs/stg` | No | Same folder as plan file |

If the user provides all info in one message, skip the interview and proceed to Step 1c.

### 1c. Confirm and remind to pull latest code

Once all required inputs are collected, show a confirmation summary and remind the user to pull latest code before proceeding. Do NOT run git fetch/pull yourself — the user must do this manually.

**Example reply:**

> **Thông tin review:**
> - PR branch: `feature/TICKET-123-update-scaling`
> - Target branch: `develop`
> - Folder: `terraform/envs/stg`
> - Plan file: `terraform/envs/stg/terraform-plan-TICKET-123`
>
> ⚠️ **Trước khi bắt đầu**, hãy đảm bảo code ở cả 2 branch đã được pull mới nhất:
> ```bash
> git fetch origin
> git checkout develop && git pull origin develop
> git checkout feature/TICKET-123-update-scaling && git pull origin feature/TICKET-123-update-scaling
> ```
> Confirm OK để tôi bắt đầu review nhé.

Wait for the user to confirm before proceeding to Step 2. This prevents analyzing stale code that doesn't match the plan output.

---

## Step 2: Gather data

**Run 2a and 2b in parallel** — git diff and reading the plan file are independent operations. Use parallel tool calls to execute both at the same time. This saves time and keeps context cleaner.

### 2a. Git diff

Run the diff from target → source (this shows what the PR adds):

```bash
git diff {TARGET_BRANCH} {PR_BRANCH} -- {TERRAFORM_FOLDER}
```

The direction matters: `git diff develop feature/xxx` shows what the feature branch introduces compared to develop. Getting this backwards will invert the entire analysis — you'd see additions as deletions and vice versa.

**If the diff command fails** (branch not found, etc.): Do NOT run `git fetch` or `git pull` yourself. Instead, tell the user:
> Branch `{BRANCH_NAME}` không tìm thấy. Bạn có thể chạy lệnh sau để update rồi thử lại:
> ```bash
> git fetch origin
> git checkout {BRANCH_NAME}
> ```

**If the diff output is very large** (>500 lines): First run `git diff --stat` to get an overview of which files changed and how many lines. Then diff individual files one at a time to avoid truncation:
```bash
git diff {TARGET} {SOURCE} -- {FOLDER}/specific_file.tf
```

### 2b. Read terraform plan

Read the terraform plan output file the user provided. Parse the summary line at the bottom:
```
Plan: X to add, Y to change, Z to destroy.
```

Identify every resource action:
- `+` create
- `~` update in-place
- `-` destroy
- `-/+` destroy and recreate

---

## Step 3 & Step 4: Cross-check and Code review (run in parallel)

Step 3 (cross-check diff vs plan) and Step 4 (code review) are independent — Step 3 needs both diff + plan data, Step 4 only needs the diff. Run them in parallel using separate subagents or parallel tool calls to save time and reduce context pressure.

When running in parallel:
- Each step writes its findings to memory/notes independently
- Merge the results when writing the output files in Step 5

### Step 3: Cross-check diff vs plan

Compare every change in the git diff against the terraform plan to verify they match. This is the core value of the skill — catching discrepancies before they reach production.

When documenting findings, list every single resource individually. Do not group or summarize with phrases like "16 services tương tự" — each resource must appear as its own row or bullet point.

### How to map .tf code to plan resources

Terraform resource addresses in the plan follow the pattern `{resource_type}.{resource_name}` (or `module.{module_name}.{resource_type}.{resource_name}` for modules). To map from diff to plan:

1. Look at the `resource` blocks in the diff — the type and name form the address
   - `resource "aws_cloudwatch_metric_alarm" "my_alarm"` → `aws_cloudwatch_metric_alarm.my_alarm`
2. For `for_each` resources, the plan shows keys in brackets: `aws_cloudwatch_metric_alarm.my_alarm["key"]`
3. For `count` resources, the plan shows index: `aws_cloudwatch_metric_alarm.my_alarm[0]`
4. Module resources get prefixed: `module.my_module.aws_sqs_queue.queue`

### What to check

- Every resource changed in the diff should appear in the plan with the expected action
- Every resource in the plan should be traceable back to a code change (or identified as drift/external change)
- Resource names, keys, and attributes should match between diff and plan
- `for_each` key changes (e.g., `["scale-in"]` → `["scale_in"]`) cause destroy+create — verify this is reflected
- Comment-out of resources should result in destroy in plan
- New resources in diff should appear as create in plan

### Attribute-level detail requirement

For every resource change, document the **specific attributes** that changed with their old and new values. The reader needs to understand exactly what terraform will do at the attribute level, not just "this resource will be updated". This is the most critical quality factor — a summary that says "cập nhật expression mới" without showing the actual expression is nearly useless.

For **update in-place** resources, show each changed attribute as `attribute: old_value → new_value`. Extract these from the terraform plan output where it shows `~` changes with `->` arrows.

For **create** resources, list the key configuration attributes and their values (thresholds, periods, expressions, etc.). The reader should be able to verify the configuration is correct without looking at the plan file.

For **destroy+create** resources, show both what's being removed and what's being added, with the key differences highlighted.

**Example of BAD attribute documentation:**
> `aws_cloudwatch_metric_alarm.worker_scale_out` — Cập nhật metric_query expression mới

**Example of GOOD attribute documentation:**
> `aws_cloudwatch_metric_alarm.worker_scale_out` — update in-place:
> - `metric_query.e1.expression`: `"FILL(m1, 0)"` → `"IF(m1 > 0, m1, 0)"`
> - `datapoints_to_alarm`: `2` → `1`
> - `evaluation_periods`: `2` → `1`

### Flag discrepancies

- **Resources in plan but not in diff** → likely state drift or changes from other `.tf` files not in the diff scope. Label these clearly so the user knows they'll be applied alongside the PR changes.
- **Resources in diff but not in plan** → the plan may have been generated from a different branch or before the latest commit. Recommend the user regenerate the plan.
- **Action mismatch** → diff suggests an update but plan shows destroy+create (common with `for_each` key changes, `name` changes on resources that don't support in-place rename).

---

### Step 4: Review PR code changes

For every finding below, be exhaustive — list every file, every occurrence, every affected resource. The review report is the only document the reviewer will read, so nothing can be left out or summarized.

### 4.1 Code quality

Be specific — include file names, block references, and show the problematic code snippet. The reader should be able to go directly to the issue and fix it without searching.

Check for:
- **Dead code**: commented-out blocks that should be deleted (git history preserves old code, no need to keep it in comments)
- **Indentation errors**: misaligned blocks, inconsistent spacing — show the exact block
- **Naming conventions**: inconsistent metric IDs, resource names across similar services
- **Missing newline at end of file**: `\ No newline at end of file` in diff
- **Copy-paste errors**: duplicated blocks with wrong references
- **Hardcoded values** that should be variables

**Example of a good code quality finding:**

> **Indentation sai** – `ecs_s4_grading_not_submit_worker_service.tf`
>
> Trong cả 3 alarm mới (`_set_1_task`, `_scale_out`, `_scale_in`), block `metric_query` cho `m3` bị thụt lề thừa:
> ```hcl
> # SAI - thừa 2 space trước metric_query
>     metric_query {
>     id = "m3"
>
> # ĐÚNG - nên là
>   metric_query {
>     id = "m3"
> ```
> Xuất hiện 3 lần (1 lần trong mỗi alarm resource).

### 4.2 Security

Check for:
- IAM policy changes (overly permissive, wildcard resources)
- Security group changes (open ports, 0.0.0.0/0)
- Secret/credential exposure in plain text
- S3 bucket policy changes (public access)
- KMS key policy changes
- VPC/subnet changes

If no security-relevant changes exist, state that clearly: "Không có vấn đề bảo mật. PR chỉ thay đổi [resource types], không touch IAM, security group, hay secret nào."

### 4.3 Impact assessment

Evaluate each category and provide a clear verdict:

**Downtime:**
- Will any running service be interrupted?
- Will any ECS task be restarted?
- Will any database be modified?
- Distinguish between "application downtime" (users affected) and "monitoring/autoscaling gap" (app runs but scaling doesn't work temporarily)

**Data loss:**
- Are any stateful resources (RDS, S3, SQS, DynamoDB) being destroyed?
- Are any queues being deleted (messages would be lost)?
- Is alarm history being lost (destroy+recreate alarms)?
- Distinguish between "critical data loss" (user data, business data) and "non-critical metadata loss" (alarm history, tags)

**Cost impact:**
- New resources being created → estimate monthly cost with specific numbers
- Resources being destroyed → estimate monthly savings
- Configuration changes that affect cost (e.g., more alarms, larger instances)
- Example: "$0.10/alarm/month × 16 new alarms = +$1.60/month"

**Blast radius:**
- List every service/component affected
- For each, state whether it's direct (code change targets it) or indirect (dependency)
- Rate severity: High (service disruption), Medium (monitoring gap), Low (cosmetic/tag only)

**Apply timing recommendation:**
Based on the impact assessment, recommend when to apply:
- If monitoring/autoscaling gap exists → "Nên apply ngoài giờ cao điểm để giảm rủi ro. Tránh khung giờ {peak hours based on region}."
- If no impact → "Có thể apply bất kỳ lúc nào."
- If high risk → "Cần plan deployment window cụ thể với team."

**Summary table (include in every review):**

| Tiêu chí | Mức độ | Ghi chú |
|---|---|---|
| Downtime ứng dụng | ❌ Không / ⚠️ Có thể / 🔴 Có | Chi tiết |
| Gián đoạn monitoring/autoscaling | ❌ Không / ⚠️ Có thể / 🔴 Có | Ước tính thời gian |
| Mất dữ liệu | ❌ Không / ⚠️ Không nghiêm trọng / 🔴 Nghiêm trọng | Mất gì |
| Chi phí phát sinh | ❌ Không / ⚠️ Nhẹ / 🔴 Đáng kể | Ước tính/tháng |
| Độ khó rollback | ✅ Dễ / ⚠️ Trung bình / 🔴 Khó | Cách rollback |

---

## Step 5: Generate output files

**Output completeness:** Write every resource, every file, every section in full detail. Do not abbreviate, summarize, or skip repetitive items. Specifically banned phrases: "(N file còn lại tương tự)", "(các service khác tương tự)", "(xem thêm...)", "etc.", "and so on", "similarly for...", or any variation. The output files are the final deliverable — readers will not have access to the raw diff or plan, so every detail must be present. If a section has 20 rows in a table, write all 20 rows. If 16 services are affected, list all 16 with their full resource addresses. Use `fsWrite` followed by `fsAppend` to handle large files — do not truncate to save space.

### Output 1: Review report

Save to `{OUTPUT_DIR}/review-{TICKET_ID}.md`

Structure:
```markdown
# Terraform PR Review – {TICKET_ID}

## 1. Cross-check: Git Diff vs Terraform Plan
### Kết luận
(Khớp/Không khớp + tóm tắt)
### Chi tiết
(Bảng so sánh từng nhóm resource)
### Thay đổi thừa trong plan (nếu có)
(Resources không thuộc PR - drift)

## 2. Code Review
### 2.1 Code Quality
(Từng issue cụ thể với file/block reference và code snippet)
### 2.2 Security
(Findings hoặc "không có vấn đề")

## 3. Impact Assessment
### 3.1 Downtime
### 3.2 Data Loss
### 3.3 Cost Impact
### 3.4 Blast Radius
### 3.5 Apply Timing Recommendation
### 3.6 Summary Table
```

Write the files in this order: review report first (requires reading diff + plan), then summary files (requires reading template). This order ensures the analysis is complete before formatting begins.

### Output 2: Terraform plan summary

Save to:
- `{OUTPUT_DIR}/summary-{TICKET_ID}.md` (Vietnamese)
- `{OUTPUT_DIR}/summary-{TICKET_ID}-ja.md` (Japanese)

Before writing, read the summary template to get the exact output format. Search for the file `tf-pr-reviewer-summary-template.md` using `fileSearch`, then read it. The filename is intentionally unique to avoid matching other files in the workspace.

The template contains section patterns for common terraform change types: AMI updates, tag changes, ECS task definitions, ECS services, SQS queues, SQS visibility timeout, IAM policies, S3 bucket ACL, CloudWatch dashboards, and AWS provider changes. The output summary must follow this template's format exactly — use the same heading structure, table format, scope labels, and note conventions.

Key rules:
- Select only template patterns that match the actual plan output — do not include empty sections
- If the plan contains a change type not covered by any template pattern, create a new section following the same style (numbered heading, resource table, scope label, explanation)
- Group related changes together (e.g., all alarm destroy+create for the same refactor go in one section)
- Label each section: `[Thuộc phạm vi release]` or `[Không thuộc phạm vi release nhưng không ảnh hưởng]`
- For destroy+create pairs, explain why (key rename, refactor, etc.) so readers don't panic seeing large destroy counts
- Resource counts in the summary must match the plan summary line exactly

**Attribute-level detail in summary (critical):**

The summary is the primary document that reviewers and the Japanese team read. They will NOT look at the raw plan output. So every section must include enough detail for the reader to understand exactly what terraform will change at the attribute level.

For each section, after the resource table, add a "Chi tiết thay đổi" (変更詳細) subsection that shows:
- **Update in-place**: Each changed attribute with `old_value → new_value`
- **Create**: Key configuration values (threshold, period, expression, scaling_adjustment, cooldown, etc.)
- **Destroy+Create**: What the old config looked like vs the new config, highlighting key differences

When multiple resources share the same pattern of changes (e.g., 16 alarms all get the same expression refactor), you can describe the pattern once with a concrete example from one resource, then state "Tương tự cho N resource còn lại" — but the example must show actual values, not placeholders.

**Example of a well-detailed summary section:**

```markdown
### 1. Refactor CloudWatch Metric Alarm – 80 resources (32 destroy + 48 create)

| # | Resource destroy (for_each cũ) | Resource create (riêng lẻ mới) |
|---|---|---|
| 1 | `...["scale-in"]` | `..._scale_in` |
| 2 | `...["scale-out"]` | `..._scale_out` |
| 3 | _(không có)_ | `..._set_1_task` |

**Chi tiết thay đổi (ví dụ từ s1_combine_certificate_worker_fg):**

**Alarm cũ** (for_each – 2 alarm: scale-in, scale-out):
- `metric_query.e1.expression`: `"FILL(m1, 0)"`
- Dùng chung 1 expression cho cả scale-in và scale-out, chỉ khác threshold

**Alarm mới** (riêng lẻ – 3 alarm):
- `_scale_out`: expression = `"IF(m1 > 0, m1, 0)"`, threshold = `1`, period = `60`, evaluation_periods = `1`
- `_scale_in`: expression = `"IF(m1 <= 0, 1, 0)"`, threshold = `1`, period = `300`, evaluation_periods = `1`
- `_set_1_task` _(mới)_: expression = `"IF(m1 == 1, 1, 0)"`, threshold = `1`, period = `60` — alarm mới để detect khi chỉ còn 1 task running

Tương tự cho 15 service còn lại.
```

**Japanese version conventions:**
- `[Thuộc phạm vi release]` → `[リリース範囲内]`
- `[Không thuộc phạm vi release nhưng không ảnh hưởng]` → `[リリース範囲外・影響なし]`
- `Thêm mới (Add)` → `新規追加 (Add)`
- `Thay đổi (Change)` → `変更 (Change)`
- `Xóa (Destroy)` → `削除 (Destroy)`
- Keep all resource names, terraform identifiers, technical terms, and code snippets in English
- Translate explanatory text, section headers, and descriptions to Japanese

---

## Step 6: Present results

After generating files, provide a brief summary in chat:
- Confirmation that files were created with their paths
- Top 2-3 findings from the review (most important issues)
- Whether the plan is safe to apply or has concerns that need addressing

If the user asks for changes to the output language or format, regenerate the affected file.

---

## Edge cases

- **Plan file not found**: Ask the user to verify the path.
- **No diff output**: Branches might be identical for that folder, or branch names might be wrong. Ask the user to verify. Do NOT run git fetch/pull yourself.
- **Plan shows changes not in diff**: Flag as state drift. These are real changes that will be applied but aren't part of the PR.
- **Diff shows changes not in plan**: The plan might have been generated from a different branch or before the latest commit. Recommend regenerating the plan.
- **Very large plans (100+ resources)**: Group by resource type and list every resource. For attribute-level detail, when many resources share the same change pattern, show full detail for one representative resource and state "Tương tự cho N resource còn lại" — but always list every resource address individually in the table.
- **Very large diffs**: Use `git diff --stat` first, then diff individual files to avoid truncation.

---

## Examples

Below are sanitized excerpts showing what good output looks like for each deliverable.

### Example: Review report excerpt (cross-check section)

```markdown
## 1. Cross-check: Git Diff vs Terraform Plan

**Kết luận: Plan phản ánh đúng và đủ các thay đổi trong PR.**

- 4 file `.tf` thay đổi alarm → plan cho thấy destroy alarm `for_each` cũ + create alarm riêng lẻ mới → khớp
- `s4_worker_queue_test_grading` alarm đã tồn tại dạng riêng lẻ → plan chỉ update in-place → khớp
  - Diff thay đổi `expression` trong `metric_query.e1`: `"FILL(m1, 0)"` → `"IF(m1 > 0, m1, 0)"` → plan show `~ expression: "FILL(m1, 0)" -> "IF(m1 > 0, m1, 0)"` → khớp
  - Diff thay đổi `datapoints_to_alarm`: `2` → `1` cho alarm `scale_out` → plan show `~ datapoints_to_alarm: 2 -> 1` → khớp
- `aws_appautoscaling_policy` key rename `scale-in`→`scale_in` → plan destroy `["scale-in"]` + create `["scale_in"]` → khớp
  - Diff thêm key `set_1_task` mới → plan create `["set_1_task"]` → khớp
  - Cấu hình `scaling_adjustment` trong diff: scale_in = `-1`, scale_out = `1`, set_1_task = `1` → plan giá trị tương ứng → khớp

**Thay đổi thừa (không nằm trong diff PR nhưng có trong plan):**
- `aws_iam_user.s5_iam_user` - `tags_all`: thêm `Env = "stg"`, `Project = "tofas"` → drift từ state, không liên quan PR
- `module.aurora_s1_postgres.aws_appautoscaling_target.read_replica_count[0]` - `tags`: thêm `CreateBy = "terraform"` → drift, không liên quan PR
```

### Example: Review report excerpt (code quality section)

```markdown
### 2.1 Code Quality

**a) Code comment out chưa xóa – 4 files**

| File | Nội dung | Số dòng |
|---|---|---|
| `ecs_s1_result_statistic_worker_fg_service.tf` | Block `for_each` alarm cũ | ~50 dòng |
| `ecs_s1_send_result_publication_mail_to_manager_worker_fg_service.tf` | Block `for_each` alarm cũ | ~50 dòng |

Nên xóa hẳn vì đã có git history.

**b) Indentation sai – `ecs_s4_grading_not_submit_worker_service.tf`**

Block `metric_query` cho `m3` bị thụt lề thừa:
\```hcl
# SAI
    metric_query {
    id = "m3"

# ĐÚNG
  metric_query {
    id = "m3"
\```
Xuất hiện 3 lần (1 lần trong mỗi alarm resource).
```

### Example: Review report excerpt (impact assessment)

```markdown
### 3.6 Summary Table

| Tiêu chí | Mức độ | Ghi chú |
|---|---|---|
| Downtime ứng dụng | ❌ Không | App vẫn chạy bình thường |
| Gián đoạn autoscaling | ⚠️ Có (3-10 phút) | 15 service bị ảnh hưởng |
| Mất dữ liệu | ❌ Không | Chỉ mất alarm history (không critical) |
| Chi phí phát sinh | ⚠️ Nhẹ | +$1.60/tháng CloudWatch |
| Độ khó rollback | ✅ Dễ | Revert PR và chạy lại terraform apply |

### 3.5 Khuyến nghị thời điểm apply
Nên apply ngoài giờ cao điểm (tránh 9:00-18:00 JST) do 15 service sẽ mất autoscaling tạm thời 3-10 phút.
```

### Example: Summary report excerpt

```markdown
## Tổng quan thay đổi Terraform plan

> ⚠️ **Dưới đây là tổng hợp thay đổi terraform...**

## MyProject STG – Tokyo (ap-northeast-1)
- **Thêm mới (Add):** 96 resources
- **Thay đổi (Change):** 6 resources
- **Xóa (Destroy):** 64 resources

> Lưu ý: 64 resources bị destroy là do refactor CloudWatch Metric Alarm
> từ `for_each` sang resource riêng lẻ. Thực tế không mất service.

---

### 1. Refactor CloudWatch Metric Alarm – 78 resources (destroy + create)

| # | Resource gốc (destroy) | Resource mới (create) |
|---|---|---|
| 1 | `aws_cloudwatch_metric_alarm.worker_fg["scale-in"]` | `aws_cloudwatch_metric_alarm.worker_fg_scale_in` |
| 2 | `aws_cloudwatch_metric_alarm.worker_fg["scale-out"]` | `aws_cloudwatch_metric_alarm.worker_fg_scale_out` |
| 3 | _(không có)_ | `aws_cloudwatch_metric_alarm.worker_fg_set_1_task` |

**Chi tiết thay đổi (ví dụ từ worker_fg):**

**Alarm cũ** (for_each – `scale-in` và `scale-out` dùng chung config):
- `metric_query.e1.expression`: `"FILL(m1, 0)"`
- `threshold`: `2` (scale-out), `1` (scale-in)
- `evaluation_periods`: `2`
- `period`: `60`

**Alarm mới** (3 alarm riêng biệt với logic khác nhau):
- `_scale_out`: expression = `"IF(m1 > 0, m1, 0)"`, threshold = `1`, period = `60`, evaluation_periods = `1`, datapoints_to_alarm = `1`
- `_scale_in`: expression = `"IF(m1 <= 0, 1, 0)"`, threshold = `1`, period = `300`, evaluation_periods = `1`, datapoints_to_alarm = `1`
- `_set_1_task` _(alarm mới)_: expression = `"IF(m1 == 1, 1, 0)"`, threshold = `1`, period = `60` — detect khi chỉ còn 1 task running, dùng để giữ min 1 task

**Khác biệt chính:**
- Expression cũ dùng `FILL(m1, 0)` → mới dùng `IF()` logic riêng cho từng mục đích scale
- Thêm alarm `set_1_task` hoàn toàn mới để prevent scale-in về 0 task
- `datapoints_to_alarm` giảm từ 2 → 1 cho scale_out (phản ứng nhanh hơn)

Tương tự cho 15 service còn lại (cùng pattern thay đổi).

➡️ [Thuộc phạm vi release] Refactor alarm để có logic scale riêng biệt.

---

### 2. Cập nhật Tags – 3 resources (update in-place)

| Resource | Thay đổi |
|---|---|
| `aws_iam_user.s5_iam_user` | `tags_all`: thêm `Env = "stg"`, `Project = "tofas"` |
| `module.aurora_s1_postgres.aws_appautoscaling_target.read_replica_count[0]` | `tags`: thêm `CreateBy = "terraform"` |
| `module.aurora_s6.aws_appautoscaling_target.read_replica_count[0]` | `tags`: thêm `CreateBy = "terraform"` |

➡️ [Không thuộc phạm vi release nhưng không ảnh hưởng] State drift – Terraform tự động bổ sung tag.
```
