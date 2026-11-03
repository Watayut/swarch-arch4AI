# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Watayut Aiamanan

**Student ID / รหัสนิสิต:** 6970267821

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 505 | 585 | 93.5% | 100% |
| t0-small | 58 | 117 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> Large model ซื้อความแม่นยำ: accuracy 93.5% (t0-large) เทียบกับ 72.5% (t0-small) คือเพิ่มขึ้น 21 จุด เพราะมันเห็นคู่คำและกลุ่มคำ จึงอ่าน "not bad" หรือประโยคที่มี "but" ได้ ราคาที่ต้องจ่ายคือ latency ที่สูงขึ้นราว 9 เท่าที่ p50 (505 vs 58 ms) และ 5 เท่าที่ p95 (585 vs 117 ms) บวก cost ที่ทุกรีวิวกลายเป็น large call (100% vs 0%) และเพดาน capacity 20 calls/s ที่ small model ไม่มี

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.2 | 227 | 352 | 18 | 0 | 93.4% |
| t1-after | 9.3 | 373 | 1010 | 0 | 27 | 93.1% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 ของ t1-before (227 ms) ดูดีกว่า baseline (505 ms) เพราะ load test คำนวณ latency เฉพาะ request ที่สำเร็จ: 18 request ที่ค้างจนถูก client ตัดที่ 10 วินาทีหลุดจากการคำนวณไปอยู่ใน errors และระหว่างที่พวกมันค้าง slot อยู่ request ที่เหลือมีคนแย่งคิว large model น้อยลงจึงเร็วขึ้น ค่าที่บอกความจริงคือ errors (18 รายการ = รีวิวหายไป 9%) และเวลาที่ run ใช้ (26.2 s เทียบกับราว 10 s ของ baseline)

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างไม่ได้แค่ช้า แต่ถือทรัพยากรไว้ตลอดเวลาที่รอ: 1 ใน 10 slot ของ client, connection ใน pool และ coroutine ของ gateway (timeout 60 s) และของ review service (ตอนนั้น timeout=None) ไปจนกว่า large model จะตอบใน 20 วินาที ความเสียหายจึงเท่ากับ สัดส่วน × เวลาที่ถือ: 18 calls × 10 s = 180 slot-วินาที บน pool ที่มี 10 slot เท่ากับเวลารวมเพิ่มราว 16–18 วินาที ซึ่งใกล้เคียง 26.2 s ที่วัดได้ ถ้าโหลดเข้ามาต่อเนื่อง request ที่ค้างจะสะสมจนกิน pool หมดและรีวิวที่ไม่เกี่ยวกับ call ที่ค้างก็เข้าไม่ได้ (resource exhaustion ลามเป็น cascading failure) ต่างจากรีวิวที่แค่ช้าแล้วคืนทรัพยากรทันที

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> เก็บรีวิวไว้ก่อน (review service มี `GET /reviews/pending` ให้อยู่แล้ว) แล้วให้ batch job หรือ worker มาดึงรีวิวที่ pending ไปส่ง inference service ติด label ย้อนหลังเป็นชุดเมื่อ large model กลับมาปกติ และ retry จนกว่าจะได้ label topology จาก lecture ที่ทำแบบนี้คือ T4 Asynchronous / batch (เข้าคิวผ่าน broker จาก W10 หรือเก็บใน store แล้วค่อย score ทีหลัง)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> ถ้า 0.1 s จะสั้นกว่าเวลาปกติของ large model (140–260 ms ต่อ call และ p95 585 ms เมื่อรวมคิว 4 slot) แทบทุกรีวิวจะกลายเป็น pending ทั้งที่ระบบไม่ได้เสีย และ inference service ยังทำงานต่อจนจบหลัง timeout จึงจ่าย cost ของ large call โดยไม่ได้ label ถ้า 10 s จะเท่ากับ timeout ของ client ผู้ใช้ต้องรอ 10 วินาทีแล้ว client ยอมแพ้พร้อมกัน ผลจึงเหมือน t1-before (errors และ run 26 s) และผิด QR2 ที่ให้ review service รอได้ไม่เกิน 1 วินาที timeout ที่ดีต้องยาวกว่า p95 ปกติ (ราว 0.6 s) แต่สั้นกว่าที่ผู้ใช้ทนได้ (1 s)

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 19 | 537 | 91.5% | 38% |
| t2-0.7 | 0.7 | 375 | 581 | 93.0% | 53% |
| t2-0.8 | 0.8 | 447 | 577 | 93.5% | 64% |
| t2-0.9 | 0.9 | 462 | 579 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก 0.8 เพราะได้ accuracy 93.5% (t2-0.8) เท่ากับ large model ทุกประการ โดยส่ง large แค่ 64% ของรีวิว (ประหยัด 36%) ส่วน 0.9 ให้ accuracy เท่ากันแต่แพงกว่า (77%) และ 0.6 แม้ยังผ่าน QR3 (91.5%) แต่เหลือ margin เพียง 1.5 จุดบนตัวอย่าง 200 รีวิว ซึ่งเสี่ยงหลุดต่ำกว่า 90% ใน run จริง quality ที่ยอมเสียคือ cost (QR4) และ performance ที่ p50: 0.8 ได้ p50 447 ms เทียบกับ 19 ms ของ 0.6

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> p95 คือ request ที่ช้าอันดับประมาณ 10 จาก 200 ซึ่งเป็น request ที่ถูกส่งต่อไป large model และต้องรอคิว 4 slot เสมอ cascade เปลี่ยนแค่ "สัดส่วน" ของรีวิวที่เดินทางช้า (64% ที่ 0.8) ไม่ได้ทำให้ทางช้าเร็วขึ้น ตราบใดที่เกิน 5% ยังไป large p95 จึงค้างอยู่ราว 577–581 ms ทุก threshold ส่วน p50 ตอบจากรีวิวตรงกลาง: ที่ 0.8 รีวิวตรงกลางยังเป็น large call จึงลดจาก 505 เป็น 447 ms เพราะคิวสั้นลงเท่านั้น แต่พอสัดส่วน small เกินครึ่งที่ 0.6 p50 ร่วงเหลือ 19 ms ทันที

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> Small model เป็น bag-of-words ของ 100 คำที่พบบ่อย ไม่เห็นลำดับคำ จึงมั่นใจผิดกับประโยคปฏิเสธหรือขัดแย้ง เช่น "Not bad at all, will come back" เห็นคำว่า bad แล้วตอบ negative ด้วย confidence สูง cascade จึงไม่ส่งต่อไป large เลย ผลคือเจ้าของร้านเห็นรีวิวชมเป็นรีวิวด่า (หรือกลับกันจนพลาดลูกค้าที่ไม่พอใจ) และถ้าเปิด cache คำตอบที่ผิดนั้นจะถูกเสิร์ฟซ้ำทุกครั้งที่ข้อความนี้โผล่มา

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache-0.8 | 28 | 546 | 93.5% | 40% | 34% |
| t3-cache (threshold 0.7, for comparison) | 17 | 550 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวที่ข้อความซ้ำกันทุกตัวอักษร (ต่างได้แค่ตัวพิมพ์เล็ก/ใหญ่และช่องว่าง) เช่นรีวิวสั้นยอดนิยม "Delicious!", "Too salty", "Cold food" ซึ่งใน workload มี 25% บวกรีวิวจาก test set ที่ถูกสุ่มซ้ำ ทำให้ได้ cache hits 34% และ large calls ลดจาก 64% เหลือ 40% โดย accuracy เท่าเดิม 93.5% (t3-cache-0.8) รีวิวที่ไม่มีวันได้ประโยชน์คือรีวิวยาวที่เขียนเฉพาะตัว การเจอข้อความครั้งแรก และข้อความที่ต่างกันแม้นิดเดียว (เครื่องหมายวรรคตอน สะกดต่าง หรือความหมายเดียวกันคนละคำ) เพราะ key ไม่เท่ากัน

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้องทำให้คำตอบเก่าใน cache หมดอายุทั้งหมด ไม่เช่นนั้นข้อความที่พบบ่อยจะได้ label จากโมเดลเก่าตลอดไป เพราะ LRU ไม่มีวัน evict entry ที่ถูกใช้บ่อย ออกแบบโดยใส่ model version ลงใน cache key (เช่น `large-v2|too salty`) หรือเก็บ version ไว้ใน entry แล้วถือว่าไม่ตรงคือ miss การ deploy โมเดลใหม่จะทำให้ของเก่า miss เองโดยไม่ต้อง flush มือ และใส่ TTL ไว้เป็นชั้นป้องกันสุดท้าย ตอนนี้ cache อยู่ใน memory ของ process จึงหายเมื่อ restart แต่ถ้าย้ายไป shared cache อย่าง Redis จะไม่มีอะไรล้างให้

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> สิ่งที่ไม่คาดคิดคือ metric ก็โกหกได้: ใน t1-before ระบบกำลังทำรีวิวหาย 9% (18 errors) แต่ p50 กลับดีขึ้นเกือบเท่าตัว (227 vs 505 ms) และ p95 ก็ดีขึ้นด้วย เพราะ request ที่ล้มเหลวถูกตัดออกจากการคำนวณ latency และตัวที่เหลือแย่งคิวกันน้อยลง บทเรียนคือ percentile ของ latency ใช้ได้เฉพาะเมื่ออ่านคู่กับ errors, pending และเวลาที่ run ใช้ ไม่อย่างนั้นระบบที่กำลังพังจะดูเหมือนระบบที่เร็วขึ้น
