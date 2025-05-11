# Crash Simulation Test Procedure

Bhai, socho ek DBA (Database Administrator) hai jo apne MySQL server pe kaam kar raha hai. Ek din usse khabar milti hai ki uska database crash ho gaya aur ab usse ye check karna hai ki uska **binlog** (binary log) safe hai ya nahi. Binlog matlab ek aisa record jo har database transaction ko track karta hai, jaise ek bank ka transaction ledger. Agar crash ke baad binlog corrupt ho gaya, toh data recovery mushkil ho sakti hai. Toh is DBA ne socha, "Kyun na ek crash simulation test kar loon aur ye dekhoon ki binlog kitna durable hai?" Ye kahani hai crash simulation test ki, jahan hum artificially crash create karte hain aur dekhte hain ki binlog aur MySQL ka engine kaise behave karta hai.

Is chapter mein hum samajhenge ki crash simulation test kya hota hai, kaise kiya jata hai, aur binlog ki integrity ko crash ke baad kaise verify karte hain. Hum desi analogies ka use karenge taki concepts beginners ko bhi aasani se samajh aa sakein, lekin saath hi saath hum MySQL ke engine internals aur code ki depth mein bhi jayenge. Chalo, shuru karte hain!

## Kya Hai Crash Simulation Test Aur Kyun Zaruri Hai?

Crash simulation test matlab ek aisa experiment jahan hum jaan-boojhkar MySQL server ko crash kar dete hain taki ye dekhein ki uske critical components, jaise binlog, crash ke baad bhi safe aur reliable rehte hain ya nahi. Ye test bilkul waise hi hai jaise car manufacturers apni gaadiyon ka safety test karte hain - jaan-boojhkar accident simulate karte hain taki ye pata chale ki airbag, seatbelt sab kaam kar rahe hain ya nahi. Database ke context mein, binlog ek bahut important part hai kyunki ye replication aur recovery ke liye use hota hai. Agar crash ke baad binlog mein kuch galat ho gaya, toh replication slaves sync nahi rahenge aur data loss ka risk bhi ban sakta hai.

Binlog durability test karna zaruri hai kyunki real-world mein hardware failure, power outage, ya software bugs ki wajah se crash ho sakta hai. MySQL ke engine mein binlog ko kaise handle kiya jata hai, ye samajhna bhi zaroori hai. Binlog events ko disk pe write karne ke liye MySQL ke pass specific mechanisms hote hain, jaise `sync_binlog` aur `innodb_flush_log_at_trx_commit` settings, jo control karti hain ki data kitna safe hai. Hum age chalke iski technical details mein jayenge, lekin pehle ye samajh lete hain ki crash simulation ka basic process kya hota hai.

### Crash Simulation Ka Basic Concept

Crash simulation mein hum MySQL server ko forcibly terminate karte hain (jaise `kill -9` command se) ya hardware failure simulate karte hain. Isse hum dekhte hain ki binlog file mein jo events write ho rahi thi, wo corrupt toh nahi ho gayi? Ya phir incomplete transactions ka kya hua? MySQL ke InnoDB engine aur binlog ke coordination ko test karna isliye zaroori hai kyunki crash ke waqt dono ko sync mein hona chahiye. Agar sync nahi hai, toh recovery ke waqt problem ho sakti hai.

## Tools Aur Commands Jo Crash Simulation Ke Liye Use Hote Hain

Ab hum practical taraf chalte hain. Crash simulation ke liye humein kuch tools aur commands ki zarurat hoti hai. Ye tools aur commands humein MySQL server ko crash karne, monitor karne, aur phir binlog ki integrity check karne mein help karte hain. Chalo inhe step-by-step dekhte hain.

### 1. MySQL Server Ko Crash Karne Ke Liye Commands
- **`kill -9 <pid>`**: Ye command MySQL server ke process ko immediately terminate kar deti hai, jo ek hard crash simulate karta hai. Ye command use karne se MySQL ko gracefully shutdown hone ka mauka nahi milta, aur hum dekhte hain ki binlog aur InnoDB logs ka kya hota hai.
- **Power Failure Simulation**: Agar hum physical server pe kaam kar rahe hain, toh hum power cord nikal kar ya VM mein power off karke crash simulate kar sakte hain. Ye real-world failure ko simulate karta hai.

