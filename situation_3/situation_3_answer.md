โจทย์ข้อนี้คล้ายกับข้อแรก ซึ่งสามารถใช้สถาปัตยกรรมแบบเดิมได้

`[prompts] Gemini (1)`  

จากการใช้ Gemini ในการตรวจสอบความสามารถของ Redis พบว่า ยังสามารถรับการร้องขอในจำนวน 100000 Req/Second ได้ ดังนั้น เรายังใช้สถาปัตยกรรมแบบเดิมก็ คือ

Request & Process Discount Process
```
(1) Client --> Server <--> Redis # client ส่ง request ไปที่ server และ server ส่งให้ Redis คำนวณและเพิ่มคนเป็น Key และ Stream และส่งผลลัพธ์กลับมาที่ Server
(3) Server --> Client # Server ส่งผลลัพธ์ให้ client
```

Write Discount Registration Result Worker
```
(1) Redis -->(subscription) Worker <--> Database (PostgreSQL) # server subscribe Redis stream และเขียนข้อมูล user และส่วนลด เข้าไปใน PostgreSQL
```

แต่เนื่องจากมีความสงสัยเรื่อง limit เพิ่มเติม จึงค้นหาข้อมูลเพิ่ม

`[prompts] Gemini (2)`  

ได้ความว่า หลักแสนอาจจะเกิดปัญหาได้ และ AI ได้แนบ Solutions มาให้ทันที คือ

1. ทำ sharding key ใน Redis เพื่อกระจายโหลด
2. เพิ่ม Flag ให้ Server ทันทีที่จำนวนคูปองเต็ม ให้เปลี่ยน Flag และ Response ทันที 
3. เพิ่ม Rate-Limit ให้ Request และใช้ Connection Pool กับ Redis
4. เรียก XADD ผ่าน Server แทน

เนื่องจากข้อ 4 อาจทำให้ Record หายเมื่อเกิด Error ก่อนเรียก XADD ได้ จึงถามถึงปัญหาเรื่องนี้

`[prompts] Gemini (3)`  
`[prompts] Gemini (4)`  

ดังนั้นในที่นี้จะทำการเพิ่มข้อ 2-3 ก็เพียงพอต่อความต้องการของระบบ

จะได้เป็น 

1. ทำการจำกัด Request ด้วย Rate-Limit ที่ 10,000 Req/sec เช่น ใช้ NGINX
2. ทำการเช็ค Flag บน Server RAM ว่าปัจจุบันได้คูปองครบแล้วหรือไม่ ถ้าหากครบแล้วให้ตอบกลับ Request ทันที
3. Server ทำการเรียก Redis โดยใช้ Lua Script เพื่อทำการ ลดจำนวนที่เหลือของคูปอง
4. ตอบกลับผลลัพธ์ไปที่ Client
5. Worker ทำการ Read Redis Stream และเขียนไปที่ PostgreSQL
6. การเขียนข้อมูลที่ PostgreSQL ควรใช้เป็น Upsert เพราะหากเกิด Server Crash อาจเกิดการ Read Stream ซ้ำได้ เพื่อไม่ให้เพิ่มข้อมูลซ้ำ
7. มีระบบ Reconcile เช่น ตรวจสอบ Stream เทียบกับ ข้อมูลบน PostgreSQL เช่นตอนสิ้นวัน สิ้นแคมเปญ หรือเป็นช่วงๆ