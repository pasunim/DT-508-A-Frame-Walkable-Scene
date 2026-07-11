# Workshop 04 — A-Frame Walkable Scene with Teachable Machine Pose

การพัฒนาแอปพลิเคชันบนเว็บไซต์ที่ผสาน **A-Frame** (WebGL/WebXR framework
สำหรับสร้างฉาก 3 มิติบนเว็บ) เข้ากับโมเดล Machine Learning จาก
**Teachable Machine** (Pose Project) เพื่อควบคุมกล้องในฉาก 3 มิติด้วย
ท่าทางของร่างกายที่จับภาพจากเว็บแคม แทนการใช้คีย์บอร์ด/เมาส์

อ้างอิงจากเอกสารประกอบวิชา *Machine Learning: การเรียนรู้ด้วยเครื่องและ
การประยุกต์ใช้ปัญญาประดิษฐ์สำหรับการพัฒนาแอปพลิเคชัน* (DT508-WE674-14)

---

## ภาพรวมการทำงาน

1. ผู้ใช้ยืนหน้าเว็บแคม แล้วทำท่าทาง เช่น เอียงตัว/ยกมือซ้าย-ขวา หรือก้าวเดิน
2. โมเดล Teachable Machine (Pose) วิเคราะห์ภาพจากเว็บแคมแบบเรียลไทม์
   และทำนายว่าท่าทางปัจจุบันตรงกับคลาสใด (`left`, `right`, `walk`)
   พร้อมค่าความน่าจะเป็น (probability)
3. ผลการทำนายถูกนำไปแปลงเป็นคำสั่งเคลื่อนที่กล้อง (camera) ในฉาก A-Frame
   ที่แสดงโมเดล 3 มิติสนามแข่ง Monza

```
Webcam → Teachable Machine (Pose model) → probability ต่อคลาส → ขยับกล้องใน A-Frame
```

---

## โครงสร้างไฟล์

| ไฟล์ | หน้าที่ |
|---|---|
| `index.html` | หน้าเว็บหลัก ประกอบด้วยฉาก A-Frame (ซ้าย) และส่วนเว็บแคม/ปุ่ม Start/label ผลทำนาย (ขวา) |
| `app.js` | โหลดโมเดล Teachable Machine, เปิดเว็บแคม, รันลูปทำนายท่าทาง, และขยับกล้องตามผลทำนาย |
| `model.json` | โครงสร้างโมเดล (layers, config) ที่ export จาก Teachable Machine |
| `metadata.json` | ข้อมูลคลาส (class labels) และ metadata ของโมเดล |
| `weights.bin` | ค่าน้ำหนัก (weights) ของโมเดลในรูปแบบไบนารี |
| `monza_circuit_1998_layout.glb` | โมเดล 3 มิติสนามแข่ง Monza (glTF Binary) ที่ใช้แสดงในฉาก A-Frame |
| `README.md` | ไฟล์นี้ |

> UI ของหน้าเว็บใช้ **Bootstrap 5** (โหลดผ่าน CDN ใน `index.html`) สำหรับ
> จัดวาง layout, navbar, card, และปุ่ม ให้ดูเป็นระบบมากกว่าการจัดหน้าด้วย
> inline style ล้วนๆ

---

## ขั้นตอนที่ 1 — สร้างโมเดลด้วย Teachable Machine

Teachable Machine เป็นเครื่องมือบนเว็บที่ช่วยสร้างโมเดล Machine Learning
ได้โดยไม่ต้องเขียนโค้ด สามารถฝึกให้จดจำภาพ เสียง หรือท่าทาง แล้ว export
ออกมาเป็นไฟล์ JavaScript ขนาดเล็กเพื่อใช้งานร่วมกับเว็บไซต์

**ลิงก์:** https://teachablemachine.withgoogle.com/

### 1.1 สร้างโปรเจกต์ใหม่

- เลือก **Pose Project** (ฝึกจากภาพท่าทางร่างกาย จากไฟล์หรือเว็บแคม)

