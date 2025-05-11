# Binlog Rotation aur Purging

Ek baar ki baat hai, jab ek MySQL server ek bade online shopping platform ke liye kaam kar raha tha, toh usne dekha ki uski binlog files ka size din-ba-din badh raha hai. Ye binlog files uske liye ek diary ki tarah thi, jahan har transaction aur change record kiya ja raha tha. Lekin ek din, disk space khatam hone ke kareeb aa gaya, aur server ko samajh mein aaya ki purani entries ko saaf karna aur nayi files mein likhna zaroori hai. Yahi hai binlog rotation aur purging ki kahani – ek aisa process jo MySQL ke liye disk space manage karne aur performance maintain karne ke liye critical hai.

Is chapter mein hum binlog rotation aur purging ke concept ko detail mein samajhenge. Hum dekheinge ki binlog files kab aur kaise rotate hoti hain, purani files ko kaise purge kiya jata hai, aur is process ke piche ka engine code kaise kaam karta hai. Ye sab ek beginner-friendly style mein hoga, lekin 'Database Internals' book jaisa deep aur technical bhi rahega. Chalo, pehle thoda analogy se samajhte hain – binlog rotation matlab jaise ek clerk apni purani ledger book ko band karke ek nayi book shuru karta hai, aur purging matlab woh purani books jo ab kaam ki nahi, unhe store se nikaal deta hai.

## Binlog Rotation: Nayi File Mein Shift Karna

Binlog rotation ka matlab hai ki MySQL ek active binlog file ko band karke ek nayi binlog file create karta hai, aur usmein transactions ko log karna shuru karta hai. Yeh process automatically ya manually trigger ho sakta hai. Socho, jaise ek dukaandaar apni roz ki hisaab kitaab ek register mein likhta hai, aur jab woh register bhar jaata hai, toh woh ek naya register shuru karta hai. Binlog rotation bhi aisa hi hai – MySQL ke liye yeh ensure karta hai ki ek hi file ka size uncontrolled na ho jaye, aur server pe load na pade.

### Kab Hota Hai Binlog Rotation?
Binlog rotation kai situations mein hota hai. Chalo inhe detail mein dekhte hain:

- **Size Limit Cross Hone Par**: MySQL mein ek variable hota hai `max_binlog_size`, jo define karta hai ki ek binlog file ka maximum size kya hona chahiye. Default value generally 1GB hoti hai. Jab active binlog file ka size is limit ko cross kar jaata hai, toh MySQL ek nayi binlog file create karta hai. Yeh ensure karta hai ki disk pe ek hi file bohut badi na ho jaye, warna file handling slow ho sakti hai.
  
- **Server Restart**: Jab MySQL server restart hota hai, toh woh bhi binlog rotation trigger kar deta hai. Isse nayi binlog file shuru hoti hai, taaki server ke fresh state ke saath logs bhi fresh ho. Yeh bohut helpful hota hai troubleshooting ke liye, kyunki restart ke baad ke logs alag se track kiye ja sakte hain.

- **Manual Trigger**: DBAs `FLUSH BINARY LOGS` command ka use karke manually bhi binlog rotation trigger kar sakte hain. Yeh command active binlog file ko close karke ek nayi file shuru karta hai. Is tarah ke manual intervention aksar backup ya maintenance ke dauraan kiya jaata hai.

### Rotation Ke Piche Ka Technical Mechanism
Ab thoda engine internals ki baat karte hain. MySQL ke binlog rotation ka core logic `sql/binlog.cc` file mein defined hai. Yeh file binlog management ke liye responsible hoti hai. Jab binlog rotation trigger hota hai, toh MySQL internally `MYSQL_BIN_LOG::new_file()` function ko call karta hai. Yeh function ek nayi binlog file create karta hai aur uska name generally `binlog.000001`, `binlog.000002` jaise sequence mein hota hai. Har file ke saath ek index number badh jaata hai.

Yeh dekho ek code snippet from `binlog.cc` jo binlog file creation ko handle karta hai (GitHub Reader Tool se liya gaya hai):

```c
int MYSQL_BIN_LOG::new_file(int flags)
{
  // Logic to close current file and open a new one
  close(LOG_CLOSE_TO_BE_OPENED | LOG_CLOSE_INDEX);
  if (open(get_log_fname(NULL), LOG_BIN))
    return 1;
  return 0;
}
```

Is code mein dekho, `close()` function current binlog file ko close karta hai, aur `open()` function ek nayi binlog file ko shuru karta hai `LOG_BIN` flag ke saath. Yeh process disk pe nayi file create karta hai aur index file mein iski entry update karta hai. Isse MySQL ko pata rahta hai ki kaunsi binlog file active hai aur kaunsi purani hai.

