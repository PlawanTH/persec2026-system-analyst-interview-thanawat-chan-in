# Situation 1: Question 2

สืบเนื่องจากโครงสร้างปัจจุบัน เราใช้ Redis เป็น Atomic Update ทำให้ป้องกันเรื่อง Race Condition อย่างแน่นอน

ถ้าย้อนกลับไปที่ Version 1 ก็ยังป้องกันเรื่อง Race Condition เช่นกัน เพราะเรา Update และ Insert เป็น Transaction เดียว ตัว Database จะทำการ Lock Record ไว้

แล้วจะเกิดขึ้นได้เมื่อไหร่บ้าง ถ้าหากเราเขียนโปรแกรมเป็น อ่านมาก่อนแล้วค่อยเรียกเขียนอีกครั้ง ในกรณีนี้ถ้าหาก Server เป็นแบบ Asynchronous ก็จะทำให้สามารถเกิด Race Condition ได้