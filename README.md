# Ubuntu YouTube Music → FAT32 / Car Audio Workflow

บันทึกงานจริงสำหรับสร้างคลังเพลงจาก YouTube จัดอัลบั้ม/แยกศิลปิน สำรองอุปกรณ์เดิม และคัดลอกเพลงลงสื่อสำหรับเครื่องเสียงรถยนต์บน Ubuntu

> หลักประจำงาน: **ทวนก่อนทำเสมอ — Identify → Backup → Verify → Change → Measure → Isolate → Fix → Verify again**
>
> งาน storage ไม่เน้นลีนจนเสี่ยงผิดพลาด โดยเฉพาะก่อน Format / Delete / Move ต้องรู้ให้แน่ว่าอุปกรณ์ไหนคือเป้าหมายและมี Backup ที่ตรวจสอบแล้ว

## 1. ดาวน์โหลดเฉพาะเพลงที่ยังไม่เคยดาวน์โหลด

เก็บประวัติด้วย archive เพื่อไม่โหลดรายการเดิมซ้ำ

    yt-dlp -x --audio-format mp3 --download-archive downloaded.txt "<URL_PLAYLIST>"

ควรเก็บ YouTube video ID ไว้ในชื่อไฟล์ เพราะใช้ย้อนกลับไปดึง metadata ได้ในภายหลัง

## 2. ตรวจ metadata ก่อนจัดอัลบั้ม

    exiftool -Artist -Album -Title *.mp3 | head -80

ถ้า MP3 ไม่มี Artist/Album ที่เชื่อถือได้ แต่ชื่อไฟล์มี [VIDEO_ID] ให้ใช้ yt-dlp ดึงข้อมูลจากต้นทาง แล้วจัดโฟลเดอร์ตาม Artist; หากไม่มี Artist ค่อย fallback เป็น uploader

ข้อสังเกต: metadata จาก MV / Official Audio ไม่ได้เป็น album metadata ที่สม่ำเสมอ จึงเหมาะกับการแยก **ศิลปิน** มากกว่าฝืนแยก album

## 3. ก่อนแตะอุปกรณ์ปลายทาง: Identify ให้ชัด

ห้ามเดาจากชื่อที่เรียกอุปกรณ์ ให้ระบบบอกว่า device จริงคืออะไร

    lsblk -o NAME,TRAN,RM,SIZE,FSTYPE,LABEL,MOUNTPOINTS

ตรวจซ้ำให้รู้แน่ว่า partition ใดเป็นข้อมูล, Windows, Ubuntu หรือสื่อที่จะใช้กับเครื่องเสียงรถ

**ห้าม Format ทั้ง disk เช่น /dev/nvme0n1 ถ้าเป้าหมายจริงเป็นเพียง partition เช่น /dev/nvme0n1p6**

## 4. สำรองของเดิมก่อน Format

ตัวอย่าง:

    mkdir -p "/home/gwagk/Desktop/สำรองงานป้า"
    rsync -avh --progress "/run/media/gwagk/DATA I/" "/home/gwagk/Desktop/สำรองงานป้า/"

อย่าเชื่อว่า copy สำเร็จเพียงเพราะคำสั่งจบ ให้ตรวจจำนวนไฟล์และขนาดทั้งสองฝั่ง

    find "/run/media/gwagk/DATA I" -type f | wc -l
    du -sh "/run/media/gwagk/DATA I"

    find "/home/gwagk/Desktop/สำรองงานป้า" -type f | wc -l
    du -sh "/home/gwagk/Desktop/สำรองงานป้า"

ตรวจเนื้อหาด้วย checksum dry-run อีกชั้น:

    rsync -avnc --delete "/run/media/gwagk/DATA I/" "/home/gwagk/Desktop/สำรองงานป้า/"

ถ้าไม่มีรายการแตกต่าง จึงค่อยไปขั้น Format

## 5. FAT32 สำหรับเครื่องเสียงรถยนต์

เมื่อยืนยัน device และ Backup แล้ว จึง unmount และ format **เฉพาะ partition เป้าหมาย**

ตัวอย่างจากงานจริง:

    udisksctl unmount -b /dev/nvme0n1p6
    sudo mkfs.vfat -F 32 -n MUSIC /dev/nvme0n1p6
    udisksctl mount -b /dev/nvme0n1p6

FAT32 เลือกเพื่อ compatibility กับเครื่องเสียงรถยนต์หลายรุ่น ข้อจำกัดไฟล์เดี่ยว 4 GiB ไม่เป็นปัญหาสำหรับ MP3 ทั่วไป