### 1.2 เตรียมข้อมูล (Input Data)

สร้างคลาส (class) อย่างน้อย 3 คลาสสำหรับโปรเจกต์นี้:

| คลาส | ท่าทาง | ใช้ควบคุม |
|---|---|---|
| `left` | ยกมือซ้าย | เลื่อนกล้องไปทางซ้าย (`x - 0.5`) |
| `right` | ยกมือขวา | เลื่อนกล้องไปทางขวา (`x + 0.5`) |
| `walk` | วางมือขนานหน้าอก/ท่าเดิน | เดินหน้า (`z - 0.5`) |
| `stop` *(แนะนำให้เพิ่ม)* | ยืนนิ่ง/ท่าพัก | ไม่ขยับกล้อง (ใช้เป็นคลาสพื้นหลัง) |

> **สำคัญ:** ควรมีคลาสพื้นหลัง (background class เช่น `stop`) เสมอ
> ไม่เช่นนั้นเมื่อไม่ได้ทำท่าใดเลย ระบบอาจจำแนกท่าทางผิดพลาดไปเป็นหนึ่งใน
> คลาสที่กำหนดไว้โดยไม่ตั้งใจ

นำเข้าตัวอย่างภาพ (Add Pose Samples) ได้ 2 ทาง:
- **Webcam** — กด "Hold to Record" ค้างไว้ขณะทำท่า (แนะนำอย่างน้อย
  ~60-150 ภาพต่อคลาส เพื่อความแม่นยำ)
- **Upload** — อัปโหลดไฟล์ภาพ/วิดีโอที่มีอยู่แล้ว

### 1.3 ฝึกโมเดล (Training)

กด **Train Model** ระบบจะฝึกโมเดลในเบราว์เซอร์ (ใช้เวลาไม่กี่วินาทีถึง
นาที ขึ้นกับจำนวนภาพ) สามารถปรับ Parameter เพิ่มเติมได้ที่ **Advanced**
(เช่น Epochs, Batch Size, Learning Rate)

### 1.4 ทดสอบ (Preview)

หลังฝึกเสร็จ ทดสอบผลลัพธ์แบบเรียลไทม์ในช่อง Preview ก่อน export จริง
เพื่อดูว่าโมเดลแยกแยะแต่ละท่าทางได้แม่นยำหรือไม่

### 1.5 Export Model

กด **Export Model** → เลือกแท็บ **Tensorflow.js** → เลือก **Download**
จะได้ไฟล์ 3 ไฟล์:

- `model.json`
- `metadata.json`
- `weights.bin`

นำไฟล์ทั้ง 3 ไฟล์นี้มาวางไว้ในโฟลเดอร์เดียวกับ `index.html` และ `app.js`
(โฟลเดอร์นี้)

---

## ขั้นตอนที่ 2 — โครงสร้างหน้าเว็บ (`index.html`)

```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
<script src="https://aframe.io/releases/1.8.0/aframe.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@teachablemachine/pose@0.8/dist/teachablemachine-pose.min.js"></script>
```

โหลด **Bootstrap 5** (CSS) เพิ่มเติมสำหรับจัดหน้าตาและ layout ให้ดูเป็น
ระบบมากขึ้น ควบคู่กับ 3 ไลบรารีหลักเดิม:
- **A-Frame** — สร้างฉาก 3 มิติ/WebXR ด้วย HTML custom elements (`<a-scene>`,
  `<a-entity>` ฯลฯ)
- **TensorFlow.js** — เอนจินรัน Machine Learning model บนเบราว์เซอร์
- **Teachable Machine Pose library (`tmPose`)** — wrapper สำหรับโหลด/รัน
  โมเดล Pose ที่ export มาจาก Teachable Machine

โครงสร้าง `<body>` ใช้ Bootstrap grid (`container` / `row` / `col-lg-*`)
แทนการ `float` แบบเดิม แบ่งเป็น navbar ด้านบน และการ์ด 2 คอลัมน์:

