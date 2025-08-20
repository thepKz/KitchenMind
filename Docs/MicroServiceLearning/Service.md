**Danh sách microservice** cho KitchenMind (tối ưu cho **RBAC theo permission**). Mỗi service kèm **namespace quyền** gợi ý để bạn map role → permission.

# 🟢 Core (MVP)

1. **auth-service**

   * Đăng ký/đăng nhập, JWT/OAuth, session, refresh token, email verify.
   * **Perms**: `auth.user:read|update`, `auth.session:revoke`, `auth.role:list`, `auth.account:link`

2. **rbac-service** (policy center)

   * Quản lý **role**, **permission**, **role-binding** (user↔role), **policy**.
   * **Perms**: `rbac.role:create|update|delete|assign`, `rbac.permission:list`, `rbac.policy:evaluate`

3. **user-profile-service**

   * Hồ sơ cơ bản (tên, avatar, phone), tùy chọn thông báo.
   * **Perms**: `profile.self:read|update`, `profile.admin:read|update`

4. **inventory-service**

   * Kho, item, batch/lot, zone, FEFO/FIFO, nhập/xuất kho.
   * **Perms**: `inventory.item:create|read|update|delete`, `inventory.batch:receive|move|deduct`, `inventory.zone:manage`, `inventory.report:read`

5. **recipe-service**

   * Công thức, nguyên liệu, dinh dưỡng cơ bản, versioning nhẹ.
   * **Perms**: `recipe:create|read|update|delete`, `recipe.publish`, `recipe.rate:write`

6. **planner-service**

   * Lập **menu tuần** theo tồn kho/khẩu phần/ràng buộc; commit/complete bữa.
   * **Perms**: `planner.menu:create|read|update|delete|commit`, `planner.meal:complete`, `planner.constraint:manage`

7. **shopping-list-service**

   * Sinh danh sách mua từ menu – tồn kho; tick/check; gợi ý thay thế.
   * **Perms**: `shopping.list:create|read|update|delete|share`, `shopping.item:check|adjust`

8. **notification-service**

   * Push/email: HSD, nhắc bữa, cảnh báo ngân sách; preference.
   * **Perms**: `notify.template:manage`, `notify.send`, `notify.pref:read|update`

# 🟡 Nên có (khi bắt đầu thu phí/mở rộng)

9. **payment-subscription-service**

   * Gói, chu kỳ, cổng thanh toán (Momo/VNPAY/PayOS/Visa); webhook.
   * **Perms**: `billing.plan:read`, `billing.subscription:create|read|upgrade|cancel`, `billing.invoice:read`

10. **health-profile-service** (encrypted)

* Dị ứng, bệnh nền, tôn giáo/chay; chỉ expose **flags/ids** cho planner.
* **Perms**: `health.self:read|update`, `health.admin:read` (audit), *no export by default*

11. **ocr-input-service**

* OCR hóa đơn, parse item/giá/số lượng; barcode scan; template.
* **Perms**: `ocr.job:create|read`, `ocr.template:manage`

12. **reporting-analytics-service**

* Báo cáo tồn kho, chi tiêu, lãng phí, dinh dưỡng, completion rate.
* **Perms**: `report.inventory:read`, `report.spend:read`, `report.nutrition:read`

# 🔵 Hạ tầng & chia sẻ

13. **api-gateway**

* Entry point, authN, **authZ (PDP call rbac-service)**, rate limit.

14. **event-bus / outbox-relay**

* Kafka/RabbitMQ + outbox worker (inventory, planner, payments, shopping).
* **Event tiêu biểu**: `ItemAdded`, `BatchCreated`, `StockDeducted`, `MenuCommitted`, `MealCompleted`, `ShoppingListFinalized`, `PaymentSucceeded`, `SubscriptionActivated`, `ExpirySoon`.

15. **audit-log-service**

* Ghi log truy cập & thay đổi (PII-safe), tra cứu sự cố.
* **Perms**: `audit:read` (admin only)

16. **file-media-service**

* Upload ảnh sản phẩm/recipe (Cloudinary proxy), ký URL tạm.
* **Perms**: `media.upload`, `media.read:own`, `media.admin:read`

# 🧭 Gợi ý role → permission (mẫu)

* **Guest**: `recipe:read`, `planner.menu:read(public)`
* **Registered User**: `profile.self:*`, `inventory.* (own)`, `planner.* (own)`, `shopping.* (own)`
* **Premium User**: như Registered + `report.* (own)`
* **Family Owner**: như Premium + `rbac.role:assign (household)`, `inventory.zone:manage`
* **Family Member**: `inventory.item:read|deduct`, `planner.menu:read`, `shopping.list:check`
* **Admin**: `rbac.*`, `audit:read`, `billing.*`, `report.*`
* **Nutritionist** (Pro): `planner.menu:propose`, `recipe:modify(pro)`, `health.self:read (by consent)`

# ✅ Lời khuyên triển khai RBAC

* **Check theo permission (resource\:action)** ở gateway/middleware; role chỉ là **tập permission**.
* **Namespace rõ ràng** như trên để tránh đụng nhau giữa service.
* **ABAC nhẹ** cho phạm vi “own/household”: dùng thuộc tính `subject.userId == resource.ownerId` hoặc `householdId ∈ subject.households`.
* **Outbox bật** tại: `inventory`, `planner`, `shopping`, `payment` (điểm giao dịch dữ liệu ↔ event).
