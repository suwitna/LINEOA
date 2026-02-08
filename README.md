# LINEOA
LINE OA MACHINE ALERT SYSTEM

ภาพรวม Tool & File ที่ใช้

ลำดับ	งาน	Tool / Platform  

1.	Database	SSMS
2.	Scheduler	SQL Server Agent
3.	Script	PowerShell (.ps1)
4.	Messaging API	LINE Developers
5.	LINE Official Account	LINE OA Manager
6.	Client	LINE App (Mobile)

---

# Job 1: Check Machine Stop (Producer)


<img width="496" height="214" alt="image" src="https://github.com/user-attachments/assets/0643594b-db4a-43e4-beb3-2d36f18548c6" />


## หน้าที่
- ตรวจสอบเครื่องจักรหยุดเกิน 30 นาที
- แล้วสร้าง Alert ลง Queue


## ไฟล์ที่ใช้
- Check-MachineStop.ps1
- Tool:  PowerShell
- รันโดย:  SQL Server Agent


## ขั้นตอนการทำงาน
- Connect SQL Server
- Query MachineEventLog
- ตรวจ Stop > 30 นาที
- Insert ข้อมูลลง `LineAlertQueue`

---


# Job 2: Send LINE Notification (Consumer)

<img width="578" height="161" alt="image" src="https://github.com/user-attachments/assets/5d9af6a4-363e-4a0b-92fb-60e40de1b4cd" />


## หน้าที่:
- อ่าน Alert จาก Queue
- ส่ง LINE
- Update Status

## ไฟล์ที่ใช้
- Send-LineOA.ps1`
- Tool: PowerShell
- รันโดย: SQL Server Agent

## ขั้นตอนการทำงาน
- Read `LineAlertQueue`
- Read `LineRecipients`
- Call LINE Messaging API
- Update Status (SENT / FAILED)

---
---

# ขั้นตอนการพัฒนา Job 1 และ Job 2
## Job 1: Check Machine Stop (Producer)
---
### เป้าหมาย
- ตรวจสอบว่าเครื่องจักร หยุดเกิน 30 นาที
- แล้ว สร้าง Alert ลง Queue โดยไม่แจ้งซ้ำ

---

### Step 1: เตรียม Database Structure
#### 1.1 ตรวจสอบ Table ต้นทาง
ข้อมูลลักษณะนี้:
MachineEventLog
-	machine
-	status_name
-	start_time
โดยข้อมูล  event log ต้องเรียงตามเวลา
#### 1.2 สร้าง Table Queue

```sql
CREATE TABLE dbo.LineAlertQueue (
    QueueID BIGINT IDENTITY PRIMARY KEY,
    AlertType NVARCHAR(50),
    Machine NVARCHAR(50),
    StatusName NVARCHAR(50),
    StartTime DATETIME,
    StopMinutes INT,
    Message NVARCHAR(MAX),
    AlertStatus NVARCHAR(20) DEFAULT 'NEW',
    RetryCount INT DEFAULT 0,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    SentAt DATETIME2 NULL,
    LastError NVARCHAR(4000)
);
```
#### 1.3 ป้องกัน Alert ซ้
ำ
```sql
CREATE UNIQUE INDEX UX_Machine_Stop_Alert
ON dbo.LineAlertQueue (Machine, AlertType, AlertStatus)
WHERE AlertStatus = 'NEW';
```

---
### Step 2: พัฒนา Logic ตรวจสอบ (SQL)
2.1 เขียน Query ตรวจเครื่องล่าสุด

```sql
WITH LatestStatus AS (
    SELECT
        machine,
        status_name,
        start_time,
        DATEDIFF(MINUTE, start_time, GETDATE()) AS StopMinutes,
        ROW_NUMBER() OVER (PARTITION BY machine ORDER BY start_time DESC) AS rn
    FROM dbo.MachineEventLog
)
SELECT *
FROM LatestStatus
WHERE rn = 1;
```

*** ตรวจว่าดึงเครื่องละ 1 record จริง
  
2.2 เพิ่มเงื่อนไข Stop > 30 นาที

```sql
AND status_name = 'Stop'
AND StopMinutes >= 30
```

2.3 เช็คไม่ให้ Insert ซ้ำ

```sql
AND NOT EXISTS (
    SELECT 1
    FROM dbo.LineAlertQueue q
    WHERE q.Machine = LatestStatus.machine
      AND q.AlertType = 'MACHINE_STOP'
      AND q.AlertStatus IN ('NEW','SENT')
)
```
---
### Step 3: Insert Queue

```sql
INSERT INTO dbo.LineAlertQueue (...)
SELECT ...
```

#### ทดสอบ:
- รันซ้ำ → ต้องไม่ Insert ซ้ำ
- เปลี่ยนเครื่องกลับ RUN → ต้อง Insert ใหม่ได้ในอนาคต

---
### Step 4: สร้าง PowerShell Script (Job 1)
#### ไฟล์: Check-MachineStop.ps1
#### สิ่งที่ต้องมี:
- Connect SQL
- Execute Insert Query
- Log Error
#### ทดสอบ:
- Manual run ผ่าน PowerShell
- ตรวจ Queue มีข้อมูล NEW

---
### Step 5: สร้าง SQL Agent Job (Job 1)
- Type: PowerShell
- Schedule: ทุก 10 นาที
- Owner: SQL Agent Service Account
- Enable History

#### ทดสอบ:
- Run Job manually
- ดู Job History → Success

---
## Job 2 : Send LINE Notification (Consumer)
- เป้าหมาย
- อ่าน Alert จาก Queue
- ส่ง LINE
- Update สถานะ

---
### Step 6: เตรียม Table ผู้รับ LINE

```sql
CREATE TABLE dbo.LineRecipients (
    RecipientID INT IDENTITY PRIMARY KEY,
    LineUserID NVARCHAR(100),
    IsActive BIT DEFAULT 1
);