```html
<nav class="navbar navbar-dark bg-dark mb-4">
  <div class="container">
    <span class="navbar-brand mb-0 h1">A-Frame Walkable Scene</span>
  </div>
</nav>

<div class="container">
  <div class="row g-4">
    <div class="col-lg-8">
      <div class="card shadow-sm scene-card">
        <div class="card-body p-0">
          <a-scene embedded>
            <a-assets>
              <a-asset-item id="event" src="monza_circuit_1998_layout.glb"></a-asset-item>
            </a-assets>
            <a-entity gltf-model="#event" ...></a-entity>
            <a-entity camera look-controls wasd-controls ...>
              <a-cursor color="black"></a-cursor>
            </a-entity>
            <a-light type="ambient" .../>
            <a-light type="directional" .../>
          </a-scene>
        </div>
      </div>
    </div>

    <div class="col-lg-4">
      <div class="card shadow-sm">
        <div class="card-body text-center">
          <h5 class="card-title mb-3">Pose Control</h5>
          <button type="button" class="btn btn-primary btn-lg w-100 mb-3" onclick="init()">Start</button>
          <div id="webcam-wrapper" class="mb-3">
            <canvas id="canvas"></canvas>
          </div>
          <div id="label-container" class="text-start"></div>
        </div>
      </div>
    </div>
  </div>
</div>
```

- **Navbar** — แถบหัวเรื่องด้านบนแบบ Bootstrap (`navbar navbar-dark bg-dark`)
- **การ์ดซ้าย (`col-lg-8`)** — ฉาก A-Frame แสดงโมเดล 3 มิติสนาม Monza
  (`gltf-model`) พร้อมกล้อง (`camera`) ที่รองรับทั้ง `wasd-controls`
  (บังคับด้วยคีย์บอร์ดได้ตามปกติ) และการขยับด้วยโค้ดจากผลทำนายท่าทาง
  ครอบด้วย Bootstrap `card` พร้อมเงา (`shadow-sm`)
- **การ์ดขวา (`col-lg-4`)** — ปุ่ม **Start** (`btn btn-primary btn-lg`)
  สำหรับเริ่มต้นเว็บแคม, `<canvas>` แสดงภาพจากเว็บแคมพร้อม
  keypoints/skeleton ที่ตรวจจับได้ (จัดกลางด้วย flexbox ผ่าน
  `#webcam-wrapper`) และ `label-container` แสดงค่าความน่าจะเป็นของแต่ละ
  คลาสแบบเรียลไทม์ (สไตล์เป็นแถบพื้นหลังสีอ่อน อ่านง่ายกว่า `<div>` เปล่าๆ)
- Layout เป็น **responsive**: บนจอเล็ก (มือถือ/แท็บเล็ต) การ์ดทั้งสองจะ
  เรียงต่อกันในแนวตั้งโดยอัตโนมัติ (Bootstrap grid breakpoint `lg`)

---

## ขั้นตอนที่ 3 — โค้ดหลัก (`app.js`)

### 3.1 `init()` — เริ่มต้นระบบ

```js
async function init() {
    const modelURL = URL + "model.json";
    const metadataURL = URL + "metadata.json";
    model = await tmPose.load(modelURL, metadataURL);
    maxPredictions = model.getTotalClasses();

    const size = 200;
    const flip = true;
    webcam = new tmPose.Webcam(size, size, flip);
    await webcam.setup();   // ขอสิทธิ์เข้าถึงกล้อง
    await webcam.play();
    window.requestAnimationFrame(loop);
    ...
}
```

ทำงานเมื่อกดปุ่ม **Start**:
1. โหลดโมเดลจาก `model.json` + `metadata.json` (ไฟล์ `weights.bin` จะถูก
   อ้างอิงโดยอัตโนมัติผ่าน `model.json`)
