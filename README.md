# GISSkill

ชุดทักษะสำหรับ Codex ด้านงานภูมิสารสนเทศ

## Skill ที่เพิ่มแล้ว

### `georeference-grid-map-to-kml`

ใช้ตรึงพิกัดภาพแผนที่หรือ PDF ที่ไม่มี Spatial Reference โดยอาศัยค่าพิกัดระวาง/เส้นกริดที่พิมพ์รอบกรอบแผนที่ จากนั้นสกัดเส้นขอบเขตและส่งออกเป็น KML

ตำแหน่งไฟล์:

```text
skills/georeference-grid-map-to-kml/SKILL.md
```

ความสามารถหลัก:

- รองรับ PNG, JPEG, TIFF และ PDF
- ตรวจ CRS, Datum, UTM Zone และ Hemisphere ก่อนประมวลผล
- ใช้ค่าพิกัด Tick/Grid เพื่อสร้าง Pixel-to-Map Transform
- สร้าง GeoTIFF ใหม่โดยไม่แก้ไขต้นฉบับ
- สกัดขอบเขตสีหรือเส้นที่ผู้ใช้ระบุ
- ตรวจ Geometry, Residual, Ground Pixel Size และ Axis Order
- ส่งออก KML เป็น WGS 84 (`EPSG:4326`) ตามลำดับ `Longitude, Latitude`
- สร้าง QC Report, Preview และ Processing Log
- หยุดเมื่อ CRS หรือขอบเขตกำกวม แทนการเดาข้อมูล

## ตัวอย่างบางกระเจ้า

ไฟล์ตัวอย่าง:

```text
examples/bang-kachao/bang_kachao_boundary.kml
examples/bang-kachao/qc_report.json
```

ตัวอย่างนี้ตรึงพิกัดจากค่าระวางที่พิมพ์บนภาพ:

- ระบบพิกัดต้นทาง: WGS 84 / UTM Zone 47N (`EPSG:32647`)
- ความละเอียดที่คำนวณได้ประมาณ 5.80 เมตรต่อพิกเซล
- ขอบเขตสกัดจากเส้นสีแดงกึ่งกลางความหนาของเส้น
- พื้นที่ประมาณ 15.185 ตารางกิโลเมตร
- สถานะ QC: `WARNING` เนื่องจากเป็นการตีความจากแผนที่มาตราส่วน 1:50,000 ไม่ใช่หมุดควบคุมภาคสนาม

> KML ตัวอย่างเหมาะสำหรับการอ้างอิงและตรวจสอบเบื้องต้น ไม่ควรใช้แทนขอบเขตรังวัด ขอบเขตที่ดิน หรือขอบเขตตามกฎหมาย