```

---
### Step 7: เตรียม LINE OA
- สร้าง LINE OA
- เปิด Messaging API
- Generate Channel Access Token
- เก็บ Token อย่างปลอดภัย (Credential Manager / Encrypted file)

<img width="496" height="378" alt="image" src="https://github.com/user-attachments/assets/e6c2526b-9a29-44da-bc9f-4cc479eeaa48" />


<img width="499" height="376" alt="image" src="https://github.com/user-attachments/assets/6ee83242-fc0f-4354-a2e3-b81b51997439" />


<img width="545" height="421" alt="image" src="https://github.com/user-attachments/assets/7b852007-bae3-4a4e-8f9f-39c518d60e46" />


<img width="541" height="412" alt="image" src="https://github.com/user-attachments/assets/bcf074dc-28be-4d81-8a4c-6780ef52a06d" />

#### Web hook:
```
https://docs.google.com/spreadsheets/d/***/edit?gid=0#gid=0
 ```
<img width="602" height="289" alt="image" src="https://github.com/user-attachments/assets/c967f5ca-b2a8-44fd-85b3-ca2261293245" />


#### Web hook URL:
```
https://script.google.com/macros/s/***/exec
```

---
### Step 8: พัฒนา PowerShell Script (Job 2)
#### ไฟล์: Send-LineOA.ps1
#### Logic:
- Get AlertStatus = NEW
- Get LineRecipients
- Loop ส่ง LINE
- Update Status
#### สิ่งที่ต้องมี:
- UTF-8 Encoding
- try / catch
- RetryCount
- Error Logging

---
### Step 9: Test Script แยกเดี่ยว
#### 9.1 Test ไม่มี Alert
- Script ต้อง exit แบบ clean
#### 9.2 Test Alert สำเร็จ
- AlertStatus → SENT
- SentAt มีค่า
#### 9.3 Test LINE Fail
- AlertStatus → FAILED
- RetryCount +1

### Step 10: สร้าง SQL Agent Job (Job 2)
- Type: PowerShell
- Schedule: ทุก 1 นาที
- Enable retry step
- Enable Output file


### Step 11: End-to-End Test
#### Scenario:
- เครื่องหยุดจริง
- Job 1 → Insert Queue
- Job 2 → ส่ง LINE
- ตรวจ Log / Table

---
# Checklist ก่อนขึ้น Production
- Queue ไม่ซ้ำ
- LINE Fail ไม่ทำให้ข้อมูลหาย
- Job History เปิด
- Token ไม่ Hardcode
- มี Log

### Table: MachineEventLog

<img width="638" height="506" alt="image" src="https://github.com/user-attachments/assets/8c2eed39-91b7-449d-922d-b51c523f65e2" />


### Table: LineRecipients

<img width="600" height="331" alt="image" src="https://github.com/user-attachments/assets/65d134fd-f699-450d-8583-239e628278cc" />


### Tab: LineAlertQueue

<img width="602" height="295" alt="image" src="https://github.com/user-attachments/assets/e0f70be2-5b57-467e-951d-14cb3d684273" />


 ```
