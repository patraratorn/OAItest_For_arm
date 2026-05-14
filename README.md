# O-RAN E2SM-KPM Metrics Extraction (OAI & FlexRIC)

Repository นี้รวบรวมไฟล์ Source Code และ Configuration ที่เกี่ยวข้องกับการดึงค่า L1 Metrics (เช่น RSRP, SNR, Timing Advance, และ Beam Index) จาก OAI gNB ผ่าน E2SM-KPM ไปยัง xApp (FlexRIC) และส่งต่อให้ Python Controller เพื่อทำ Closed-Loop Control

เนื่องจากโปรเจกต์นี้พัฒนาบนพื้นฐานของ **OAI (OpenAirInterface)** หากคุณใช้งาน **srsRAN** คุณสามารถใช้ไฟล์ใน Repository นี้เป็น **Reference** เพื่อทำความเข้าใจโครงสร้างของ xApp และแนวทางในการดัดแปลง Source Code ของ gNB ให้สามารถส่ง Metrics ที่ต้องการออกมาได้

## โครงสร้างของ Repository

### 1. `src/xApp/` (ส่วนของ xApp บน FlexRIC)
*   **`xapp_kpm_moni.c`**: โค้ดภาษา C สำหรับ xApp ที่ทำหน้าที่ Subscribe และดึงค่า KPM Metrics จาก gNB 
    *   **สิ่งที่น่าสนใจ:** โค้ดนี้ถูกแก้ไขบั๊กเรื่อง Memory Allocation (`calloc`) ให้รองรับ Metrics จำนวนมากขึ้น และ **เปลี่ยนรูปแบบการ Subscribe เป็น Format 1 (No Filter)** เพื่อแก้ปัญหาที่ xApp มองไม่เห็น UE หาก srsRAN ของคุณเจอปัญหาคล้ายกัน แนะนำให้ลองศึกษาการตั้งค่า Subscription ในไฟล์นี้

### 2. `src/controller/` (ส่วนของ Python Controller)
*   **`ro_controller.py`**: สคริปต์ Python ที่รับค่า Metrics จาก C-xApp นำมาประมวลผลก่อนส่งให้ AI
    *   **สิ่งที่น่าสนใจ:** มี Logic การแก้ปัญหา (Workaround) สำหรับค่า RSRP ที่ส่งมาจาก C-xApp ในรูปแบบ 32-bit Unsigned Integer ให้กลับเป็นค่า Signed Integer (ค่าติดลบ) อย่างถูกต้อง เช่น `value = raw_value - 4294967296`

### 3. `config/` (ส่วนของ Configuration)
*   **`gnb.sa.band78.fr1.106PRB.usrpb210.conf`**: ไฟล์ตั้งค่าของ OAI gNB 
    *   **สิ่งที่น่าสนใจ:** คุณสามารถดูค่าพารามิเตอร์ต่างๆ เช่น PLMN (MCC=001, MNC=01, TAC=1), Bandwidth (106 PRB), และการเปิดใช้งาน Multi-beam (`ssb_PositionsInBurst_Bitmap = 15`) เพื่อนำไปเป็นแนวทางในการตั้งค่าไฟล์ `gnb.yml` ของ srsRAN ให้ตรงกัน เพื่อให้สามารถเชื่อมต่อกับ 5GC และ FlexRIC ตัวเดียวกันได้

### 4. `oai_patches/` (ส่วนดัดแปลง OAI Source Code)
*   **`ran_func_kpm.c`** และ **`ran_func_kpm_subs.c`**: โค้ด 2 ไฟล์นี้มีความสำคัญมาก เป็นการเจาะเข้าไปใน MAC Layer ของ OAI เพื่อดึงตัวแปรภายในออกมาสร้างเป็น E2SM-KPM Metrics ตัวใหม่ (เช่น `DRB.UE.RSRP`, `DRB.UE.SNR`, `DRB.UE.TA`, `DRB.UE.BeamIdx`)
    *   **คำแนะนำสำหรับผู้ใช้ srsRAN:** srsRAN อาจจะยังไม่ได้รองรับ Metrics เหล่านี้แบบ Default ทาง E2SM-KPM คุณสามารถดูโค้ด 2 ไฟล์นี้เพื่อทำความเข้าใจว่า "ต้องดึงค่าอะไร (เช่น `pusch_pc` สำหรับ SNR, `cumul_rsrp` สำหรับ RSRP)" และนำแนวคิดนี้ไปประยุกต์ใช้ในการดัดแปลง Source Code ของ srsRAN เพื่อให้ส่งค่าที่เหมือนกันออกมา

## หมายเหตุ
*   โปรดตรวจสอบเวอร์ชันของ FlexRIC และ E2SM-KPM ที่ srsRAN รองรับ เนื่องจากโครงสร้างของ ASN.1 อาจมีความแตกต่างกัน
*   การพัฒนา xApp ด้วยภาษา C บน FlexRIC สามารถนำไปใช้งานร่วมกับ srsRAN ได้ตราบใดที่ gNB รองรับ Service Model (SM) เดียวกัน (เช่น E2SM-KPM)