### Edge Cases aur Troubleshooting
Binlog rotation ke dauraan kuch unexpected scenarios bhi ho sakte hain. For example, agar disk space full ho jaye aur nayi binlog file create na ho paye, toh MySQL error throw karta hai aur transactions log nahi ho paate. Aise case mein DBA ko disk space free karna padta hai ya `max_binlog_size` ko temporarily reduce karna padta hai. Ek aur edge case hai permissions – agar MySQL server ko binlog directory mein write permission na ho, toh rotation fail ho jaata hai. Iske liye `log_bin` variable ki directory permissions check karni hoti hai.

## Binlog Purging: Purani Files Ko Saaf Karna

Binlog purging ka matlab hai purani binlog files ko delete karna, taaki disk space free ho sake. Socho, jaise ek office clerk purani ledgers jo ab kaam ki nahi, unhe store se nikaal deta hai, taaki naye ledgers ke liye jagah bane. MySQL mein binlog purging bohut zaroori hai, kyunki agar purani files ko delete na kiya jaye, toh disk space bhar jaata hai, aur server performance degrade hoti hai.

### Purging Kaise Hota Hai?
Purging do tarah se ho sakta hai – automatic aur manual. Chalo detail mein dekhte hain:

- **Automatic Purging**: MySQL mein ek variable hota hai `binlog_expire_logs_seconds`, jo define karta hai ki binlog files kitne time tak rakhi jayengi. Default value generally 30 din hoti hai (2592000 seconds). Jab koi binlog file is expiry time se purani ho jaati hai, toh MySQL use automatically purge kar deta hai. Yeh process background mein chalta hai aur server ke liye transparent hota hai.

- **Manual Purging**: DBAs `PURGE BINARY LOGS TO 'binlog.000123'` command ka use karke specific binlog file tak sab purani files delete kar sakte hain. Ya phir `PURGE BINARY LOGS BEFORE '2023-Runs 10-01 00:00:00'` command se kisi specific date/time tak ki files delete ki ja sakti hain. Yeh manual purging aksar jab disk space emergency ho, tab kiya jaata hai.

### Purging Ke Piche Ka Engine Logic
Purging ka logic bhi `sql/binlog.cc` mein defined hota hai. Jab purging trigger hota hai, toh MySQL `MYSQL_BIN_LOG::purge_logs()` function ko call karta hai. Yeh function binlog index file se purani entries ko read karta hai aur unhe filesystem se delete kar deta hai. Yeh process ensure karta hai ki binlog index aur actual files hamesha sync mein rahe.

Yeh dekho ek relevant code snippet `binlog.cc` se:

```c
bool MYSQL_BIN_LOG::purge_logs(const char *to_log, bool included,
                               bool need_lock_index,
                               bool need_update_threads,
                               ulonglong *previous_ref_count,
                               bool purge_descendant)
{
  // Logic to identify and delete old binlog files
  // Updates index file and frees disk space
}
```

Is code mein dekho, `purge_logs()` function binlog files ko identify karta hai jo delete karni hai, aur phir unhe filesystem se remove kar deta hai. Yeh process index file ko bhi update karta hai taaki MySQL ko pata rahe ki kaunsi files ab available nahi hai.

### Edge Cases aur Warnings
Purging ke dauraan bhi kuch issues ho sakte hain. For example, agar koi binlog file replication slave pe abhi read nahi hui ho, aur master pe purge ho jaye, toh replication break ho sakta hai. Isliye manual purging se pehle hamesha slave status check karna chahiye.

> **Warning**: Binlog purging se pehle ensure karo ki replication slaves ne purani binlog files ko process kar liya hai, warna replication lag ya break ho sakta hai. `SHOW SLAVE STATUS` command se relay log position check karo.

## Key Configurations aur Functions

Binlog rotation aur purging ko control karne ke liye kuch key system variables aur functions hote hain. Inhe detail mein samajhna zaroori hai:

- **`log_bin`**: Yeh variable binlog ko enable ya disable karta hai. Agar yeh set nahi hai, toh binlog files create nahi hoti.
- **`max_binlog_size`**: Yeh define karta hai ki ek binlog file ka maximum size kya hoga. Default 1GB hota hai.
- **`binlog_expire_logs_seconds`**: Yeh binlog files ke expiry time ko set karta hai. Default 30 din hota hai.
- **`MYSQL_BIN_LOG::new_file()`**: Yeh function nayi binlog file create karta hai.
- **`MYSQL_BIN_LOG::purge_logs()`**: Yeh function purani files ko delete karta hai.

## Comparison of Approaches

| **Approach**              | **Pros**                           | **Cons**                           |
|---------------------------|------------------------------------|------------------------------------|
| **Automatic Purging**     | Background mein chalta hai, no manual effort | Expiry time set karna padta hai, accidental delete ho sakta hai |
| **Manual Purging**        | Full control DBA ke pass       | Time consuming, replication break ka risk |

Automatic purging beginners ke liye better hai kyunki yeh maintenance burden kam karta hai, lekin experienced DBAs manual purging prefer karte hain taaki unhe full control mile. Agar replication ya point-in-time recovery ka use ho raha hai, toh manual purging se pehle backups aur slave status ka dhyan rakhna zaroori hai.