## 6. คัดลอกคลังเพลง

สำหรับ FAT32 ไม่จำเป็นต้องพยายามรักษา Linux owner/group/permissions

    rsync -rvh --progress \
      "/home/gwagk/Desktop/เพลงแยกศิลปิน/" \
      "/run/media/gwagk/MUSIC/"

ถ้าพบ `rsync code 23` อย่าก๊อบใหม่แบบเดาสุ่ม ให้หาว่าอะไรหายจริง

## 7. Measure: นับเพลงต้นทางกับปลายทาง

    echo "=== ต้นทาง ==="
    find "/home/gwagk/Desktop/เพลงแยกศิลปิน" -type f -iname "*.mp3" | wc -l

    echo "=== MUSIC ==="
    find "/run/media/gwagk/MUSIC" -type f -iname "*.mp3" | wc -l

งานจริงรอบ 2026-10-04:
- ต้นทาง 434 MP3
- รอบแรกปลายทาง 427 MP3
- ขาด 7 เพลง

## 8. Isolate: หาเฉพาะไฟล์ที่หาย

อย่าใช้ timestamp เป็นตัวตัดสินกับ FAT32 โดยไม่จำเป็น เพราะความแตกต่างของ timestamp semantics อาจทำให้ rsync รายงานไฟล์จำนวนมากทั้งที่ไฟล์มีอยู่แล้ว

ตรวจ existence โดยตรง:

    cd "/home/gwagk/Desktop/เพลงแยกศิลปิน"

    find . -type f -iname "*.mp3" -print0 |
    while IFS= read -r -d '' f; do
        [ -f "/run/media/gwagk/MUSIC/$f" ] || printf '%s\n' "$f"
    done

งานจริงพบ 7 เพลงที่หายอยู่ใน 4 directory ซึ่งชื่อมี **space ต่อท้าย**:
- `Winzt Thor Official `
- `ดัง พันกร - DK OFFICIAL `
- `BDLMD `
- `Jumbo1991 `

Linux ยอมรับชื่อแบบนี้ แต่ FAT32/เครื่องมือที่ทำงานกับ FAT มีข้อจำกัดเรื่องชื่อที่ลงท้ายด้วย space จึงควร sanitize ชื่อก่อน copy

หลังตัด trailing space แล้ว sync เฉพาะของที่ยังไม่มี:

    rsync -rvh --ignore-existing \
      "/home/gwagk/Desktop/เพลงแยกศิลปิน/" \
      "/run/media/gwagk/MUSIC/"

ผลตรวจสุดท้าย: **434 / 434 เพลง**

## 9. ปิดงานอย่างปลอดภัย

    sync
    udisksctl unmount -b /dev/nvme0n1p6

รอให้ unmount สำเร็จก่อนถอดอุปกรณ์

## Checklist ก่อนทำทุกครั้ง

1. **Identify** — ตรวจ device/partition จริงด้วย lsblk
2. **Backup** — สำรองของเดิมก่อน Format/Delete/Move
3. **Verify** — เทียบจำนวน ขนาด และ checksum เมื่อเหมาะสม
4. **Change** — ทำ destructive action เฉพาะ target ที่ยืนยันแล้ว
5. **Measure** — เทียบจำนวนไฟล์ต้นทาง/ปลายทาง
6. **Isolate** — ถ้ามี error หาเฉพาะรายการที่ผิด ไม่แก้เหมา
7. **Fix** — แก้ต้นเหตุ เช่นชื่อ directory ที่ FAT32 รับไม่ได้
8. **Verify again** — รอบนี้ต้องได้ 434/434
9. **sync + unmount** — จึงถือว่างานจบ

## บทเรียน

ปัญหาใหญ่ไม่จำเป็นต้องมีสาเหตุใหญ่ รอบนี้ `rsync code 23` ถูกบีบจากข้อมูล 3.63 GB → 434 เพลง → ขาด 7 เพลง → 4 โฟลเดอร์ → trailing space

**ไม่เดา • ไม่แก้เหมา • วัดก่อน • แก้เฉพาะจุด • พิสูจน์ผล**

แนวทางนี้ตั้งใจให้ใช้ซ้ำกับงาน “สร้างอัลบั้มเพลง → สำรอง Handy/สื่อเดิม → Format เมื่อจำเป็น → Copy เพลง → ตรวจครบ → ถอดอย่างปลอดภัย” ทุกครั้ง