2. นับจำนวนคลาสทั้งหมด (`maxPredictions`)
3. สร้าง object เว็บแคมขนาด 200x200 พิกเซล พร้อม flip แนวนอน (เหมือน
   ส่องกระจก) แล้วขอสิทธิ์เข้าถึงกล้อง (`webcam.setup()`) — **นี่คือจุดที่
   เบราว์เซอร์จะเด้ง popup ขออนุญาตใช้กล้อง**
4. เริ่ม loop การทำนายด้วย `requestAnimationFrame`
5. สร้าง `<div>` ว่างในหน้าเว็บสำหรับแสดง label ผลลัพธ์แต่ละคลาส

### 3.2 `loop()` — วนอัปเดตเฟรมและทำนายผล

```js
async function loop(timestamp) {
    webcam.update();
    await predict();
    window.requestAnimationFrame(loop);
}
```

เรียกซ้ำทุกเฟรม (ประสาน frame rate กับหน้าจอผ่าน `requestAnimationFrame`)
เพื่ออัปเดตภาพจากเว็บแคมและรันการทำนายท่าทางต่อเนื่อง

### 3.3 `predict()` — ทำนายท่าทางและควบคุมกล้อง

```js
const { pose, posenetOutput } = await model.estimatePose(webcam.canvas);
const prediction = await model.predict(posenetOutput);
```

- `estimatePose` วิเคราะห์ keypoints ของร่างกายจากภาพเว็บแคม
- `predict` นำ keypoints เหล่านั้นไปจำแนกว่าตรงกับคลาสใด พร้อมค่า
  probability ของแต่ละคลาส

```js
const leftProbability  = prediction.find(p => p.className === "left").probability;
const rightProbability = prediction.find(p => p.className === "right").probability;
const walkProbability  = prediction.find(p => p.className === "walk").probability;

const camera = document.querySelector('[camera]');
const currentPosition = camera.getAttribute('position');

if (leftProbability  > 0.5) camera.setAttribute('position', { x: currentPosition.x - 0.5, y: currentPosition.y, z: currentPosition.z });
if (rightProbability > 0.5) camera.setAttribute('position', { x: currentPosition.x + 0.5, y: currentPosition.y, z: currentPosition.z });
if (walkProbability  > 0.5) camera.setAttribute('position', { x: currentPosition.x,       y: currentPosition.y, z: currentPosition.z - 0.5 });
```

ถ้าความน่าจะเป็นของคลาสใดคลาสหนึ่งเกิน 0.5 (มั่นใจมากกว่า 50%) จะขยับ
ตำแหน่งกล้องในแกนที่สอดคล้องกันทีละ 0.5 หน่วยต่อเฟรม:

| คลาส | แกนที่เปลี่ยน | ทิศทาง |
|---|---|---|
| `left` | X | ลบ (เลื่อนซ้าย) |
| `right` | X | บวก (เลื่อนขวา) |
| `walk` | Z | ลบ (เดินหน้าเข้าฉาก) |

> **หมายเหตุ:** ชื่อคลาสในโค้ด (`"left"`, `"right"`, `"walk"`) ต้อง
> **ตรงกับชื่อคลาสที่ตั้งไว้ใน Teachable Machine ทุกตัวอักษร** (case-sensitive)
> ไม่เช่นนั้น `prediction.find(...)` จะคืนค่า `undefined` และโค้ดจะ error
> ที่ `.probability`

### 3.4 `drawPose()` — วาดผลลัพธ์ลง canvas

```js
function drawPose(pose) {
    if (webcam.canvas) {
        ctx.drawImage(webcam.canvas, 0, 0);
        if (pose) {
            const minPartConfidence = 0.5;
            tmPose.drawKeypoints(pose.keypoints, minPartConfidence, ctx);
            tmPose.drawSkeleton(pose.keypoints, minPartConfidence, ctx);
        }
    }
}
```

