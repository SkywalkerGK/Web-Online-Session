# Web-Online-Session
Flow Web Online Session เว็บรายงานสรุปประสิทธิภาพของการใช้งานออนไลน์รายเดือน 

ActiveSub คือ จำนวนผู้ใช้งานที่สมัครสมาชิกและใช้งานระบบอยู่จริงในขณะนั้น โดยนับจากผู้ที่มีสถานะการสมัครสมาชิก (Subscription) ที่ยังไม่หมดอายุและสามารถล็อกอินใช้งานได้
CONCURRENT (Concurrent Users) คือ จำนวนผู้ใช้งานที่เข้าใช้ระบบพร้อมกันในเวลาเดียวกัน

Flow การทำงานหลังบ้าน 

1. ข้อมูลในส่วนของกราฟแท่ง และ ตาราง (CONCURRENT,Old Active Sub)

- Job Oracle TT_RADIUS

01-APR-20 11.00.00.000000000 PM +07:00    (วันที่ 1 เวลา 5 ทุ่ม)
TT_RADIUS.PROC_ACTIVE_SUB_CNT  -------->> TRUNCATE TBL_ACTIVE_SUB_CNT และ INSERT DATEOPER,TYPE_NAME,MEDIA,NASIPADDRESS,BRAS,ACTIVESUB 

01-APR-20 11.20.00.000000000 PM +07:00   (วันที่ 1 เวลา 5 ทุ่ม 20 นาที)
TT_RADIUS.PROC_ACTIVE_SUB_UPDATE  ------>> UPDATE ค่า  CONCURRENT ใน TT_RADIUS.TBL_ACTIVE_SUB_CNT

ค่า ACTIVESUB กับ CONCURRENT จะอยู่ใน Table  :  TT_RADIUS.TBL_ACTIVE_SUB_CNT

- Server ODA

(วันที่ 2 เวลา เที่ยงคืน 20 นาที)
20 00 2 * * sh /u01/oracle/DB1RP/MAPUSER/export_active_sub.sh      ------>>  Export CSV จาก TT_RADIUS.TBL_ACTIVE_SUB_CNT  (PROC_ACTIVE_SUB_CNT_EXPORT('MY_DIR','active_sub.csv'))

(วันที่ 2 เวลา เที่ยงคืน 30 นาที)
30 00 2 * * sh /u01/oracle/DB1RP/MAPUSER/transferActiveSub206.sh     ------>>  FTP to 206  active_sub.csv

- Web Server / Shell Server 206

_file_path="/home/mapuser/active_sub.csv";  -- ไฟล์จะมาตอน วันที่ 2 เวลา 00:30

(วันที่ 2 เวลา ตี 1)
00 01 2 * * sh /home/mapuser/activesub_imp.sh    --insert into server database mysql icb active_sub_cnt

(วันที่ 2 เวลา ตี 1 15 นาที)
15 01 2 * * sh /home/mapuser/update_active_sub.sh   --update SORT By Media and Package


Web Server 206 ดึงข้อมูลไปแสดงผลเป็นกราฟ กับ ตาราง

__________________________________01-01-2024 แก้เคสค่า Active Sub ผิดปกติ__________________________________________

2. ข้อมูลในส่วนของตาราง (New Active Sub ทดแทนกรณีมีการถ่ายโอนลูกค้าจาก Node บ้านส้มไป Node บ้านเขียว โดยที่ไม่กระทบ Flow เดิม)
   **ตามบ้านเขียวจะมีการแบ่งพื้นที่ต่างจากของบ้านส้ม จาก RO เป็น Region และ Bras เป็น RC**

- Job Oracle TT_MIMS

(วันที่ 1 เวลา 7 โมง 10 นาที) 
TT_MIMS.PROC_EXPORT_REP_ACTIVESUB   -- Export CSV to ODA

- Server ODA

cd /backup/expdp/arch199

ls -ltrh |grep 'rep_activesub'

ตัวอย่างชื่อไฟล์   rep_activesub_tgroup_202410.csv

ให้ insert ใน server database mysql  icb  active_sub_t_group

กรณีที่หน่วยงานอื่นต้องการข้อมูลล่าสุดของเดือนปัจจุบันให้ใช้ Query นี้ Export

select to_char(trunc(add_months(sysdate,0),'MM'),'DD/MM/YYYY HH24:MI:SS') as DATEOPER,
    a2.T_GROUP,a2.BRAS,a2.rc,a2.region,count(*) as ACTIVESUB from (

        select cus.username,cus.offering_name,cus.offering_price,cus.t_group, '-' as media,
            'RO-'||cus.zone_id as bras,r.rc,r.region  
        from tt_mims.tbl_cust3bbactive cus
        left join tt_mims.tbl_map_region r on cus.province_name = r.province_th2
        where cus.t_group is not null
 
    ) a2                
    group by trunc(add_months(sysdate,0),'MM') ,a2.T_GROUP,a2.BRAS,a2.rc,a2.region;
