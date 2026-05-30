<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>สร้างตู้ปลาเสมือนจริงด้วยตัวเอง</title>
    <script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-database-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-storage-compat.js"></script>
    
    <style>
        body { margin: 0; padding: 0; font-family: sans-serif; overflow: hidden; background: #000; }
        #view-selection { display: flex; justify-content: center; align-items: center; height: 100vh; gap: 20px; background: #222; }
        button { padding: 20px 40px; font-size: 1.5rem; cursor: pointer; border-radius: 10px; border: none; background: #00bcd4; color: white; font-weight: bold; }
        
        /* หน้าจออควาเรียม (TV) */
        #aquarium-container { display: none; width: 100vw; height: 100vh; position: relative; }
        #aquariumCanvas { width: 100%; height: 100%; display: block; }
        
        /* หน้าจอสแกน (Mobile) */
        #scanner-container { display: none; padding: 20px; text-align: center; color: white; }
        #webcam { width: 100%; max-width: 500px; border: 4px solid #fff; border-radius: 10px; transform: scaleX(-1); }
        #canvas-process { display: none; }
    </style>
</head>
<body>

    <div id="view-selection">
        <button onclick="startAquarium()">เปิดหน้าจอ ตู้ปลา (TV)</button>
        <button onclick="startScanner()">เปิดหน้าจอ สแกนปลา (มือถือ)</button>
    </div>

    <div id="aquarium-container">
        <canvas id="aquariumCanvas"></canvas>
    </div>

    <div id="scanner-container">
        <h2>📷 วางกระดาษระบายสีให้อยู่ในกรอบแล้วกดสแกน</h2>
        <video id="webcam" autoplay playsinline></video>
        <br><br>
        <button onclick="captureAndUpload()">🚀 สแกนและส่งปลาลงตู้</button>
        <canvas id="canvas-process"></canvas>
    </div>

    <script>
        // === [ ส่วนที่ 1: ตั้งค่า FIREBASE ของคุณตรงนี้ ] ===
        const firebaseConfig = {
            apiKey: "YOUR_API_KEY",
            authDomain: "YOUR_AUTH_DOMAIN",
            databaseURL: "YOUR_DATABASE_URL",
            projectId: "YOUR_PROJECT_ID",
            storageBucket: "YOUR_STORAGE_BUCKET",
            messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
            appId: "YOUR_APP_ID"
        };
        // เริ่มต้น Firebase (เปิดคอมเมนต์เมื่อใส่รหัสผ่านเรียบร้อย)
        // firebase.initializeApp(firebaseConfig);
        // const database = firebase.database();
        // const storage = firebase.storage();

        // === [ ส่วนที่ 2: ระบบจัดการโหมดหน้าจอ ] ===
        function startAquarium() {
            document.getElementById('view-selection').style.display = 'none';
            document.getElementById('aquarium-container').style.display = 'block';
            initAquarium();
        }

        function startScanner() {
            document.getElementById('view-selection').style.display = 'none';
            document.getElementById('scanner-container').style.display = 'block';
            initScanner();
        }

        // === [ ส่วนที่ 3: ระบบตู้ปลา (AQUARIUM LOGIC) ] ===
        let canvas, ctx, fishes = [];
        const bgImage = new Image();
        bgImage.src = 'https://images.unsplash.com/photo-1544551763-46a013bb70d5?q=80&w=1920'; // <-- เปลี่ยนเป็น URL รูปพื้นหลังของคุณได้ที่นี่

        function initAquarium() {
            canvas = document.getElementById('aquariumCanvas');
            ctx = canvas.getContext('2d');
            resizeCanvas();
            window.addEventListener('resize', resizeCanvas);

            // ดักฟังข้อมูลจาก Firebase เมื่อมีคนส่งปลามาใหม่
            // database.ref('fishes').on('child_added', (snapshot) => {
            //     const fishData = snapshot.val();
            //     addFishToAquarium(fishData.url);
            // });

            // (จำลอง) ปล่อยปลาตัวอย่างระบบ
            addFishToAquarium('https://i.imgur.com/v8tX8Yp.png'); 

            animate();
        }

        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }

        function addFishToAquarium(imgUrl) {
            const fishImg = new Image();
            fishImg.crossOrigin = "anonymous";
            fishImg.src = imgUrl;
            fishImg.onload = () => {
                fishes.push({
                    img: fishImg,
                    x: Math.random() * canvas.width,
                    y: Math.random() * (canvas.height - 200) + 100,
                    speedX: (Math.random() * 2 + 1) * (Math.random() > 0.5 ? 1 : -1),
                    speedY: (Math.random() * 0.5 - 0.25),
                    width: 150,
                    height: 100
                });
            };
        }

        function animate() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // วาดรูปพื้นหลังอควาเรียมของคุณ
            ctx.drawImage(bgImage, 0, 0, canvas.width, canvas.height);

            // คำนวณและวาดปลาแต่ละตัว
            fishes.forEach(fish => {
                fish.x += fish.speedX;
                fish.y += fish.speedY;

                // ชนขอบซ้าย-ขวาให้กลับตัว
                if (fish.x < -fish.width || fish.x > canvas.width + fish.width) {
                    fish.speedX *= -1;
                }
                // ชนขอบบน-ล่าง
                if (fish.y < 50 || fish.y > canvas.height - 150) {
                    fish.speedY *= -1;
                }

                ctx.save();
                // กลับด้านรูปปลาตามทิศทางที่ว่าย
                if (fish.speedX < 0) {
                    ctx.translate(fish.x + fish.width, fish.y);
                    ctx.scale(-1, 1);
                    ctx.drawImage(fish.img, 0, 0, fish.width, fish.height);
                } else {
                    ctx.drawImage(fish.img, fish.x, fish.y, fish.width, fish.height);
                }
                ctx.restore();
            });

            requestAnimationFrame(animate);
        }

        // === [ ส่วนที่ 4: ระบบสแกนและตัดสีพื้นหลัง (SCANNER & LOGIC) ] ===
        let video;
        function initScanner() {
            video = document.getElementById('webcam');
            navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" }, audio: false })
                .then(stream => { video.srcObject = stream; })
                .catch(err => { alert("ไม่สามารถเปิดกล้องได้: " + err); });
        }

        function captureAndUpload() {
            const processCanvas = document.getElementById('canvas-process');
            const pCtx = processCanvas.getContext('2d');
            
            // ตั้งขนาดการดึงรูปปลา (ดึงเฉพาะช่วงกลางหน้าจอที่จัดวางกระดาษ)
            processCanvas.width = 400;
            processCanvas.height = 300;
            
            // ถ่ายภาพจากวิดีโอลงคอมพิวเตอร์
            pCtx.drawImage(video, 80, 90, 480, 360, 0, 0, 400, 300);

            // อัลกอริทึมลบพื้นหลังสีขาว (Chroma Key สีขาว)
            const imgData = pCtx.getImageData(0, 0, processCanvas.width, processCanvas.height);
            const data = imgData.data;
            
            for (let i = 0; i < data.length; i += 4) {
                let r = data[i];
                let g = data[i+1];
                let b = data[i+2];
                
                // ถ้าพิกเซลไหนเข้าใกล้สีขาว (กระดาษ) ให้ทำเป็นโปร่งแสง (Transparent)
                if (r > 190 && g > 190 && b > 190) { 
                    data[i+3] = 0; // เปลี่ยนค่า Alpha เป็น 0
                }
            }
            pCtx.putImageData(imgData, 0, 0);

            // แปลงภาพเป็นไฟล์ Base64 หรือ PNG
            const fishBase64 = processCanvas.toDataURL('image/png');

            // --- ขั้นตอนการอัปโหลดขึ้น Firebase ---
            // 1. นำข้อมูลไปเซฟลง Firebase Storage
            // 2. นำดาวน์โหลด URL ที่ได้ ไปบันทึกลง Realtime Database
            // 3. หน้าจอทีวี (Aquarium) จะได้รับภาพและปล่อยปลาว่ายน้ำทันที
            
            alert("สแกนปลาสำเร็จ! (เชื่อมต่อ Firebase เพื่อส่งลงตู้ปลาจริงได้เลย)");
            
            // ตัวอย่างจำลองส่งเข้าตู้ปลาภายในหน้าเดียวกันเพื่อทดสอบ
            if(document.getElementById('aquarium-container').style.display === 'none') {
                addFishToAquarium(fishBase64);
            }
        }
    </script>
</body>
</html>
