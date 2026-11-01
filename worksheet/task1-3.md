# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Theerayut Attajak

**Student ID / รหัสนิสิต:** 6970142021

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 483 | 590 | 93.5% | 100% |
| t0-small | 12 | 27 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

- ได้ความแม่นยำ: 93.5% (`t0-large`) เทียบกับ 72.5% (`t0-small`) เพิ่มขึ้น 21 จุด มีแค่ large model ที่ผ่าน QR3 (≥ 90%)
- แลกด้วย latency: p50 483 ms เทียบกับ 12 ms (~40 เท่า), p95 590 เทียบกับ 27 ms แลกด้วยเงิน: ทุก review (100%) เป็นการเรียก large ที่ต้องจ่ายเงิน
- แลกด้วย capacity ด้วย: 4 concurrent × 200 ms จำกัดไว้ที่ 20 calls/s (QR5) ดังนั้น p50 ที่ ~480 ms ภายใต้ 10 concurrent clients
  เกิดจากคิวที่รออยู่หน้า model ไม่ใช่เวลา 200 ms ของตัว model เอง


## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 25.9 | 254 | 437 | 18 | 0 | 92.9% |
| t1-after | 9.5 | 396 | 1005 | 0 | 26 | 93.7% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

- p50/p95 คำนวณจากเฉพาะ request ที่ *สำเร็จ* เท่านั้น request ที่ค้าง 18 รายการกลายเป็น `errors`
  (client ยอมแพ้ที่ 10 s) และหลุดออกจากตัวเลข latency ทำให้ p50 (254 ms) ดูดีกว่า baseline (483)
- เหตุผลหลัก: call ที่ค้างจะ sleep *ก่อน* เข้าคิวของ large model (ไม่ได้ยึด slot ใดใน 4 slot) จึงหลุดออกจากคิวไปเลย
  request ที่เหลือจึงรอคิวน้อยลงและเร็วขึ้น ทั้งๆ ที่ระบบโดยรวมแย่ลง
- เรื่องจริงคือ: `errors` = 18 (review หายไป 9%) และการรันใช้เวลา 25.9 s แทนที่จะเป็น ~10 s



**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

- request ที่ค้างยึด connection/coroutine ที่เปิดค้างไว้ใน review service และ gateway ตลอด 20 s และยึด
  1 ใน 10 concurrent client workers ของ load test ไว้จน client ยอมแพ้ที่ 10 s worker ที่ถูกยึดส่ง review ใหม่ไม่ได้
  throughput ของทั้งระบบจึงลดลง ไม่ใช่แค่ 10% ของ review ที่ช้า (ในระบบจริง GPU ที่ค้างอาจยึด model slot ด้วย
  แต่ในโค้ดนี้การค้างเกิดก่อนเข้า semaphore)
- ไม่มี error เกิดขึ้น จึงไม่มีอะไรปล่อย resource ก่อนเวลา และไม่มีใครได้รับการแจ้งเตือน (silent failure)
- เปรียบเทียบกับ `t1-after`: timeout 1 s ปล่อย resource คืน เวลารันลดจาก 25.9 s → 9.5 s, errors 18 → 0



**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

- เก็บไว้ด้วย status `pending` ให้ background/batch job มาให้คะแนนใหม่ภายหลัง (retry แบบ
  backoff อาจใช้ large model ช่วง off-peak) จากนั้นอัปเดต label และยอดนับของเจ้าของร้าน
- Topology: **T4 asynchronous / batch**: queue (message broker, W10) หรือตารางที่เก็บไว้แล้วสแกนเป็นระยะ
- `t1-after` ได้ pending 26 รายการ (13%) นั่นคือ workload ของ batch job



**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

- 0.1 s: สั้นกว่า large call ปกติ (~200 ms + คิว ~480 ms ที่ p50) แทบทุก review จะ
  timeout → เกือบทั้งหมดเป็น pending, FR2 แทบใช้งานไม่ได้, backlog ของ batch พุ่งสูง inference service
  ก็ยังทำงานนั้นไปเปล่าๆ (เสียค่าใช้จ่ายฟรี)
- 10 s: ละเมิด QR2 (รอ ≤ 1 s) และ QR1 call ที่ค้างยังยึด resource ไว้ 10 s ดังนั้น
  การรอต่อกันเป็นทอดๆ แบบใน Q1.2 ส่วนใหญ่จะกลับมา client (timeout 10 s) อาจยอมแพ้ก่อน → เกิด errors อีก
- 1 s อยู่สูงกว่า p95 ปกติ (590 ms) แต่ต่ำกว่าการค้าง (20 s) มาก สังเกตว่า `t1-after` p95 = 1005 ms:
  ตัว timeout เองเป็นตัวกำหนด tail

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 4 | 536 | 91.5% | 38% |
| t2-0.7 | 0.7 | 367 | 573 | 93.0% | 53% |
| t2-0.8 | 0.8 | 453 | 578 | 93.5% | 64% |
| t2-0.9 | 0.9 | 480 | 594 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

