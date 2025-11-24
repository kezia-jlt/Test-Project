# Manual Webhook Tools

Tools untuk mengirim webhook manual ke Digiflazz dan monitoring webhook yang pending.

## 🚀 Quick Start

### 1. Check Pending Webhooks

```bash
npm run webhook:check
# atau
node check-pending-webhooks.js
```

**Output:**

```
📋 PENDING WEBHOOKS CHECKER
======================================================================

⚠️  Found 3 transactions with pending webhooks:

1. abc123xyz
   Status: SUCCESS | RC: 00
   Product: TELKOMSEL10 → 081234567890
   SN: SN123456789
   Age: 15m ago
   Created: 24/11/2025 14:30:00

2. def456uvw
   Status: FAILED | RC: 07
   Product: AXIS5 → 081298765432
   SN: (none)
   Age: 1h 5m ago
   Created: 24/11/2025 13:25:00

💡 To manually send webhook:
   node manual-webhook.js <ref_id> <STATUS> [SN]

Example:
   node manual-webhook.js abc123xyz SUCCESS SN123456789
```

### 2. Send Manual Webhook

```bash
# SUCCESS with serial number
npm run webhook:send abc123xyz SUCCESS SN123456789

# FAILED without serial number
npm run webhook:send def456uvw FAILED

# PENDING (rare)
npm run webhook:send xyz789abc PENDING
```

**Full command:**

```bash
node manual-webhook.js <ref_id> <status> [serial_number]
```

## 📖 Usage Examples

### Example 1: Resend SUCCESS webhook

```bash
node manual-webhook.js dfj026mt04zxal20 SUCCESS SN987654321
```

**Output:**

```
🚀 MANUAL WEBHOOK SENDER
======================================================================
📋 ref_id: dfj026mt04zxal20
📊 status: SUCCESS
🔢 serial_number: SN987654321
🔗 webhook_url: https://api.digiflazz.com/v1/seller/callback
======================================================================

🔍 Step 1: Fetching transaction from database...
✅ Transaction found!
   Product: TELKOMSEL10
   Destination: 081234567890
   Price: Rp 10.450
   Supplier: SAGARA
   Current Status: PENDING
   Current RC: 39
   Webhook Sent Before: NO

🔄 Step 2: Mapping status to Digiflazz format...
   Digiflazz Status: "1" (SUCCESS)
   Response Code: "00"
   Message: "SUCCESS"

💾 Step 3: Updating transaction in database...
✅ Transaction updated successfully

📦 Step 4: Building webhook payload...
{
  "data": {
    "ref_id": "dfj026mt04zxal20",
    "status": "1",
    "code": "TELKOMSEL10",
    "hp": "081234567890",
    "price": "10450",
    "message": "SUCCESS",
    "balance": "10000000",
    "tr_id": "TRX1732451234567",
    "rc": "00",
    "sn": "SN987654321"
  }
}

🚀 Step 5: Sending webhook to Digiflazz...
   URL: https://api.digiflazz.com/v1/seller/callback
   ⏳ Waiting for response...

======================================================================
✅ WEBHOOK SENT SUCCESSFULLY!
======================================================================
📊 HTTP Status: 200
📨 Response Type: string
📨 Response Body:
   "FAIL!"

⚠️  NOTE: Digiflazz returns 'FAIL!' even for valid webhooks
    This is normal behavior. HTTP 200 = success.
======================================================================

💾 Step 6: Updating webhook status in database...
✅ webhook_sent flag updated to TRUE

======================================================================
✅ MANUAL WEBHOOK COMPLETED SUCCESSFULLY
======================================================================
✓ Transaction updated: dfj026mt04zxal20
✓ Status: SUCCESS
✓ RC: 00
✓ Serial Number: SN987654321
✓ Webhook sent to Digiflazz
✓ Database updated
======================================================================
```

### Example 2: Send FAILED webhook

```bash
node manual-webhook.js xyz123 FAILED
```

### Example 3: Using npm scripts

```bash
# Check pending first
npm run webhook:check

# Then send
npm run webhook:send ref123 SUCCESS SN999
```

## 📊 Status Options

| Status  | Digiflazz Code | RC   | Description                       |
| ------- | -------------- | ---- | --------------------------------- |
| SUCCESS | "1"            | "00" | Transaction berhasil              |
| FAILED  | "2"            | "07" | Transaction gagal                 |
| PENDING | "0"            | "39" | Transaction masih proses (jarang) |

## 🔍 Troubleshooting

### ❌ "Transaction not found"

**Problem:** ref_id tidak ada di database

**Solution:**

