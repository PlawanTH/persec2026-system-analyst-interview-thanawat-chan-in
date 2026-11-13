# Situation 1: Question 1

## Final Solution

### Version 3

#### Architecture

**(System 1)** Request & Decrease Discount Process
```
(1) Client --> Server <--> Redis # client ส่ง request ไปที่ server และ server ส่งให้ Redis คำนวณและเพิ่มคนด้วย Key `discount:{discount_id}:user:{user_id}` และเขียนลง Redis Stream ด้วย XADD key `discount:{discount_id}:stream` และส่งผลลัพธ์กลับมาที่ Server
(2) Server --> Client # Server ส่งผลลัพธ์ให้ client
```

**(System 2)** Write Discount Registration Worker
```
(1) Redis -->(subscription) Worker <--> Database (PostgreSQL) # server subscribe Redis stream และเขียนข้อมูล user และ discount_id เข้าไปใน PostgreSQL
```

#### Schema PostgreSQL
1. ข้อมูลของ user ที่ได้รับส่วนลด `(user UUID, discount_id UUID, requested_at TIMESTAMPZ, created_at TIMESTAMPZ)`

#### Redis Key
1. จำนวนคงเหลือของส่วนลด `discount:{discount_id}:remain`
2. คนที่ได้ส่วนลด `discount:{discount_id}:user:{user_id} "YYYY-MM-DDThh:mm:ssZ"`

## Step of Thought

คำนวณจำนวนคนที่เข้าเฉลี่ยต่อวินาที  
50000 / 5 / 60 = 166.667 คนต่อวินาที  
Peak: ~500 คนต่อวินาที

ข้อมูลที่ใช้
1. จำนวนคงเหลือของส่วนลด `(discount_id UUID, remain int)`
2. ข้อมูลของ user ที่ได้รับส่วนลด `(user UUID, discount_id UUID, created_at TIMESTAMPZ)`

### Version 1 ระบบที่ซับซ้อนน้อยที่สุด

```
(1) Client --> Server  # client ส่ง request
(2) Server <--> Database (PostgreSQL) # server update database ด้วยการลด count และเพิ่ม record ของ user
(3) Server --> Client # response ผลกลับไปหา client
```

ปัญหา 
1. อาจจะเกิดการโหลดของ Server ที่สูงหากมี Request เป็นจำนวนมาก ในเคสนี้ หากเจอพีค ก็อาจเกิดปัญหา และมีคิวรอเพิ่มขึ้นเป็นจำนวนมากได้ ส่งผลทำให้ระบบล่มเนื่องจาก Resource ไม่พอ

การแก้ไข
1. ทำให้ Request ประมวลผลเรื่องจำนวนของส่วนลดให้น้อยที่สุด 
2. Offload การอ่านเขียน PostgreSQL ไปให้อีก Process เพื่อจำกัดจำนวนการอ่านเขียน Database ไม่ให้โหลดเยอะเกินไป

วิธีแก้ไข
1. สามารถใช้ Cache Storage แทน PostgreSQL ในการ Update ค่าของจำนวนส่วนลด และ User ที่กดรับได้
2. สามารถใช้ Event Streaming เพื่อรับ Request และมี Server เพื่อมา Subscribe Topics เพื่อ Offload และจำกัดการอ่านเขียน PostgreSQL

ข้อเสีย
1. จะต้องเพิ่มระบบของการ Subscription จาก Event Streaming เพื่อเขียนลง PostgreSQL ทำให้ระบบซับซ้อนขึ้น

### Version 2 ใช้ Redis แทน PostgreSQL ในการคำนวณจำนวนส่วนลด ใช้ PostgreSQL ในการเขียนข้อมูลอย่างเดียว และใช้ Kafka ในการ Offload การเขียน PostgreSQL

**(System 1)** Request & Decrease Discount Process
```
(1) Client --> Server <--> Redis # client ส่ง request ไปที่ server และ server ส่งให้ Redis คำนวณจำนวนส่วนลดและเพิ่ม user ด้วย key และส่งผลลัพธ์กลับมาให้ Server
(2) Server --> Kafka # หากได้ส่วนลด Server ส่ง event ไปรอคิวดำเนินการ
(3) Server --> Client # Server ส่งผลลัพธ์ให้ client
```

**(System 2)** Write Discount Registration Worker
```
(1) Kafka -->(subscription) Worker <--> Database (PostgreSQL) # Worker subscribe topic และนำมาเขียนข้อมูล user กับ discount_id เข้าไปใน PostgreSQL
```

ปัญหา
1. อาจเกิด Inconsistency หาก Server Crash กลางทาง เนื่องจากข้อมูลบน Redis อาจไม่ได้ส่งไป Kafka ทำให้มูลไม่ตรงกัน

`[prompts] Gemini (1)`  
`[prompts] Gemini (2)`

หลังจากหาคำตอบจาก Gemini เรื่องการแก้ปัญหาเชิงสถาปัตยกรรม ได้คำตอบว่า ตัว Redis สามารถทำเป็น Streaming ได้เช่นเดียวกันกับ Kafka และความสงสัยว่า ถ้าแบบนั้น อาจจะไม่จำเป็นต้องมี Kafka แล้ว เพราะจำนวน Request Per Second ไม่ได้เยอะมาก และเราไม่ได้ต้องการเก็บข้อมูลคนที่ไม่ได้ส่วนลด

ดังนั้น เพื่อลดความซับซ้อนของระบบ จะได้สรุปโครงสร้างเป็น

### Version 3 ลดความซับซ้อนของระบบ โดยนำ Kafka ออก และใช้ Redis Stream แทน

**(System 1)** Request & Decrease Discount Process
```
(1) Client --> Server <--> Redis # client ส่ง request ไปที่ server และ server ส่งให้ Redis คำนวณและเพิ่มค้วย Key และเพิ้มเข้า Stream และส่งผลลัพธ์กลับมาที่ Server
(2) Server --> Client # Server ส่งผลลัพธ์ให้ client
```

**(System 2)** Write Discount Registration Worker
```
(1) Redis -->(subscription) Worker <--> Database (PostgreSQL) # server subscribe Redis stream และเขียนข้อมูล user และ discount_id เข้าไปใน PostgreSQL
```

ปัญหาที่สังเกตได้
1. เนื่องจากเราใช้วิธีขียนลง PostgreSQL แบบ Asynchronous อาจทำให้ ข้อมูลใน PostgreSQL ไม่ตรงกับ Redis (กรณีเกิดการผิดพลาด) อาจจะต้องเพิ่ม process สำหรับการ reconcile ข้อมูล เช่น ช่วงหมดวัน เพื่อทำให้ข้อมูลเท่ากัน
