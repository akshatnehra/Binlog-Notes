# Binlog File Structure

Ek baar ki baat hai, ek DBA (Database Administrator) apne MySQL server ke logs ko check kar raha tha. Usne dekha ki binlog files ka size kaafi badh gaya hai, aur uske man mein sawaal aaya, "Ye binlog files kaise store hote hain? Inki internal structure kya hai?" Usne socha ki agar binlog files ko samajhna hai, to usse replication aur disaster recovery ke liye behtar planning karni hogi. To aaj hum usi DBA ki tarah binlog file structure ko step-by-step explore karenge, aur ye samajhenge ki ye "diary" jaise files kaise kaam karti hain, aur MySQL engine ke andar inka code-level implementation kaisa hai.

Binlog, yaani Binary Log, ek aisi file hoti hai jo MySQL server ke har transaction ya event ka record rakhti hai. Ye ek diary ki tarah hai, jisme aap har din ka hisaab likhte ho – kab kya hua, kitna kharcha kiya, kitna aaya. Isi tarah, binlog file mein database ke changes (INSERT, UPDATE, DELETE) aur structural changes (CREATE TABLE, ALTER TABLE) ka record hota hai, jo replication aur recovery ke liye use hota hai. Lekin ye diary sirf likha nahi jata, balki iski ek khas structure hoti hai, jise hum aaj detail mein samajhenge.

## Binlog File ka Physical Structure

Chalo pehle binlog file ki physical structure ko samajhte hain. Ye file teen main hisson mein banti hai – **Header**, **Events**, aur **Footer** (haalaanki footer ka concept thoda implicit hai). Is structure ko samajhna ek ghar ke blueprint ko samajhne jaisa hai – aapko pata hona chahiye ki diwaar kahan hai, khidki kahan, aur darwaza kahan.

### Header
Binlog file ka sabse pehla hissa hota hai header. Ye ek chhota sa section hota hai, jo file ke shuruaat mein hota hai aur isme magic number aur format version jaisi basic information hoti hai. Magic number ek unique identifier hota hai, jo ye confirm karta hai ki ye file sach mein ek binlog file hai. Iske alawa, format version batata hai ki binlog ka format version kya hai (jaise MySQL 5.0 se 8.0 tak alag format ho sakte hain). Header ka size fixed hota hai, aur ye sirf 4 bytes ka hota hai in modern MySQL versions.

Header ke baad aata hai ek **Format Description Event**, jo ek special event hota hai. Ye event binlog file ke metadata ke baare mein information deta hai, jaise ki binlog format, server version, aur event types jo is file mein ho sakte hain. Ye event har binlog file ke shuruaat mein hota hai, aur iske bina MySQL binlog ko sahi tarah se parse nahi kar sakta.

### Events
Ab hum aate hain binlog file ke dil pe – **Events**. Ye wo actual records hote hain jo database mein hone wale changes ko store karte hain. Har event ek chhoti si entry hoti hai, jisme ek header aur body hota hai. Event header mein timestamp, event type, server ID, aur event ka size hota hai. Event body mein actual data hota hai, jaise ki koi query (agar statement format hai) ya row changes (agar row format hai).

Har event ka ek type hota hai, jaise `WRITE_ROWS_EVENT`, `UPDATE_ROWS_EVENT`, `DELETE_ROWS_EVENT`, etc. Ye types MySQL ke code mein define hote hain, aur hum inhe `sql/log_event.h` mein dekh sakte hain. Events sequential order mein likhe jate hain, aur ek event ke baad doosra event shuru hota hai. Isiliye binlog file ko ek stream of events ke roop mein dekha ja sakta hai.

### Footer
Binlog file mein koi explicit footer nahi hota, lekin jab file rotate hoti hai ya server shutdown hota hai, to last event ke baad koi naya data nahi likha jata. Iske alawa, agar binlog checksum enabled hai (MySQL 5.6 se), to har event ke end mein checksum value hota hai, jo data integrity check karne ke liye use hota hai.

**Edge Case**: Agar binlog file corrupt ho jaye, to MySQL us file ko parse karne mein fail ho sakta hai. Is case mein, aapko `mysqlbinlog` tool se file ko analyze karna hoga aur last valid event tak data recover karna hoga. Ye process thoda tedious hai, lekin disaster recovery ke liye kaafi useful hai.

## Binlog File Naming Convention

Ab hum baat karte hain binlog file ke naamkaran (naming convention) ki. MySQL mein binlog files ka naam ek specific pattern follow karta hai, jaise `binlog.000001`, `binlog.000002`, aur aage bhi. Ye naam server ke configuration file (`my.cnf`) mein define kiye base name (`log_bin`) pe depend karta hai. Agar aapne `log_bin = mysql-bin` set kiya hai, to files ka naam `mysql-bin.000001` jaisa hoga.

