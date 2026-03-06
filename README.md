### Setup ban đầu
```
Key A: AIzaSyDKCH... ✅ healthy | 0/13 RPM | 0/250 RPD
Key B: AIzaSyBymu... ✅ healthy | 0/13 RPM | 0/250 RPD
Key C: AIzaSyCxyz... ✅ healthy | 0/13 RPM | 0/250 RPD
```

---

## PHASE 1: Requests đầu tiên (T=0s)

### Request 1-6 đến đồng thời:

```
┌─────────────────────────────────────────────────────────────┐
│ RequestLimiter: Max 6 concurrent Gemini requests            │
│ Cho phép tối đa 6 requests xử lý cùng lúc                   │
└─────────────────────────────────────────────────────────────┘

Request 1 → RequestLimiter ✅ → APIKeyRotator.getBestKey()
    ↓
    availableKeys = [Key A, Key B, Key C] (cả 3 đều available)
    Round-robin index = 0 → Chọn Key A
    Key A: lastRequestTime = T=0s

Request 2 → RequestLimiter ✅ → APIKeyRotator.getBestKey()
    ↓
    availableKeys = [Key B, Key C] (Key A vừa dùng nhưng vẫn available)
    Wait! Kiểm tra lại: Key A đã dùng lúc T=0s, now = T=0s
    now - lastRequestTime = 0s < 1s (minDelay) → Key A KHÔNG available

    availableKeys = [Key B, Key C] (chỉ còn 2 keys)
    Round-robin index = 1 → Chọn Key B
    Key B: lastRequestTime = T=0s

Request 3 → RequestLimiter ✅ → APIKeyRotator.getBestKey()
    ↓
    Kiểm tra:
    - Key A: lastRequestTime = 0s, now = 0s → KHÔNG available
    - Key B: lastRequestTime = 0s, now = 0s → KHÔNG available
    - Key C: chưa dùng → Available

    availableKeys = [Key C]
    Round-robin index = 2 → Chọn Key C
    Key C: lastRequestTime = T=0s

Request 4 → RequestLimiter ✅ → APIKeyRotator.getBestKey()
    ↓
    Kiểm tra TẤT CẢ keys:
    - Key A: lastRequestTime = 0s, now = ~0.001s → 0.001s < 1s → ❌
    - Key B: lastRequestTime = 0s, now = ~0.001s → 0.001s < 1s → ❌
    - Key C: lastRequestTime = 0s, now = ~0.001s → 0.001s < 1s → ❌

    availableKeys = [] (EMPTY!)

    → Tính toán wait time:
    Wait time cho Key A = lastRequestTime + minDelay - now
                        = 0 + 1000ms - 1ms = 999ms

    ❌ THROW ERROR: "All keys busy. Next available in 1s"

    → Request 4 được XẾP HÀNG trong RequestLimiter queue
    → Sẽ retry sau khi có key available

Request 5, 6 → Tương tự Request 4, vào hàng đợi
```

---

## PHASE 2: Sau 1 giây (T=1s)

```javascript
// Request 1, 2, 3 đã gọi API và đang chờ response
// Key A, B, C đều đã qua 1s delay

T=1.0s: Request 4 retry → getBestKey()
    ↓
    Kiểm tra:
    - Key A: lastRequestTime = 0s, now = 1000ms → 1000ms >= 1000ms ✅
    - Key A: requestsThisMinute = 1, limit = 13 ✅
    - Key A: isInCooldown = false ✅
    - Key A: status = healthy ✅

    availableKeys = [Key A, Key B, Key C] (cả 3 available lại)

    Round-robin index = 3 % 3 = 0 → Chọn Key A (lần 2)
    Key A: requestsThisMinute = 2
    Key A: lastRequestTime = T=1.0s

T=1.001s: Request 5 retry → getBestKey()
    ↓
    - Key A: lastRequestTime = 1000ms, now = 1001ms → 1ms < 1000ms ❌
    - Key B: lastRequestTime = 0ms, now = 1001ms → 1001ms >= 1000ms ✅

    availableKeys = [Key B, Key C]
    Round-robin index = 4 % 2 = 0 → Chọn Key B (lần 2)
    Key B: requestsThisMinute = 2
    Key B: lastRequestTime = T=1.001s

T=1.002s: Request 6 retry → getBestKey()
    ↓
    availableKeys = [Key C]
    Chọn Key C (lần 2)
    Key C: requestsThisMinute = 2
    Key C: lastRequestTime = T=1.002s
```

---

## PHASE 3: Sau 2 giây (T=2s)

