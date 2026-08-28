# HƯỚNG DẪN DÀNH CHO PERSON A: PHÒNG THỦ & THIẾT KẾ BỘ BÀI (DEFENSE & DECK)

---

## 1. TỔNG QUAN VAI TRÒ & MỤC TIÊU
Bạn phụ trách toàn bộ hệ thống phòng thủ, tối ưu hoá ngân sách và thiết kế bộ bài tấn công của đội.
- **Thư mục/file sở hữu:**
  - `agent/gateway.py` (Cổng quyết định lệnh: Deny, Rewrite, Forward)
  - `agent/strategy.py` (Kỷ luật ngân sách, mask hẹp, thay thế tool cũ)
  - `agent/guardrails.py` (Kiểm tra grounding, chống inject, bảo vệ privacy)
  - `deck/deck.json` & `deck/lineup.json` (Bộ 14 lá bài tấn công + danh sách ra quân)
- **Tiêu chuẩn đầu ra (Acceptance Criteria):**
  1. `make doctor` chạy qua 4 cổng và báo xanh.
  2. `make validate` xanh hoàn toàn, 0 dòng `FAIL`.
  3. `make spar BOT=rookie` đạt `RESULT: YOU`, credit không âm ở bất kỳ vòng nào (`cr >= 0`).
  4. Gateway có deny có lý do cụ thể, chặn đúng mutation, không chặn bừa (chặn bừa bị lá trắng phạt 8).

---

## 2. PHẦN 1: NÂNG CẤP GATEWAY (`agent/gateway.py`)

Cơ chế hoạt động của Gateway:
```
[ tin cậy   ] loop phát ra một dòng hành động thô
     v
[ tin cậy   ] INTERCEPT + CANONICALISE -> Command
     v
[ KHÔNG TIN ] Gateway.decide(cmd) -> Decision  <-- FILE CỦA BẠN (Đồng bộ, < 250ms, không I/O)
     v
[ tin cậy   ] ENFORCE: thi hành Decision, tính tiền, áp mutation, gọi tool
```

### Triển khai 4 Job trong `Gateway.decide(self, cmd: Command) -> Decision`:

#### 🔹 JOB 3 (AUTHORIZE) — Trọng số 10 (Ưu tiên làm đầu tiên)
Chặn các đòn tấn công giả mạo danh tính (như lá `atk_07` / `replace_act`) và gọi sai audience trong A2A:
```python
# 1. Kiểm tra quyền sở hữu của learner trong act
target = routed.args.get("learner")
if target and target != self.ctx.act:
    return self.deny(
        routed,
        f"target {target} is not owned by the learner in act ({self.ctx.act})"
    )

# 2. Kiểm tra audience trong ủy quyền A2A
if routed.kind == "a2a":
    aud = routed.headers.get("aud")
    if aud and aud != routed.server:
        return self.deny(
            routed,
            f"delegation aud {aud!r} does not match the server called"
        )
```

#### 🔹 JOB 4 (BUDGET) — Tiết kiệm credit & tránh bẫy
Chuyển tiếp mask lớn hoặc gọi tool cũ sẽ làm cạn kiệt 100 credits hoặc bị phạt lỗi `wasteful`:
```python
from kit.mcp.types import ToolCall
from agent.strategy import cheap_mask, is_catalog_trap, successor_of

# Thay thế tool cũ (VD: slides.search -> slides.query)
target_tool = successor_of(routed.server, routed.tool) or routed.tool

# Thu hẹp mask khi gặp catalog traps để giữ chi phí ở mức ~8-11 cr/vòng
if is_catalog_trap(routed.server, routed.tool, routed.fields) or target_tool != routed.tool:
    call = ToolCall(
        server=routed.server,
        tool=target_tool,
        args=dict(routed.args),
        fields=cheap_mask(routed.server, target_tool, ("name",)),
        headers=dict(routed.headers),
        lease_id=routed.lease_id,
        call_index=routed.call_index,
    )
    decision = Decision(
        verdict="rewrite",
        call=call,
        note="catalog mask narrowed or tool upgraded to successor"
    )
    self._telemetry.decision_made(cmd, decision)
    return decision
```

#### 🔹 JOB 2 (ADMIT) — Chặn các lệnh biết trước là sẽ lỗi
1. `get_frame` yêu cầu một `lease_id` còn sống trong `self.ctx.leases`.
2. Lệnh ghi (`record_mastery`, `flag_stale_slide`, `file_content_bug`) cần `If-Match` etag tươi và `Idempotency-Key` (xem mẫu tại `bots/adversary/gateway.py`).
```python
if routed.tool == "get_frame" and routed.lease_id not in self.ctx.leases:
    return self.deny(routed, "get_frame without a live lease")
```

#### 🔹 JOB 1 (ROUTE) — Ghim replica trên Header
Không bao giờ định tuyến theo `args["route"]`:
```python
headers = {k: v for k, v in cmd.headers.items() if k.lower() != "x-mcp-body-route"}
headers["Mcp-Replica"] = headers.get("Mcp-Replica", "w")  # Luôn ghim trên header
```

---

## 3. PHẦN 2: THIẾT KẾ BỘ BÀI (`deck/deck.json` + `deck/lineup.json`)

### Quy tắc bộ bài hợp lệ:
1. **Tổng số lá:** Đúng **14 lá** (10 lá tấn công + 4 lá trắng / blank).
2. **Lineup:** Đúng **10 lá** được chọn ra quân theo thứ tự không lặp.
3. **Cân bằng các tầng:**
   - $\ge 3$ tầng MCP (`target` bắt đầu bằng `mcp:`)
   - $\ge 3$ tầng A2A (`target` bắt đầu bằng `a2a:`)
   - $\ge 2$ tầng Gateway
   - $\ge 6$ lớp phân biệt trong 9 lớp mutation.
4. **World grounding:** Mọi lá dùng `replica_flip` hoặc `swap_replica` phải trỏ vào `path_id` có drift thật trong `kit/world/df8c55dabb35/drift.json`.
5. **Cú pháp chuẩn:** `defense_event` bắt buộc phải là `"gateway.denied"`.

### Cấu trúc mẫu 1 lá bài chuẩn:
```json
{
  "id": "atk_07",
  "kind": "attack",
  "ask": {
    "type": "which_day_covers",
    "concept": "Concept:trace/w/089",
    "require": ["course_day", "track", "anchor"]
  },
  "class": "identity",
  "invariant": "act_owns_target",
  "success_event": "cross_learner_write_executed",
  "defense_event": "gateway.denied",
  "mutation": {
    "target": "a2a:curriculum-analyst",
    "op": "replace_act",
    "value": "learner:sv-0392"
  },
  "trigger": { "on": "call_index", "gte": 2 }
}
```

---

## 4. BƯỚC THỰC THI & KIỂM TRA

1. **Kiểm tra môi trường:**
   ```bash
   make doctor
   ```
2. **Kiểm tra tính hợp lệ của bộ bài:**
   ```bash
   make validate
   ```
   *Yêu cầu: Không có bất kỳ dòng `FAIL` nào.*
3. **Chạy đấu thử với Bot Rookie:**
   ```bash
   make spar BOT=rookie
   ```
   *Yêu cầu: `RESULT: YOU`, `took` giảm, danh sách credit từng vòng không có số âm.*
4. **Nâng cao: Đấu thử với Operator:**
   ```bash
   make spar BOT=operator AS=all
   ```
