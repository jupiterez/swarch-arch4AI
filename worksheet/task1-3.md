# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:**

**Student ID / รหัสนิสิต:**

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 508 | 613 |  93.5% | 100% |
| t0-small | 29 | 76 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> ให้ accuracy เพิ่ม จาก 72.5% เป็น 93.5%  หรือทำนายถูกเพิ่ม 42 รีวิวจาก 200 รีวิว ทำให้ผ่านข้อกำหนดความแม่นยำอย่างน้อย 90% แต่ต้องแลกกับเวลาตอบสนองที่สูงขึ้น โดย p95 เพิ่มจาก 76 เป็น 613 มิลลิวินาที แม้ยังอยู่ภายในเป้าหมาย 1 วินาที และมีต้นทุนการประมวลผลสูงขึ้น เพราะต้องเรียกใช้ Large model ทุกรีวิว ขณะที่ Small model ตอบได้เร็วและมีต้นทุนต่ำกว่า แต่ความแม่นยำยังไม่ผ่านข้อกำหนด

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 34.7 | 251 | 420 | 26 | 0 | 93.7% |
| t1-after | 9.9 | 406 | 1017 | 0 | 26 | 93.1% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> ค่า p50 ใน t1-before ทำให้เข้าใจผิด เพราะคำนวณจาก requests ที่สำเร็จเท่านั้น โดยไม่นับ requests ที่ค้างจน timeout จึงดูเหมือนระบบตอบเร็วขึ้นจาก baseline 508 ms เหลือ 251 ms ทั้งที่มีผู้ใช้บางส่วนไม่ได้รับคำตอบสำเร็จ คอลัมน์ที่บอกปัญหาจริงคือ errors ซึ่งมี 26 จาก 200 requests หรือ 13% ดังนั้นจึงไม่ควรพิจารณา p50 เพียงอย่างเดียว แต่ต้องดูจำนวน requests ที่ล้มเหลวควบคู่กัน

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> แม้การค้างเกิดกับประมาณ 10% ของ calls แต่ requests ที่รออยู่ยังถือ connection และพื้นที่ใน connection pool รวมถึงหน่วยความจำและงานที่ยังไม่เสร็จ ทั้งใน Review Service และ Gateway ทำให้ทรัพยากรสำหรับรับ requests ใหม่ลดลง และอาจเกิดการรอคิวต่อเนื่องจนกระทบรีวิวอื่นด้วย จึงไม่ได้ช้าเฉพาะ 10% นั้น จากผล t1-before มี requests ล้มเหลว 26 จาก 200 รายการ และใช้เวลารวม  34.7 วินาที เมื่อเพิ่ม timeout กับ fallback ระบบจึงยุติการรอได้เร็วขึ้นและรับรีวิวเป็น pending ทำให้ errors ลดเหลือ 0 และเวลารวมเหลือ 9.9 วินาที

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ควรเก็บรีวิวที่มีสถานะ pending ไว้ในฐานข้อมูล แล้วให้ background worker รวบรวมมาทำนายเป็นชุดเมื่อ Inference Service พร้อมใช้งาน จากนั้นอัปเดต label ภายหลัง แนวทางนี้ตรงกับ Batch processing หรือ Batch inference โดยผู้ใช้ไม่ต้องรอผลการทำนายขณะส่งรีวิว

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> หากตั้ง timeout เป็น 0.1 วินาที จะสั้นกว่าเวลาประมวลผลปกติของ Large model ทำให้รีวิวจำนวนมากกลายเป็น pending ทั้งที่โมเดลยังทำงานปกติ แต่หากตั้งเป็น 10 วินาที ระบบจะรอนานและถือทรัพยากรการเชื่อมต่อไว้นานขึ้นเมื่อโมเดลค้าง ทำให้ requests อื่นได้รับผลกระทบ อีกทั้งผิดข้อกำหนดที่ Review Service ต้องรอไม่เกิน 1 วินาที และเสี่ยงให้ client timeout ก่อนรับ fallback

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 11 | 543 | 91.5% | 38% |
| t2-0.7 | 0.7 | 399 | 592 | 93.0% | 53% |
| t2-0.8 | 0.8 | 454 | 579 | 93.5% | 64% |
| t2-0.9 | 0.9 | 482 | 627 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก threshold ที่ 0.7 สำหรับ CampusEats เพราะได้ความแม่นยำ 93.0% ซึ่งใกล้เคียงกับผลหลังปรับ Fallback ใน Task 1 ที่ได้ 93.1% และเรียกใช้โมเดลใหญ่เพียง 53% ของคำขอ จึงช่วยประหยัดการประมวลผลเมื่อเทียบกับ threshold ที่ 0.8 และ 0.9 โดยต้องยอมเสียคุณภาพด้านความแม่นยำเล็กน้อย เพราะทั้งสองค่านั้นให้ความแม่นยำสูงกว่าที่ 93.5% แต่ต้องเรียกใช้โมเดลใหญ่บ่อยขึ้น

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> Cascade ช่วยให้รีวิวที่โมเดลเล็กมั่นใจได้รับคำตอบเร็วขึ้น จึงช่วยลด p50 ส่วนรีวิวที่ยากยังต้องรอโมเดลใหญ่ ทำให้ p95 ยังสูง อย่างไรก็ตาม ผล threshold 0.7 ยังไม่แสดงว่า p50 ดีขึ้นมากกว่า p95 เพราะมีคำขอถึง 53% ที่ต้องเรียกโมเดลใหญ่ ทำให้ p50 ยังอยู่ที่ 399 มิลลิวินาที ขณะที่ p95 อยู่ที่ 592 มิลลิวินาที

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> Confidence สูงไม่ได้รับประกันว่าคำตอบถูกต้อง โดยเฉพาะเมื่อโมเดลเข้าใจความหมายหรือบริบทของข้อความผิด เช่น รีวิวประชดว่า “อร่อยมาก รอแค่สองชั่วโมงเอง” โมเดลเล็กอาจเห็นคำว่า “อร่อยมาก” แล้วมั่นใจว่าเป็นรีวิวเชิงบวก ทั้งที่ผู้ใช้กำลังตำหนิ หากค่าความมั่นใจสูงกว่า threshold ที่ 0.7 ระบบจะยอมรับผลทันทีโดยไม่ส่งให้โมเดลใหญ่ตรวจต่อ ทำให้รีวิวได้รับ label ผิด และผู้ใช้ที่อ่านสรุปรีวิวอาจเข้าใจคุณภาพบริการของร้านคลาดเคลื่อน

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 10 | 550 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวที่ได้ประโยชน์จาก cache คือรีวิวที่มีข้อความซ้ำกับข้อความที่เคยทำนายและยังมีผลเก็บอยู่ เช่น “อาหารอร่อย” เมื่อพบข้อความเดิมอีกก็ใช้ label จาก cache ได้ทันทีโดยไม่ต้องเรียกโมเดลใหม่ ส่วนรีวิวที่มีข้อความเฉพาะ ไม่ตรงกับข้อมูลใน cache และไม่เคยถูกส่งซ้ำ จะไม่ได้ประโยชน์ เพราะต้องเรียกโมเดลทำนายทุกครั้ง จากผลทดลอง มีคำขอ 35% ที่ใช้ผลจาก cache ได้

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> เมื่อ deploy large model ตัวใหม่ ต้องล้างหรือเลิกใช้ผลใน cache ที่สร้างจากโมเดลเดิม เพื่อไม่ให้ระบบยังตอบด้วยผลเก่า ควรออกแบบให้ cache key มีเวอร์ชันของโมเดลร่วมกับข้อความรีวิว เช่น “v2:อาหารอร่อย” เมื่อเปลี่ยนเวอร์ชัน ระบบจะไม่พบผลเก่าใน key ใหม่ จึงทำนายด้วยโมเดลใหม่และเก็บผลไว้ใช้ครั้งถัดไป ส่วน cache เวอร์ชันเก่าสามารถกำหนดให้หมดอายุหรือลบภายหลังได้

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> การเพิ่มการใช้โมเดลใหญ่ไม่ได้ทำให้ความแม่นยำดีขึ้นเสมอไป เช่น threshold ที่ 0.8 และ 0.9 ให้ความแม่นยำเท่ากันที่ 93.5% แต่เรียกใช้โมเดลใหญ่ 64% และ 77% ตามลำดับ แสดงว่าในการทดลองนี้ การเพิ่ม threshold เป็น 0.9 ทำให้ใช้ทรัพยากรมากขึ้นโดยไม่ได้ความแม่นยำเพิ่ม จึงควรใช้ผลการวัดประกอบการตัดสินใจเลือกค่าที่เหมาะสม