### 2. Binlog Aur System Ko Monitor Karne Ke Tools
- **`mysqlbinlog`**: Ye tool binlog files ko read karne ke liye use hota hai. Crash ke baad hum isse check karte hain ki binlog events readable hain ya corrupt ho gaye.
- **`strace` aur `gdb`**: Ye debugging tools humein ye samjha sakte hain ki crash ke waqt MySQL process kya kar raha tha. Ye advanced users ke liye helpful hote hain.
- **System Logs**: `/var/log/mysql/error.log` ya system ke logs (`dmesg`) check karne se pata chalta hai ki crash ka reason kya tha aur uske baad kya hua.

### 3. Configuration Settings Jo Test Ke Liye Adjust Karne Chahiye
- **`sync_binlog`**: Ye setting control karti hai ki binlog events ko kitni baar disk pe sync kiya jaye. Agar iski value `1` hai, toh har transaction ke baad sync hota hai, jo safe hai lekin slow. Crash simulation ke waqt hum ise alag-alag values pe test kar sakte hain.
- **`innodb_flush_log_at_trx_commit`**: Ye setting InnoDB log buffer ko disk pe flush karne ko control karti hai. Crash test ke waqt iski values (`0`, `1`, `2`) ko tweak karke dekha ja sakta hai ki binlog aur InnoDB logs ka sync kaise behave karta hai.

## Step-by-Step Crash Simulation Test Procedure

Ab hum actual steps pe aate hain ki kaise crash simulation test kiya jata hai. Ye steps practical hain aur inhe follow karke aap apne MySQL server pe crash test kar sakte hain. Hum har step ko detail mein samajhenge.

### Step 1: Test Environment Setup Karo
Pehle ek test environment banayein. Production server pe crash test karna dangerous hai, isliye ek separate test server ya VM use karo. Is test server pe MySQL install karo aur binlog enable karo (`log_bin` setting ko set karo). Ek sample database banayein aur uspe dummy transactions chalaayein, taki binlog mein events record hon.

```sql
SET GLOBAL log_bin = 'mysql-binlog';
CREATE DATABASE test_crash;
USE test_crash;
CREATE TABLE dummy_data (id INT PRIMARY KEY, value VARCHAR(100));
INSERT INTO dummy_data VALUES (1, 'test1'), (2, 'test2');
```

### Step 2: Transaction Load Generate Karo
Ab kuch load generate karo taki MySQL actively kaam kare. Hum ek simple script bana sakte hain jo continuous inserts ya updates karta rahe.

```sql
DELIMITER //
CREATE PROCEDURE load_test()
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= 1000 DO
    INSERT INTO dummy_data (id, value) VALUES (i, CONCAT('test_', i));
    SET i = i + 1;
  END WHILE;
END //
DELIMITER ;
CALL load_test();
```

### Step 3: Crash Simulate Karo
Jab transactions chal rahe hon, tab MySQL process ko kill kar do. Pehle MySQL ka process ID (PID) find karo aur phir `kill -9` command use karo.

```bash
ps aux | grep mysqld
kill -9 <mysqld_pid>
```

Ye hard crash simulate karega. Ab MySQL server restart karo aur dekho ki kya hota hai.

### Step 4: Binlog Integrity Verify Karo
Crash ke baad, MySQL restart hone pe error log check karo (`/var/log/mysql/error.log`) aur dekho ki koi corruption ya issue report hua hai ya nahi. Phir `mysqlbinlog` tool use karke binlog file read karo.

```bash
mysqlbinlog mysql-bin.000001 > binlog_output.txt
```

Agar binlog file readable hai aur transactions proper record hue hain, toh binlog durable hai. Agar error aata hai ya file corrupt hai, toh configuration ya setup mein kuch issue ho sakta hai.

## Practical Example: Ek Real Crash Test

Chalo ab ek practical example dekhte hain. Maan lo humne ek MySQL server setup kiya hai jahan `sync_binlog=0` set hai, matlab binlog events imediatamente disk pe nahi likhe jate. Humne load test chalaya aur beech mein server ko `kill -9` se crash kar diya. Restart ke baad humne dekha ki binlog file mein last few transactions missing hain kyunki wo sync nahi hue the. Ye example samjhaata hai ki `sync_binlog` setting ka asar crash recovery pe padta hai.