Numbering system sequential hota hai, aur har file ka number increment hota hai jab file rotate hoti hai. Ye rotation manually (`FLUSH LOGS` command) ya automatically (jab file size limit cross hota hai) ho sakta hai. Configuration parameter `max_binlog_size` ye control karta hai ki ek binlog file kitni badi ho sakti hai.

**Desi Analogy**: Ye naming system ek school ke roll number jaisa hai. Jaise class mein har student ko ek unique number milta hai (1, 2, 3...), waise hi binlog files ko bhi number milte hain, taki server ko pata rahe ki kaun si file pehle aayi aur kaun si baad mein.

**Use Case**: Agar aap replication set up kar rahe ho, to master aur slave ke beech binlog file name aur position ka sync hona zaroori hai. Agar slave kisi binlog file ko nahi dhoondh pata, to replication fail ho jayega. Isiliye, binlog file naming ko samajhna DBA ke liye critical hai.

## Binlog File Rotation

Binlog files ka rotation ek important concept hai. Jab ek binlog file ka size `max_binlog_size` se zyada ho jata hai, ya jab aap `FLUSH LOGS` command chalate ho, to MySQL ek nayi binlog file create karta hai aur purani file ko close kar deta hai. Ye process file rotation kehlata hai.

Rotation ke dauraan, MySQL pehle current binlog file ko close karta hai, aur phir ek nayi file create karta hai jisme agla sequence number hota hai. Is process mein koi data loss nahi hota, kyunki events sequential order mein likhe jate hain. Rotation ke baad, server nayi file mein events likhna shuru kar deta hai.

**Command Example**:
```sql
FLUSH LOGS;
```
Ye command binlog file ko rotate kar deta hai. Agar aap `SHOW BINARY LOGS;` chalate ho, to aapko saari binlog files ki list mil jayegi.

**Edge Case**: Agar server crash ho jaye aur binlog file properly close na ho, to last few events corrupt ho sakte hain. Is case mein, `mysqlbinlog` tool se aapko last valid event tak replay karna hoga, aur corrupt data ko skip karna hoga.

## Internal Components: LOG_EVENT Struct

Ab hum thoda deep dive karte hain aur MySQL ke source code mein binlog ke internal components ko dekhte hain. MySQL ke code mein binlog events ko `LOG_EVENT` struct ke roop mein define kiya gaya hai. Ye struct `sql/log_event.h` file mein hota hai, aur ismein events ke saare details store hote hain.

**Code Analysis** (based on `sql/log.cc`):
`sql/log.cc` file mein binlog ke writing aur rotation ke logic ko implement kiya gaya hai. Is file mein functions jaise `MYSQL_BIN_LOG::write_event` aur `MYSQL_BIN_LOG::rotate` define hote hain. Chalo ek snippet dekhte hain:

```c
int MYSQL_BIN_LOG::write_event(Log_event *event_info) {
  // Write event header
  // Write event body
  // Flush to disk if needed
}
```

Upar wala code snippet dikhata hai ki `write_event` function ek event ko binlog file mein kaise likhta hai. Pehle event ka header likha jata hai, jisme timestamp aur event type hota hai. Phir event ka body likha jata hai, jisme actual data hota hai (jaise query text ya row data). Agar `sync_binlog` parameter set hai, to har event ke baad data disk pe flush kiya jata hai, jo durability ensure karta hai.

**Troubleshooting**: Agar binlog write fail ho jaye, to MySQL error log mein entry aayegi. Aapko disk space, file permissions, aur `sync_binlog` settings check karni hogi. Performance ke liye, `sync_binlog=0` set karna better hai, lekin isse crash ke case mein data loss ka risk hota hai.

> **Warning**: Binlog file ko manually edit ya delete nahi karna chahiye, kyunki ye replication aur recovery process ko break kar sakta hai. Agar aapko purani binlog files delete karni hai, to `PURGE BINARY LOGS` command use karo.

## Comparison of Binlog Formats

| Format       | Pros                                      | Cons                                      |
|--------------|-------------------------------------------|-------------------------------------------|
| Statement    | Small size, readable queries             | Unsafe for non-deterministic queries      |
| Row          | Accurate for replication                 | Larger file size due to row data          |
| Mixed        | Balances statement and row format        | Complex to analyze                        |

Upar ki table mein binlog formats ka comparison hai. **Statement format** mein queries as text store hoti hain, jo chhoti files banati hai, lekin non-deterministic queries (jaise `NOW()`) ke saath issue ho sakte hain. **Row format** mein har row ka data store hota hai, jo accurate hota hai, lekin file size bada ho jata hai. **Mixed format** dono ka balance try karta hai.