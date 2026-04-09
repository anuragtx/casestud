# MuleSoft Excel-to-CSV: Troubleshooting & Production Readiness Guide

---

## Table of Contents
1. [Common Issues & Solutions](#common-issues--solutions)
2. [Testing Checklist](#testing-checklist)
3. [Production Deployment Checklist](#production-deployment-checklist)
4. [Performance Tuning](#performance-tuning)
5. [Security Hardening](#security-hardening)
6. [Monitoring & Alerting](#monitoring--alerting)

---

## Common Issues & Solutions

### Issue 1: "BatchJob completed but no JSON files created"

**Symptoms:**
- Flow 1 runs without errors
- HTTP returns 200 OK
- But `/json_in/` folder is empty

**Possible causes:**

| Cause | Check | Fix |
|-------|-------|-----|
| Excel file has no data | Open Excel, verify sheets exist | Add test data to Excel |
| Batch aggregator size too large | If you have 50 records but `size="1000"` | Reduce size to 10-100 for testing |
| File write path doesn't exist | Verify `C:/mulesoft_demo/json_in/` exists | Create directory manually |
| No write permissions | Check folder properties | Grant write permissions to Mule user |

**Diagnosis steps:**
```
1. Enable DEBUG logging
   <logger level="DEBUG" message="Batch started with #[payload]" />
   
2. After line "Transform Excel to Java"
   Add: <logger level="DEBUG" message="After transform: #[payload]" />
   
3. Inside Batch Step
   Add: <logger level="DEBUG" message="Processing record: #[vars.record]" />
   
4. Check console output for actual data
```

---

### Issue 2: "HTTP 400 Bad Request when uploading Excel"

**Error response:**
```json
{
  "status": 400,
  "error": "TRANSFORMATION_ERROR",
  "message": "Data transformation failed.",
  "detail": "Cannot coerce value..."
}
```

**Cause:** Excel format issue or wrong file upload

**Debugging:**
```
1. Download sample file (customers.xlsx) from a trusted source
2. Test with that file
3. If works: Problem is with YOUR Excel file, not the flow
4. If doesn't work: Problem is with the flow

To check what MuleSoft receives:
<logger level="DEBUG" message="Received file: #[attributes.fileName] | Size: #[attributes.size] bytes" />
```

---

### Issue 3: "File Listener not detecting new JSON files"

**Symptoms:**
- Flow 1 writes JSON files successfully
- Flow 2 never triggers
- `/json_processed/` folder is empty

**Possible causes:**

| Cause | Check | Fix |
|-------|-------|-----|
| File Listener polling not active | Check Mule app is running | Start/restart Mule app |
| Wrong directory path | Verify `directory="..."` path | Check path exists and is readable |
| File listener scheduling disabled | Check flow is deployed | Redeploy or restart app |
| Files moved too fast | Race condition | Shouldn't happen, but add slight delay |

**Debugging:**
```xml
<!-- Add this BEFORE File Listener -->
<logger level="INFO" message="File Listener poll cycle started" />

<!-- Inside File Listener -->
<logger level="INFO" message="File detected: #[attributes.fileName]" />

<!-- Inside flow -->
<logger level="INFO" message="Flow 2 triggered for: #[attributes.fileName]" />
```

---

### Issue 4: "CSV file has wrong encoding or special characters are corrupted"

**Symptoms:**
- CSV opens in Excel but shows ??? instead of special characters
- French accents (é, è, ê) showing as garbage
- Chinese characters completely broken

**Cause:** Encoding mismatch

**Solution:**
```xml
<file:write 
  path="..." 
  mode="OVERWRITE"
  encoding="UTF-8"/>
```

**If problem persists:**
```xml
<!-- Add encoding to Transform step -->
<ee:transform>
  <ee:message>
    <ee:set-payload encoding="UTF-8"><![CDATA[%dw 2.0
output application/csv
---
payload]]></ee:set-payload>
  </ee:message>
</ee:transform>
```

---

### Issue 5: "Only 1 CSV file created instead of 50"

**Symptoms:**
- 50,000 rows uploaded
- Only `customers_143456.csv` created
- No other CSV files in `/csv_out/`

**Cause:** Batch aggregator never triggers multiple times

**Debug:**
```xml
<batch:on-complete>
  <logger level="INFO" 
    message='#["Batch completed: " ++ (payload.totalRecords / 1000) ++ " aggregations expected, totalRecords=" ++ payload.totalRecords]'/>
</batch:on-complete>
```

If `totalRecords = 0`:
```
Cause: Excel sheet was empty or payload[0] grabbed wrong sheet
Fix: Verify Excel has data on Sheet1
```

If `totalRecords = 50000` but only 1 CSV:
```
Cause: Aggregator triggered only once (data < 1000 records? No, that doesn't match)
Actually: There should be 50 files!
Check: Are they actually created but in wrong location?
```

---

### Issue 6: "Processing the same file twice (duplicate records)"

**Symptoms:**
- CSV has duplicate records
- Two entries for "John Doe" with same data
- Can trace back to same JSON file

**Cause:** File Listener didn't move file to archive after reading

**Check:**
```xml
<file:listener 
  moveToDirectory="C:\mulesoft_demo\json_processed"  <!-- Missing? -->
  >
```

**If moveToDirectory is missing:**
```xml
<!-- Before (broken): -->
<file:listener directory="C:/mulesoft_demo/json_in">

<!-- After (fixed): -->
<file:listener 
  directory="C:/mulesoft_demo/json_in" 
  moveToDirectory="C:\mulesoft_demo\json_processed">
```

---

### Issue 7: "Error Handler never catches the error, app crashes"

**Symptoms:**
- You upload Excel
- Transform fails
- App crashes with NullPointerException
- Error handler logs don't show up

**Cause:** Error handler not applied globally

**Check:**
```xml
<!-- Missing in <configuration>? -->
<configuration defaultErrorHandler-ref="excel-batch-integrationError_Handler"/>

<!-- Or error handler name doesn't match? -->
<error-handler name="excel-batch-integrationError_Handler">
```

**Fix:**
```xml
<configuration 
  doc:name="Configuration" 
  defaultErrorHandler-ref="excel-batch-integrationError_Handler"/>
```

---

## Testing Checklist

### Pre-Deployment Tests

- [ ] **Happy Path Test**
  ```
  Upload: Valid 100-row Excel file
  Expected: 
    ✅ HTTP 200 response
    ✅ JSON file created in /json_in/
    ✅ JSON file moved to /json_processed/
    ✅ CSV file created in /csv_out/
    ✅ All log entries present
  ```

- [ ] **Large File Test**
  ```
  Upload: 100,000-row Excel file
  Expected:
    ✅ Processing time < 2 minutes
    ✅ Memory usage stays below 4GB
    ✅ 100 JSON files created (size=1000)
    ✅ 100 CSV files created
  ```

- [ ] **Error Scenario Tests**
  ```
  Test 1: Upload corrupt Excel file
    Expected: HTTP 400 + Transform Error response
  
  Test 2: Delete /csv_out/ folder while Flow 2 running
    Expected: HTTP 500 + File Error response
  
  Test 3: Upload non-Excel file
    Expected: HTTP 400 + Transformation error
  ```

- [ ] **Concurrent Requests Test**
  ```
  Upload 5 Excel files simultaneously
  Expected:
    ✅ All processed successfully
    ✅ No data loss
    ✅ Filenames unique (no collisions)
    ✅ Performance degradation < 30%
  ```

- [ ] **File Cleanup Test**
  ```
  Delete manually created JSON/CSV files
  Expected:
    ✅ Flow 2 doesn't reprocess old files
    ✅ New uploads create fresh files
  ```

---

## Production Deployment Checklist

### Before Going Live

- [ ] **Folder Paths**
  ```
  □ C:\mulesoft_demo\json_in\ (exists, writeable)
  □ C:\mulesoft_demo\csv_out\ (exists, writeable)
  □ C:\mulesoft_demo\json_processed\ (exists, writeable)
  ```

- [ ] **File Permissions**
  ```
  □ Mule runtime user has write permission on all folders
  □ Network shares (if applicable) are mounted and accessible
  □ No "read-only" flags on folders
  ```

- [ ] **Configuration**
  ```
  □ Batch aggregator size=1000 (or tuned for your data)
  □ File write mode="OVERWRITE" (not APPEND)
  □ Error handler globally configured
  □ Port 8081 is available (or configured port)
  ```

- [ ] **Security**
  ```
  □ HTTPS enabled (TLS certificates installed)
  □ API authentication configured
  □ Firewall rules allow port 8081 (prod port)
  □ Sensitive data masked in logs
  ```

- [ ] **Monitoring**
  ```
  □ Log aggregation set up (ELK, Splunk, etc.)
  □ Disk space monitoring (CSV folder growth)
  □ Memory monitoring (batch processing spikes)
  □ Error rate alerts (> 5% failures per hour)
  ```

- [ ] **Deployment**
  ```
  □ Code review completed
  □ Performance tested with production-like data
  □ Runbook created (how to handle failures)
  □ Rollback plan documented
  ```

---

## Performance Tuning

### Batch Aggregator Size Optimization

**Current:** `size="1000"`

**Impact of different sizes:**

| Size | Files Created | RAM per file | I/O Operations | Recommendation |
|------|--------------|--------------|-----------------|-----------------|
| 100 | 500 | ~10MB | High | Too small |
| 500 | 100 | ~50MB | Medium | Small batches |
| **1000** | **50** | **100MB** | **Medium** | **Sweet spot** |
| 5000 | 10 | ~500MB | Low | Large batches |
| 10000 | 5 | ~1GB | Very low | Very large batches |

**Decision matrix:**
```
If data: Small files (< 1MB total)
  → Use size=100-500
  
If data: Medium files (100MB total)
  → Use size=1000 (default)
  
If data: Large files (1GB+ total)
  → Use size=5000-10000
  
If data: Huge files (100GB+)
  → Use size=50000+
```

**Test to find optimal:**
```
1. Start with size=1000
2. Monitor: memory usage, processing time, disk I/O
3. If memory spikes above 3GB: Reduce size
4. If too many files created: Increase size
5. Find balance where memory < 2GB and processing time < 60s
```

---

### Thread Pool Configuration

**Batch Job uses multiple threads:**
```
Default: 8-16 threads (depends on CPU cores)

Calculation:
Thread count ≈ CPU cores × 2

8-core CPU → 16 threads
4-core CPU → 8 threads
```

**How to tune:**
```xml
<batch:job>
  <threading-profile maxThreadsActive="16" />
  
  <!-- Process records in parallel with 16 threads -->
</batch:job>
```

**When to increase threads:**
- CPU usage < 50% during batch processing
- Waiting time is high (I/O bound)

**When to decrease threads:**
- Memory spikes dangerously
- CPU at 100% consistently

---

### File I/O Optimization

**Current approach:** Write to local disk (fastest)

**If too slow:**
```xml
<!-- Option 1: Batch multiple JSON writes together -->
<batch:aggregator size="5000">
  <!-- Fewer write operations, each larger -->
</batch:aggregator>

<!-- Option 2: Use SSD for file storage -->
<!-- OS-level: Move /mulesoft_demo to SSD mount point -->

<!-- Option 3: Use async write -->
<file:write 
  path="..." 
  mode="OVERWRITE"
  writeAsync="true"/>  <!-- Doesn't wait for write to complete -->
```

---

## Security Hardening

### 1. HTTPS (TLS) Configuration

**Current:** HTTP on port 8081 (insecure)

**Add HTTPS:**
```xml
<http:listener-config name="HTTP_Listener_config">
  <http:listener-connection 
    host="0.0.0.0" 
    port="8443"
    protocol="HTTPS">
    <tls:context>
      <tls:key-store 
        path="keystore.jks" 
        password="keystorePassword"
        keyPassword="keyPassword"/>
    </tls:context>
  </http:listener-connection>
</http:listener-config>
```

**Create keystore:**
```bash
keytool -genkey -alias mulesoft -keyalg RSA -keystore keystore.jks -keysize 2048
```

---

### 2. API Authentication

**Add API Key validation:**
```xml
<http:listener 
  path="/upload"
  doc:name="Listener">
  <http:response-validator status="200"/>
</http:listener>

<!-- After listener, before transform -->
<choice>
  <when expression="#[attributes.headers['X-API-Key'] == 'secret-key-12345']">
    <!-- Continue processing -->
  </when>
  <otherwise>
    <raise-error type="AUTHENTICATION_ERROR" 
      message="Invalid API Key" />
  </otherwise>
</choice>
```

**Better: Use OAuth 2.0**
```xml
<http:listener-config name="HTTP_Listener_config">
  <!-- OAuth 2.0 configuration -->
  <oauth2:validate-token 
    scopes="excel:upload"
    expectedAudience="mulesoft-app"/>
</http:listener-config>
```

---

### 3. Data Masking in Logs

**Current logs show full data (privacy risk):**
```
Log: "Processing record: {id: 123, email: john@company.com, ssn: 123-45-6789}"
```

**Mask sensitive fields:**
```xml
<logger level="INFO" 
  message='#["Processing record: " ++ 
    {
      id: record.id,
      email: (record.email replace "(?<=.{2}).*(?=.{2})" with "****"),
      ssn: "***-**-" ++ (record.ssn[-4..])
    }
  ]'/>
```

**Output:**
```
Log: "Processing record: {id: 123, email: jo****om, ssn: ***-**-6789}"
```

---

### 4. Input Validation

**Add file size limits:**
```xml
<choice>
  <when expression="#[attributes.size > 104857600]">  <!-- 100MB limit -->
    <raise-error type="FILE_SIZE_ERROR" 
      message="File too large. Maximum 100MB allowed" />
  </when>
</choice>
```

**Add file type validation:**
```xml
<choice>
  <when expression='#[attributes.fileName matches ".*\.(xlsx?|xls)$"]'>
    <!-- Proceed -->
  </when>
  <otherwise>
    <raise-error type="INVALID_FILE_TYPE" 
      message="Only .xlsx or .xls files allowed" />
  </otherwise>
</choice>
```

---

## Monitoring & Alerting

### Log Monitoring

**Search patterns:**
```
# All errors
level:ERROR

# Batch job failures
message:"BATCH WARNING"

# Specific correlation ID
correlationId:"abc-123-def"

# File write failures
message:"FILE ERROR"

# Errors in past hour
timestamp:[now-1h TO now]

# Alert if error rate > 5%
SELECT COUNT(*) FROM logs 
WHERE level="ERROR" 
GROUP BY 1h
HAVING COUNT(*) > errors_threshold
```

---

### Metrics to Track

| Metric | Alert Threshold | Action |
|--------|-----------------|--------|
| **Error Rate** | > 5% per hour | Page on-call engineer |
| **Processing Time** | > 2 minutes for 50K rows | Check CPU/disk usage |
| **Disk Usage** | > 80% of partition | Archive old CSV files |
| **Memory Usage** | > 4GB peak | Reduce batch size |
| **Failed Batch Records** | > 1% | Investigate data quality |

---

### Health Check Endpoint

```xml
<flow name="health-check">
  <http:listener path="/health" />
  
  <choice>
    <when expression='#[java.io.File("C:/mulesoft_demo/json_in").exists()]'>
      <set-payload value='{
        "status": "UP",
        "timestamp": now(),
        "components": {
          "json_input_folder": "OK",
          "csv_output_folder": "OK",
          "batch_processor": "OK"
        }
      }'/>
    </when>
    <otherwise>
      <set-payload value='{
        "status": "DOWN",
        "timestamp": now(),
        "error": "Required folders not found"
      }'/>
    </otherwise>
  </choice>
</flow>
```

**Test:**
```bash
curl http://localhost:8081/health
```

---

### Alerting Rules

**Set up alerts in ELK/Splunk:**

**Alert 1: Error Rate High**
```
IF count(level="ERROR") > 100 in last 5 minutes
THEN send email to ops@company.com
TITLE: "Excel-to-CSV API - High Error Rate"
```

**Alert 2: Batch Job Stuck**
```
IF timestamp(last "Batch Job completed" log) > 30 minutes ago
THEN send Slack message to #alerts
TITLE: "🚨 No batch jobs completed in 30 minutes"
```

**Alert 3: Disk Space Low**
```
IF disk_free < 10% on /csv_out partition
THEN send PagerDuty alert
TITLE: "Disk space critical on CSV output folder"
```

---

## Summary: Readiness Checklist

Before interviewer asks "Is this production-ready?":

✅ **Performance:** Tested with 100K+ rows  
✅ **Reliability:** Error handler catches all scenarios  
✅ **Security:** HTTPS + API key validation  
✅ **Monitoring:** Logs + metrics + alerts  
✅ **Recovery:** Tested failure scenarios + runbook  
✅ **Scalability:** Can process concurrent requests  
✅ **Documentation:** Code comments + runbook  

If asked in interview: "This is NOT production-ready YET. To make it production-ready, I would:
1. Add HTTPS encryption
2. Implement API authentication (OAuth)
3. Set up centralized logging
4. Add metrics dashboards
5. Create runbook for failure scenarios
6. Test at 2x expected volume
7. Set up automated alerts

All of these are standard enterprise requirements, not project-specific issues."

---

**Good luck! You've got all the knowledge you need! 💪**