```javascript
// Giả sử Request 1, 2, 3 đã hoàn thành
// Request 4, 5, 6 đang chờ response
// Còn Request 7, 8, 9, 10 trong queue

T=2.0s: Request 7 → getBestKey()
    ↓
    Kiểm tra available:
    - Key A: lastRequestTime = 1000ms, now = 2000ms → 1000ms >= 1000ms ✅
    - Key A: requestsThisMinute = 2, limit = 13 ✅

    availableKeys = [Key A, Key B, Key C]
    Chọn Key A (lần 3)
    Key A: requestsThisMinute = 3

T=2.001s: Request 8 → Key B (lần 3)
T=2.002s: Request 9 → Key C (lần 3)
T=3.0s: Request 10 → Key A (lần 4)
```

---

## KẾT QUẢ SAU 3 GIÂY

```
Key A: 4 requests (Request 1, 4, 7, 10)
    Timeline: T=0s, T=1.0s, T=2.0s, T=3.0s
    Status: 4/13 RPM ✅ healthy

Key B: 3 requests (Request 2, 5, 8)
    Timeline: T=0s, T=1.001s, T=2.001s
    Status: 3/13 RPM ✅ healthy

Key C: 3 requests (Request 3, 6, 9)
    Timeline: T=0s, T=1.002s, T=2.002s
    Status: 3/13 RPM ✅ healthy

Total: 10 requests được phân tải đều trên 3 keys
Load balancing: 40% - 30% - 30% (khá đều)
```

---

## LOGIC PHÂN TÍCH CHI TIẾT

### 1. **Filter Available Keys**

Mỗi lần `getBestKey()` được gọi:

```javascript
const availableKeys = Array.from(this.keys.values()).filter(key => {
    const usage = this.keyUsage.get(key.id);
    const health = this.keyHealth.get(key.id);
    const now = Date.now();

    // Kiểm tra 6 điều kiện:
    return (
        key.isActive &&                                    // ✅ 1. Active
        health.status === 'healthy' &&                     // ✅ 2. Healthy
        usage.requestsThisMinute < 13 &&                   // ✅ 3. < 13 RPM
        usage.requestsThisHour < 700 &&                    // ✅ 4. < 700 RPH
        !usage.isInCooldown &&                             // ✅ 5. Không cooldown
        now - usage.lastRequestTime >= 1000                // ✅ 6. Đã chờ >= 1s
    );
});
```

**Điều kiện 6 là KEY**: Đảm bảo mỗi key chỉ xử lý 1 request/giây!

### 2. **Sort by Lowest Usage**

```javascript
availableKeys.sort((a, b) => {
    const usageA = this.keyUsage.get(a.id);
    const usageB = this.keyUsage.get(b.id);

    // Ưu tiên key có RPM thấp hơn
    if (usageA.requestsThisMinute !== usageB.requestsThisMinute) {
        return usageA.requestsThisMinute - usageB.requestsThisMinute;
    }

    // Nếu bằng nhau thì xem priority
    return b.priority - a.priority;
});
```

**Ví dụ sắp xếp:**
```
Key A: 5 RPM
Key B: 3 RPM  ← Chọn B vì ít usage hơn
Key C: 8 RPM

Sau sort: [Key B, Key A, Key C]
```

### 3. **Round-Robin Selection**

```javascript
// this.keyRotationIndex tăng dần: 0, 1, 2, 3, 4, 5...
const selectedKey = availableKeys[this.keyRotationIndex % availableKeys.length];
this.keyRotationIndex = (this.keyRotationIndex + 1) % availableKeys.length;

// Ví dụ: availableKeys = [Key B, Key A, Key C] (3 keys)
// Lần 1: index = 0 % 3 = 0 → Key B
// Lần 2: index = 1 % 3 = 1 → Key A
// Lần 3: index = 2 % 3 = 2 → Key C
// Lần 4: index = 3 % 3 = 0 → Key B (lặp lại)
```

---

## TRƯỜNG HỢP ĐẶC BIỆT: KEY HẾT QUOTA

### Scenario: Key B hết quota giữa chừng

```javascript
Timeline:
T=0s: Request 1 → Key A ✅
T=0s: Request 2 → Key B ✅ (Nhưng Gemini trả về 429 - Quota exceeded!)
      ↓
      executeGeminiAPI() catch error
      ↓
      apiManager.recordUsage(keyB, false, 100ms, error)
      ↓
      // Phát hiện lỗi quota
      isDailyQuotaExceeded = true
      ↓
      key.isActive = false  ← Key B BỊ DISABLE!
      health.status = 'quota_exceeded'
      ↓
      // Retry với key khác
      attemptCount = 1 < maxRetries (3)
      ↓
      getBestKey() lần 2
      ↓
      availableKeys = [Key A, Key C] (Key B đã bị loại!)
      ↓
      Chọn Key C → Success! ✅

T=0s: Request 3 → Key C ✅
T=1s: Request 4 → Key A ✅ (Key B vẫn disabled)
...

// Sau 24 giờ, Key B tự động re-enable
T=24h: setTimeout callback
       ↓
       key.isActive = true
       health.status = 'healthy'
```

---