รัน Job 1:powershell.exe -ExecutionPolicy Bypass -File "D:\SSRS\2568\3\LINE_OA\Check-MachineStop.ps1"
```

<img width="602" height="69" alt="image" src="https://github.com/user-attachments/assets/00eb4371-4eaf-4e50-b933-f08fbd4a7607" />


```
รัน Job 2:powershell.exe -ExecutionPolicy Bypass -File "D:\SSRS\2568\3\LINE_OA\Send-LineOA.ps1"
```

<img width="590" height="171" alt="image" src="https://github.com/user-attachments/assets/255abd4e-c44d-45e9-be8d-18e422d6edf3" />

 
<img width="602" height="56" alt="image" src="https://github.com/user-attachments/assets/8916723e-b0c9-4296-b0f7-e48b863bc1ba" />


<img width="398" height="595" alt="image" src="https://github.com/user-attachments/assets/c7b36555-38ac-4878-8f38-8eba3a66957a" />

---

### SQL Agent Job


<img width="441" height="412" alt="image" src="https://github.com/user-attachments/assets/8ad14848-d0d1-4029-8564-7fdb3001e05c" />



<img width="466" height="410" alt="image" src="https://github.com/user-attachments/assets/fe9f726c-cd48-4afe-8cc7-f8feb399af86" />

---

# 1. Send-LineOA.ps1

```ps
#############################################
# Send-LineOA.ps1  (Production Safe Version)
#############################################

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# -----------------------------
# CONFIG
# -----------------------------
$ConnStr = "Server=localhost;Database=****;User ID=***;Password=****;Integrated Security=True;Encrypt=True;TrustServerCertificate=True;"
$AccessToken = "***"
$LineUrl = "https://api.line.me/v2/bot/message/push"

# -----------------------------
# SQL FUNCTIONS
# -----------------------------
function Exec-Sql {
    param(
        [string]$Query,
        [hashtable]$Params
    )

    $conn = New-Object System.Data.SqlClient.SqlConnection($ConnStr)

    try {
        $conn.Open()
        $cmd  = $conn.CreateCommand()
        $cmd.CommandText = $Query

        if ($Params) {
            foreach ($k in $Params.Keys) {
                $null = $cmd.Parameters.AddWithValue($k, $Params[$k])
            }
        }

        $cmd.ExecuteNonQuery()
    }
    catch {
        Write-Host "SQL Execute Error: $($_.Exception.Message)"
        throw
    }
    finally {
        if ($conn.State -eq 'Open') {
            $conn.Close()
        }
    }
}

function Get-Data {
    param([string]$Query)

    $conn = New-Object System.Data.SqlClient.SqlConnection($ConnStr)
    $dt = New-Object System.Data.DataTable

    try {
        $conn.Open()
        $cmd  = $conn.CreateCommand()
        $cmd.CommandText = $Query

        $da = New-Object System.Data.SqlClient.SqlDataAdapter($cmd)
        $da.Fill($dt) | Out-Null
    }
    catch {
        Write-Host "SQL Query Error: $($_.Exception.Message)"
        return $null
    }
    finally {
        if ($conn.State -eq 'Open') {
            $conn.Close()
        }
    }

    return $dt
}

# -----------------------------
# Split-Message
# -----------------------------
function Split-Message {
    param(
        [string]$Text,
        [int]$ChunkSize = 2000
    )

    $chunks = @()
    for ($i = 0; $i -lt $Text.Length; $i += $ChunkSize) {
        $length = [Math]::Min($ChunkSize, $Text.Length - $i)
        $chunks += $Text.Substring($i, $length)
    }
    return $chunks
}

# -----------------------------
# 1️⃣ LOAD RECIPIENTS
# -----------------------------
$users = Get-Data "
    SELECT LineUserID
    FROM dbo.LineRecipients
    WHERE IsActive = 1
"

if ($null -eq $users -or $users.Count -eq 0) {
    Write-Host "No active LINE recipients"
    exit 0
}

# -----------------------------
# 2️⃣ LOAD NEW ALERTS
# -----------------------------
$alerts = Get-Data "
    SELECT QueueID, Message
    FROM dbo.LineAlertQueue
    WHERE AlertStatus = 'NEW'
    ORDER BY CreatedAt
"

if ($null -eq $alerts -or $alerts.Count -eq 0) {
    Write-Host "No NEW alerts"
    exit 0
}

# -----------------------------
# 3️⃣ SEND LINE + UPDATE STATUS (New)
# -----------------------------

# Combine all alert messages into a single string, separated by newlines.
$allMessages = ($alerts | ForEach-Object { $_.Message }) -join "`n---`n"

# Split the text into bubbles, each containing up to 5,000 characters.
$messageChunks = Split-Message -Text $allMessages -ChunkSize 2000

