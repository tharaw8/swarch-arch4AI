# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** ธรา วรานุเศรษฐ์

**Student ID / รหัสนิสิต:** 6630132921

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 505 | 613 | 93.5% | 100% |
| t0-small | 30 | 67 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> large model ให้ accuracy 93.5% เทียบ small 72.5% แต่ต้องแลกกับ latency ที่ช้ากว่ามาก และ cost ทุก call รวมถึงติด capacity 20 calls/s

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.1 | 267 | 456 | 18 | 0 | 92.9% |
| t1-after | 10.2 | 447 | 1016 | 0 | 20 | 92.8% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50/p95 คิดจาก request ที่สำเร็จเท่านั้น, 18 request ที่ค้างจน client timeout ถูกตัดออกจาก latency ทำให้ p50 ดูดีกว่า baseline, column ที่บอกความจริงคือ errors และ run time ที่เพิ่มจาก 10.3 s เป็น 26.1 s

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างไม่ได้แค่ช้า แต่ถือ connection และ slot ของ gateway, review service และ client ไว้ตลอด 20 s ทำให้ request อื่นต้องรอคิว ทำให้ทั้งระบบช้าลง ไม่ใช่แค่ 10% ของรีวิว

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ควรเก็บรีวิว pending ไว้ (ตาม FR4) แล้วให้ batch job มาดึงจาก `GET /reviews/pending` ไปติด label กับ large model ในช่วงที่ระบบว่าง และ retry ถ้ายังล้มเหลว นี่คือ topology แบบ asynchronous / batch (T4)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> 0.1 s สั้นกว่าเวลาปกติของ large model ทำให้แทบทุกรีวิวกลายเป็น pending แม้ระบบปกติ ส่วน 10 s นานเกินไป ผิด QR2 และยังถือทรัพยากรไว้นานจนเกิดปัญหาแบบ t1-before และเท่ากับ client timeout จึงยังเกิด errors ได้

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 17 | 551 | 91.5% | 38% |
| t2-0.7 | 0.7 | 368 | 627 | 93.0% | 53% |
| t2-0.8 | 0.8 | 485 | 644 | 93.5% | 64% |
| t2-0.9 | 0.9 | 482 | 582 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก 0.7 เพราะ accuracy 93.0% ผ่าน QR3 โดยมี margin 3 จุด และลด large_calls เหลือ 53% (QR4) ส่วน 0.6 ถูกกว่าแต่ margin แค่ 1.5 จุด ซึ่งเสี่ยงเมื่อข้อมูลเปลี่ยน สิ่งที่ยอมเสียคือ accuracy 0.5 จุดเมื่อเทียบกับ large ล้วน

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> cascade ทำให้รีวิวที่ small มั่นใจตอบเร็วมาก ซึ่งเป็นครึ่งล่างของ distribution จึงลด p50 แต่ยังมี 53% ของรีวิวที่ไป large และต้องรอคิว capacity 4 เหมือนเดิม tail 5% จึงเป็นรีวิวกลุ่มนี้ทั้งหมด p95 จึงแทบไม่เปลี่ยน

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> รีวิวที่มีคำปฏิเสธ เช่น "not bad at all" small model เห็นคำว่า "bad" แล้วตอบ negative ด้วย confidence สูงเกิน threshold จึงไม่ถูกส่งต่อไป large ทำให้เจ้าของร้านเห็นรีวิวชมเป็นรีวิวติ และอาจได้ alert ผิด

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 11 | 548 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> ได้ประโยชน์: รีวิวสั้นที่ซ้ำกันบ่อย เช่น "Delicious!", "Too salty" ทำให้ cache hits 35% และ large_calls ลดจาก 53% เหลือ 38% 
> 
> ไม่มีวันได้ประโยชน์: รีวิวยาวที่เขียนเอง ข้อความไม่ซ้ำกับใคร เพราะ key ต้องตรงทั้งข้อความ

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้องล้าง cache เพราะผลเก่ามาจากโมเดลเดิม ถ้าไม่ล้างจะได้คำตอบของโมเดลเก่าต่อไป ออกแบบโดยใส่ model version ใน key ให้ deploy ใหม่แล้วไม่ hit ผลเก่าโดยอัตโนมัติ และใส่ TTL ให้ entry หมดอายุเอง

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> latency percentile ที่ดูดีอาจหลอกได้ เพราะ request ที่ล้มเหลวไม่ถูกนับ ต้องดู errors และ throughput คู่กันเสมอ นอกจากนี้ cascade ลด cost ได้มากแต่แทบไม่ช่วย p95