- QR4 บอกให้ลดจำนวน large call ให้น้อยที่สุดโดยยังรักษา QR3 (≥ 90%)
  - 0.6: 91.5%, large 38% เป็นตัวเลือกที่ถูกที่สุดที่ผ่าน แต่ margin มีแค่ 1.5 จุด
  - 0.7: 93.0%, large 53% ความแม่นยำใกล้เคียงกับ large อย่างเดียว (93.5%) ด้วยต้นทุนประมาณครึ่งหนึ่ง
- เลือก **0.7**: 0.6 ทำตาม QR4 ตามตัวอักษร แต่ 0.7 ซื้อ safety margin กันข้อมูล drift แลกกับต้นทุน +15% และเป็นค่าเดียวกับที่ใช้ใน Task 3 และ Task 4
- คุณภาพที่ต้องยอมเสีย: ต้นทุน (large call 53% เทียบกับ 38% ที่ 0.6) และความแม่นยำเล็กน้อย (93.0% เทียบกับ 93.5%)
  เมื่อเทียบกับ large อย่างเดียว สังเกตว่า 0.8 และ 0.9 ได้ 93.5% เท่ากับ large อย่างเดียว แต่ต้องจ่ายสำหรับ large call 64–77%

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

- p50 คือ review ทั่วไป เมื่อ review มากกว่าครึ่งตอบโดย small model ค่ามัธยฐานก็จะเป็น
  latency ของ small model (4 ms ที่ 0.6) ที่ 0.7 ยังเป็น 367 ms เพราะ 53% ถูก escalate มัธยฐานจึงตกไปที่ large call
- p95 เป็น review ที่ถูก escalate เสมอ: ต้องจ่ายทั้ง small call *บวก* large call เต็มๆ และคิวของมัน
  (536–594 ms ≈ `t0-large` p95 590) cascade ไม่ได้ทำให้ review ยากๆ เร็วขึ้น มันแค่ตัด review ง่ายๆ ออกไป
- อัตราการ escalate เป็นตัวตัดสิน: p50 จะลดลงเหลือระดับ small model (4 ms) ก็ต่อเมื่อ large_calls < 50% ส่วนที่ 0.7
  (53%) p50 ลดลงจาก 483 เหลือ 367 ms เพียงเพราะคิวของ large model สั้นลง

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

- small model *มั่นใจแต่ผิด* กับประโยคปฏิเสธ: "Honestly the salad is not cold." (จริงๆ คือ positive) ถูกให้
  negative ที่ confidence 0.702 เพราะเห็นคำว่า "cold" ที่ threshold 0.7 คำตอบนี้ผ่านเกณฑ์และไม่ถูกส่งต่อไปที่ large model
- cascade ไม่เคย escalate กรณีเหล่านี้ เจ้าของร้านจึงเห็น label ผิด (เช่น ลูกค้าที่พอใจถูกนับเป็น negative)
  ซึ่งทำให้ยอดนับของ FR3 เพี้ยนด้วย confidence ไม่ได้ calibrated โดยเฉพาะกับ input ที่ไม่เหมือนข้อมูล training (drift: คำ
  slang ใหม่, ภาษาไทยปนอังกฤษ)

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 4 | 527 | 93.0% | 38% | 35% |
| t3-cache-0.6 | 3 | 557 | 91.5% | 31% | 36% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

- review ที่ซ้ำกันพอดีหลัง normalise: review สั้นๆ ที่พบบ่อย ("Delicious!", "Too salty") workload
  มีการซ้ำแบบนี้ 25% → cache hit 35% ใน `t3-cache` และ large call ลดจาก 53% เหลือ 38% ที่ความแม่นยำเท่าเดิม
- ไม่เคยได้ประโยชน์: review ข้อความอิสระยาวๆ ที่ไม่ซ้ำ คำเกินมาหนึ่งคำหรือพิมพ์ผิดก็ได้ key ต่างกันแล้ว cache แบบ exact-match
  ช่วยกรณีนี้ไม่ได้ ต้องใช้ semantic/embedding similarity ซึ่งก็มีความเสี่ยง hit ผิดในตัวมันเอง

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

- คำตอบใน cache มาจาก model เก่า จึงต้อง invalidate ไม่อย่างนั้นข้อความที่ซ้ำจะไม่เคยเห็น
  การปรับปรุงของ model ใหม่เลย
- การออกแบบ: ใส่ model version ไว้ใน cache key (`v2:delicious!`) หรือ flush ตอน deploy เพิ่ม TTL
  และเก็บ `model_version` ไว้กับทุก prediction (และใน review DB) เพื่อให้การ re-score pending/batch และ
  การ audit รู้ว่า model ไหนเป็นคนติด label อะไร

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

percentile ที่คิดจาก request ที่สำเร็จเท่านั้นซ่อน failure ไว้: ใน `t1-before` p50 (254 ms) ดูดีกว่า baseline (483 ms) ทั้งที่มี review หายไป 18 รายการ จึงต้องดู `errors` และเวลารันควบคู่กับ latency เสมอ