Ab humne `sync_binlog=1` set kiya aur same test repeat kiya. Is baar crash ke baad bhi binlog mein saare transactions the, kyunki har transaction ke baad sync ho raha tha. Lekin is setting se performance slow ho gayi. Ye trade-off samajhna important hai - safety vs speed.

> **Warning**: `sync_binlog=0` ya low durability settings production mein use karna dangerous ho sakta hai kyunki crash ke waqt data loss ka risk hota hai. Hamesha apne use case ke hisaab se settings tweak karo aur crash simulation test karke confirm karo.

## Binlog Handling Ka Code Internals - `sql/log.cc` Se Analysis

Ab hum MySQL ke source code ki taraf chalte hain aur dekhte hain ki binlog kaise handle kiya jata hai. MySQL ke source code mein `sql/log.cc` file binlog writing aur management ke liye responsible hai. Humne GitHub se is file ka content fetch kiya hai aur ab iska deep analysis karenge.

### `sql/log.cc` Mein Binlog Writing Ka Mechanism
`sql/log.cc` file mein binlog events ko write karne ka logic defined hai. Is file mein `MYSQL_BIN_LOG` class hoti hai jo binlog operations handle karti hai. Ek important function hai `write_event()`, jo binlog event ko file mein write karta hai.

```cpp
bool MYSQL_BIN_LOG::write_event(Log_event *event_info) {
  // Write the event to the binlog file
  // Check if sync is needed based on sync_binlog setting
  if (sync_binlog_period && ++sync_counter >= sync_binlog_period) {
    sync_counter = 0;
    sync();
  }
  return 0;
}
```

Ye snippet dikhata hai ki `sync_binlog` setting ke basis pe binlog file ko disk pe sync kiya jata hai. Agar `sync_binlog=1` hai, toh har event ke baad sync hota hai, jo crash ke waqt data loss prevent karta hai. Lekin ye operation I/O intensive hai aur performance pe asar dalta hai.

### Crash Ke Waqt Kya Hota Hai?
Crash ke waqt, agar binlog file sync nahi hui hoti, toh OS ke file buffer mein pending data lost ho sakta hai. `sql/log.cc` mein sync operation `fsync()` system call ke through hota hai, jo ensure karta hai ki data disk pe physically write ho gaya hai. Agar crash ke waqt `fsync()` nahi hua, toh binlog incomplete ho sakti hai.

## Comparison of Binlog Durability Settings

| **Setting**               | **Value** | **Durability**           | **Performance**         | **Crash Risk**                |
|---------------------------|-----------|--------------------------|-------------------------|-------------------------------|
| `sync_binlog`            | 0         | Low (no sync)            | High (fast)            | High (data loss possible)     |
| `sync_binlog`            | 1         | High (sync every event)  | Low (slow due to I/O)  | Low (data safe)               |
| `innodb_flush_log_at_trx_commit` | 0 | Low (buffered)           | High (fast)            | High (data loss possible)     |
| `innodb_flush_log_at_trx_commit` | 1 | High (sync every commit) | Low (slow)             | Low (data safe)               |

Ye table dikhata hai ki durability aur performance mein trade-off hota hai. Crash simulation test ke through aap apne environment ke liye best settings choose kar sakte ho.

## Conclusion Aur Troubleshooting Tips

Crash simulation test karna MySQL ke binlog durability ko samajhne aur improve karne ka ek powerful tareeka hai. Is test ke through hum pata laga sakte hain ki kaunse settings safe hain aur kaunse risk introduce karte hain. Agar crash ke baad binlog corrupt ho jaye, toh pehle error logs check karo, `mysqlbinlog` se file read karne ki koshish karo, aur agar problem hai toh backup se restore karne ka plan banayein.

Ye chapter ek beginner-friendly intro tha crash simulation test ka, lekin saath hi MySQL ke internals aur code analysis ke saath deep dive bhi kiya. Umeed hai ki ye content aapko binlog durability aur crash recovery samajhne mein help karega.