วาดภาพจากเว็บแคมลงบน `<canvas>` แล้ว overlay จุด keypoints และเส้น
skeleton ของท่าทางที่ตรวจจับได้ (เฉพาะจุดที่มี confidence ≥ 0.5) ทับลงไป
เพื่อให้เห็นว่าโมเดลตรวจจับท่าทางถูกต้องหรือไม่

---

## วิธีรันโปรเจกต์

**ต้องรันผ่าน local web server เท่านั้น** (ไม่สามารถเปิดไฟล์ `index.html`
ตรงๆ แบบ `file://` ได้ เพราะ browser จะบล็อกการขอสิทธิ์กล้องบน origin
แบบ `file://` และการโหลดโมเดลผ่าน `fetch` ก็ต้องใช้ `http://`)

```bash
cd path/to/workshop_04
python3 -m http.server 5500
```

จากนั้นเปิดเบราว์เซอร์ไปที่:

```
http://127.0.0.1:5500/index.html
```

กดปุ่ม **Start** → อนุญาตการเข้าถึงกล้องเมื่อเบราว์เซอร์ถาม → ทำท่าทาง
ตามคลาสที่ฝึกไว้เพื่อควบคุมกล้องในฉาก 3 มิติ

---

## การแก้ปัญหาที่พบบ่อย (Troubleshooting)

| อาการ | สาเหตุที่เป็นไปได้ | วิธีแก้ |
|---|---|---|
| กด Start แล้วไม่มีอะไรเกิดขึ้น ไม่มี popup ขอสิทธิ์กล้อง | ไม่พบ `model.json` (404) ทำให้ `init()` throw error ก่อนถึงส่วนขอสิทธิ์กล้อง | ตรวจสอบว่ามีไฟล์ `model.json`, `metadata.json`, `weights.bin` อยู่ในโฟลเดอร์เดียวกับ `index.html` แล้วเปิด Console (Cmd+Option+J) ดู error |
| Error `Cannot read properties of undefined (reading 'probability')` | ชื่อคลาสในโค้ด (`left`/`right`/`walk`) ไม่ตรงกับชื่อคลาสที่ตั้งใน Teachable Machine | ตรวจสอบชื่อคลาสให้ตรงกันทุกตัวอักษร (case-sensitive) ทั้งใน `metadata.json` และใน `app.js` |
| ไม่มี popup ขอสิทธิ์กล้อง / กล้องไม่เปิด | เปิดไฟล์ผ่าน `file://` แทนที่จะรันผ่าน local server | รันผ่าน `python3 -m http.server` แล้วเปิดผ่าน `http://127.0.0.1:...` |
| โมเดล 3 มิติ (`.glb`) แสดงผลผิดเพี้ยน/ลอย/จมพื้น | ไฟล์ต้นฉบับใช้แกนขึ้น (up-axis) เป็น Z-up แต่ A-Frame/three.js ใช้ Y-up | ปรับ `rotation` ของ `<a-entity gltf-model>` ให้เหมาะสม (มักใช้ `-90 0 0`) และปรับตำแหน่งกล้องให้อยู่ในช่วงพิกัดจริงของโมเดล |
| ทำนายผลไม่แม่นยำ / สลับคลาสไปมา | จำนวนตัวอย่างต่อคลาสน้อยเกินไป หรือไม่มีคลาสพื้นหลัง | เพิ่มจำนวนภาพตัวอย่างต่อคลาส และเพิ่มคลาส `stop`/background ไว้เสมอ |

---

## แหล่งอ้างอิง

- Teachable Machine: https://teachablemachine.withgoogle.com/
- A-Frame: https://aframe.io/
- TensorFlow.js: https://www.tensorflow.org/js
- เอกสารประกอบวิชา: *DT508-WE674-14 — Machine Learning: การเรียนรู้ด้วย
  เครื่องและการประยุกต์ใช้ปัญญาประดิษฐ์สำหรับการพัฒนาแอปพลิเคชัน*
  (ผู้ช่วยศาสตราจารย์ บัญญพนต์ พูลสวัสดิ์)