foreach ($u in $users) {

    # Create a messages array for LINE.
    $messagesArray = @()
    foreach ($chunk in $messageChunks) {
        $messagesArray += @{
            type = "text"
            text = $chunk
        }
    }

    $body = @{
        to = $u.LineUserID
        messages = $messagesArray
    } | ConvertTo-Json -Depth 5

    try {
        Invoke-RestMethod `
            -Uri $LineUrl `
            -Method Post `
            -Headers @{
                "Authorization" = "Bearer $AccessToken"
                "Content-Type"  = "application/json"
            } `
            -Body $body
    }
    catch {
        $errMsg = $_.Exception.Message
        if ($errMsg.Length -gt 3500) { $errMsg = $errMsg.Substring(0,3500) }

        foreach ($a in $alerts) {
            Exec-Sql `
                "UPDATE dbo.LineAlertQueue
                 SET AlertStatus = 'FAILED',
                     RetryCount = RetryCount + 1,
                     LastError = @err
                 WHERE QueueID = @qid" `
                @{
                    "@err" = $errMsg
                    "@qid" = [int]$a.QueueID
                }
        }

        break
    }
}

# -----------------------------
# UPDATE SENT STATUS
# -----------------------------
foreach ($a in $alerts) {
    Exec-Sql `
        "UPDATE dbo.LineAlertQueue
         SET AlertStatus = 'SENT',
             SentAt = SYSDATETIME()
         WHERE QueueID = @qid" `
        @{
            "@qid" = [int]$a.QueueID
        }
}

#############################################
# END
#############################################

```

# 2. Check-MachineStop.ps1

```ps
#############################################
# Check-MachineStop.ps1
#############################################

$SqlServer = "localhost"
$Database  = "***"
$ConnStr   = "Server=$SqlServer;Database=$Database;Integrated Security=True;Encrypt=True;TrustServerCertificate=True;"

$query = @"
WITH LatestStatus AS (
    SELECT
        machine,
        status_name,
        start_time,
        DATEDIFF(MINUTE, start_time, GETDATE()) AS StopMinutes,
        ROW_NUMBER() OVER (PARTITION BY machine ORDER BY start_time DESC) AS rn
    FROM dbo.MachineEventLog
)
INSERT INTO dbo.LineAlertQueue (
    AlertType,
    Machine,
    StatusName,
    StartTime,
    StopMinutes,
    Message
)
SELECT
    'MACHINE_STOP',
    machine,
    status_name,
    start_time,
    StopMinutes,
    CONCAT(
        '⚠️ Machine Stop Alert', CHAR(10),
        'Machine: ', machine, CHAR(10),
        'Stopped: ', StopMinutes, ' minutes', CHAR(10),
        'Since: ', CONVERT(VARCHAR, start_time, 120)
    )
FROM LatestStatus
WHERE rn = 1
  AND status_name = 'Stop'
  AND StopMinutes >= 30
  AND NOT EXISTS (
        SELECT 1
        FROM dbo.LineAlertQueue q
        WHERE q.Machine = LatestStatus.machine
          AND q.AlertType = 'MACHINE_STOP'
          AND q.AlertStatus IN ('NEW','SENT')
  );
"@

Invoke-Sqlcmd -ConnectionString $ConnStr -Query $query

#############################################
# END
#############################################

```
---

# Test script
## Dummy Data

```sql
INSERT INTO [dbo].[LineAlertQueue]
(
    [AlertType],
    [Machine],
    [StatusName],
    [StartTime],
    [StopMinutes],
    [Message],
    [AlertStatus],
    [RetryCount],
    [CreatedAt],
    [SentAt],
    [LastError]
)
VALUES
(
    'MACHINE_STOP',
    'CNC-MAZ-2XN-010',
    'Stop',
    '2025-09-03 08:30:15.000',
    153024,
    '?? Machine Stop AlertMachine: CNC-MAZ-2XN-010Stopped: 153024 minutesSince: 2025-09-03 08:30:15',
    'NEW',
    0,
    '2025-12-18 14:54:32.612',
    NULL,
    NULL
),
(
    'MACHINE_STOP',
    'CNC-MAZ-3XN-029',
    'Stop',
    '2025-09-03 08:17:28.000',
    153037,
    '?? Machine Stop AlertMachine: CNC-MAZ-3XN-029Stopped: 153037 minutesSince: 2025-09-03 08:17:28',
    'NEW',
    0,
    '2025-12-18 14:54:32.612',
    NULL,
    NULL
);

```

### Job 1:

```ps
powershell.exe -ExecutionPolicy Bypass -File "D:\SSRS\2568\3\LINE_OA\Check-MachineStop.ps1"
```

### Job 2:

```ps
powershell.exe -ExecutionPolicy Bypass -File "D:\SSRS\2568\3\LINE_OA\Send-LineOA.ps1"
```