```sql
-- Connect to database
psql -h 13.75.122.160 -p 5100 -U user_topup -d eceran_topup_h2h

-- Check transaction
SELECT * FROM transactions WHERE ref_id = 'your_ref_id';

-- Check all recent transactions
SELECT ref_id, product_code, status, created_at
FROM transactions
ORDER BY created_at DESC
LIMIT 10;
```

### ❌ "Connection refused"

**Problem:** Webhook URL tidak bisa diakses

**Solution:**

1. Check environment variable:
   ```bash
   echo $DIGIFLAZZ_WEBHOOK_URL
   ```
2. Verify URL di `.env`:
   ```env
   DIGIFLAZZ_WEBHOOK_URL=https://api.digiflazz.com/v1/seller/callback
   ```
3. Test koneksi:
   ```bash
   curl -I https://api.digiflazz.com/v1/seller/callback
   ```

### ❌ "Invalid status"

**Problem:** Status tidak valid

**Valid options:**

- SUCCESS / SUKSES
- FAILED / GAGAL
- PENDING / PROCESS

### ⚠️ "FAIL!" response tapi HTTP 200

**Ini NORMAL!** Digiflazz selalu return "FAIL!" tapi HTTP 200 = success.

## 📝 Database Check

### Check webhook status

```sql
-- All unsent webhooks
SELECT ref_id, status, product_code, webhook_sent, created_at
FROM transactions
WHERE status IN ('SUCCESS', 'FAILED')
  AND (webhook_sent = false OR webhook_sent IS NULL)
ORDER BY created_at DESC;

-- Webhook history
SELECT
  ref_id,
  status,
  webhook_sent,
  webhook_timestamp,
  webhook_response->>'status' as http_status
FROM transactions
WHERE webhook_timestamp IS NOT NULL
ORDER BY webhook_timestamp DESC
LIMIT 20;

-- Count pending webhooks
SELECT COUNT(*) as pending_webhooks
FROM transactions
WHERE status IN ('SUCCESS', 'FAILED')
  AND (webhook_sent = false OR webhook_sent IS NULL);
```

### Manual database update (emergency)

```sql
-- Mark webhook as sent
UPDATE transactions
SET webhook_sent = true,
    webhook_timestamp = NOW()
WHERE ref_id = 'your_ref_id';

-- Reset webhook flag (to resend)
UPDATE transactions
SET webhook_sent = false,
    webhook_timestamp = NULL
WHERE ref_id = 'your_ref_id';
```

## 🔄 Workflow

### Production Workflow

1. Transaction masuk → Status PENDING
2. Supplier response → Update status (SUCCESS/FAILED)
3. **Auto webhook** di callback endpoint
4. Jika gagal → Check dengan `webhook:check`
5. Manual send dengan `webhook:send`

### Recovery Workflow

```bash
# 1. Check pending webhooks
npm run webhook:check

# 2. Copy ref_id dari output

# 3. Send webhook manual
npm run webhook:send <ref_id> <STATUS> [SN]

# 4. Verify
npm run webhook:check
```

## 🛠️ Environment Variables

```env
# Required
DIGIFLAZZ_WEBHOOK_URL=https://api.digiflazz.com/v1/seller/callback

# Database (for transaction lookup)
DB_HOST=13.75.122.160
DB_PORT=5100
DB_NAME=eceran_topup_h2h
DB_USER=user_topup
DB_PASSWORD=your_password
```

## 📊 Monitoring Commands

```bash
# Check pending webhooks
npm run webhook:check

# Watch for new pending webhooks (manual)
watch -n 10 "npm run webhook:check"

# Count pending
psql -h 13.75.122.160 -p 5100 -U user_topup -d eceran_topup_h2h \
  -c "SELECT COUNT(*) FROM transactions WHERE status IN ('SUCCESS','FAILED') AND webhook_sent = false;"
```

## 💡 Tips

1. **Always check database first** sebelum manual send
2. **Serial number optional** untuk FAILED transactions
3. **HTTP 200 = success** meskipun response "FAIL!"
4. **webhook_sent flag** otomatis update ke TRUE
5. **Run webhook:check regularly** untuk monitoring

## 🚨 Important Notes

- ⚠️ Webhook payload MUST use STRING for status, price, balance
- ⚠️ Payload must be wrapped in `data: {}` object
- ⚠️ Digiflazz always returns "FAIL!" - ignore this, check HTTP status
- ⚠️ Check `webhook_sent` flag di database untuk verify
- ⚠️ Tool ini aman dijalankan berkali-kali (idempotent)

---

**Version:** 1.0.0  
**Last Updated:** November 2025
