
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <!-- Google Sign-In Client ID (실제 값 적용) -->
  <meta name="google-signin-client_id" content="171514115990-llkmtm1154n8p257smbihuja1sn56vgo.apps.googleusercontent.com">
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>3D 캐릭터 HUD, 달력, 채팅 & 말풍선 (Google 로그인만 사용)</title>
  
  <!-- Google Sign-In 라이브러리 (구버전) -->
  <script src="https://apis.google.com/js/platform.js" async defer></script>
  
  <style>
    /* CSS Reset 및 모든 요소에 box-sizing 적용 */
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      font-family: Arial, sans-serif;
      overflow: hidden;
    }
    /* Google Sign-In 버튼 컨테이너 (화면 오른쪽 상단 고정) */
    #google-signin {
      position: fixed;
      top: 10px;
      right: 10px;
      z-index: 30;
    }
    /* 오른쪽 HUD: 채팅 및 기타 기능 (화면 오른쪽에 고정) */
    #right-hud {
      position: fixed;
      top: 70px; /* 로그인 버튼 아래 */
      right: 10px;
      padding: 10px;
      background: rgba(255,255,255,0.8);
      border-radius: 5px;
      z-index: 20;
      width: 300px;
    }
    /* 왼쪽 HUD: 달력 UI (화면 왼쪽에 고정) */
    #left-hud {
      position: fixed;
      top: 70px;
      left: 10px;
      padding: 10px;
      background: rgba(255,255,255,0.9);
      border-radius: 5px;
      z-index: 20;
      width: 320px;
      max-height: 90vh;
      overflow-y: auto;
    }
    /* 달력 UI 스타일 */
    #calendar-container { margin-top: 10px; }
    #calendar-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 5px;
    }
    #calendar-header button { padding: 2px 6px; font-size: 12px; }
    #month-year-label { font-weight: bold; font-size: 14px; }
    #year-select { font-size: 12px; padding: 2px; margin-left: 5px; }
    #calendar-grid {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      gap: 2px;
    }
    #calendar-grid div {
      border: 1px solid #ccc;
      min-height: 40px;
      font-size: 12px;
      padding: 2px;
      position: relative;
      cursor: pointer;
    }
    #calendar-grid div:hover { background: #f0f0f0; }
    .day-number { position: absolute; top: 2px; left: 2px; font-weight: bold; }
    .event {
      margin-top: 18px;
      font-size: 10px;
      color: #333;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    /* 채팅 로그 */
    #chat-log {
      height: 100px;
      overflow-y: scroll;
      border: 1px solid #ccc;
      padding: 5px;
      margin-top: 10px;
    }
    /* 채팅 입력 영역 */
    #chat-input-area {
      display: flex;
      margin-top: 10px;
    }
    #chat-input { flex: 1; padding: 5px; font-size: 14px; }
    #send-chat-button { padding: 5px 10px; font-size: 14px; margin-left: 5px; }
    /* 3D 캔버스 (배경에 고정) */
    #canvas {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      z-index: 1;
      display: block;
    }
    /* 말풍선 스타일 */
    #speech-bubble {
      position: fixed;
      background: white;
      padding: 5px 10px;
      border-radius: 10px;
      font-size: 12px;
      display: none;
      z-index: 30;
      white-space: pre-line;
      pointer-events: none;
    }
  </style>
  
  <script>
    // 초기 Google 인증 초기화
    function initGoogleAuth() {
      gapi.load('auth2', function() {
        gapi.auth2.init({
          client_id: "171514115990-llkmtm1154n8p257smbihuja1sn56vgo.apps.googleusercontent.com"
        }).then(() => {
          console.log("Google auth 초기화 완료");
        }, (error) => {
          console.error("Google auth 초기화 에러:", error);
        });
      });
    }

    // 전역 변수: Google 로그인 후 받아온 내 이메일
    let userProfileEmail = "";
    
    // Google Sign-In 성공 시 호출되는 함수
    function onSignIn(googleUser) {
      const profile = googleUser.getBasicProfile();
      userProfileEmail = profile.getEmail();
      document.getElementById("login-status").textContent = "로그인한 이메일: " + userProfileEmail;
      console.log("Google 로그인 성공:", userProfileEmail);
    }
    
    // 채팅 관련 함수
    function appendToChatLog(message) {
      const chatLog = document.getElementById("chat-log");
      chatLog.innerHTML += "<div>" + message + "</div>";
      chatLog.scrollTop = chatLog.scrollHeight;
    }
    
    async function sendChat() {
      const inputEl = document.getElementById("chat-input");
      const input = inputEl.value.trim();
      let response = "";
      if (!input) return;
      const lowerInput = input.toLowerCase();
      if (lowerInput.includes("안녕")) {
        response = "안녕하세요, 주인님! 오늘 기분은 어떠세요?";
      } else if (lowerInput.includes("캐릭터 넌 누구야")) {
        response = "저는 당신의 개인 비서에요.";
      } else if (lowerInput.includes("일정")) {
        response = "캘린더는 좌측에서 확인하세요.";
      } else if (lowerInput.includes("날씨") && (lowerInput.includes("알려") || lowerInput.includes("어때"))) {
        response = "현재 날씨는 맑음입니다.";
      } else if (lowerInput.includes("캐릭터 춤춰줘")) {
        response = "춤출게요!";
        const danceInterval = setInterval(() => {
          // 예: 간단한 팔 회전 애니메이션
        }, 50);
        setTimeout(() => { clearInterval(danceInterval); }, 3000);
      } else {
        response = "죄송해요, 잘 이해하지 못했어요. 다시 한 번 말씀해주시겠어요?";
      }
      appendToChatLog("사용자: " + input);
      appendToChatLog("캐릭터: " + response);
      showSpeechBubbleInChunks(response);
      inputEl.value = "";
    }
    
    document.getElementById("chat-input").addEventListener("keydown", function(e) {
      if (e.key === "Enter") { sendChat(); }
    });
    
    function showSpeechBubbleInChunks(text, chunkSize = 15, delay = 3000) {
      const bubble = document.getElementById("speech-bubble");
      const chunks = [];
      for (let i = 0; i < text.length; i += chunkSize) {
        chunks.push(text.slice(i, i + chunkSize));
      }
      let index = 0;
      function showNextChunk() {
        if (index < chunks.length) {
          bubble.textContent = chunks[index];
          bubble.style.display = "block";
          index++;
          setTimeout(showNextChunk, delay);
        } else {
          setTimeout(() => { bubble.style.display = "none"; }, 3000);
        }
      }
      showNextChunk();
    }
    
    // 윈도우 리사이즈 이벤트 - 캔버스와 카메라 업데이트
    window.addEventListener("resize", function(){
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  </script>
</head>
<body onload="initGoogleAuth()">
  <!-- Google Sign-In 버튼 (오른쪽 상단) -->
  <div id="google-signin">
    <div class="g-signin2" data-onsuccess="onSignIn"></div>
  </div>
  
  <!-- 오른쪽 HUD: 채팅 및 기타 기능 -->
  <div id="right-hud">
    <!-- 로그인 상태 표시 -->
    <div id="login-status">로그인 전</div>
    <h3>채팅창</h3>
    <div id="chat-log"></div>
    <div id="chat-input-area">
      <input type="text" id="chat-input" placeholder="채팅 입력..." />
      <button id="send-chat-button" onclick="sendChat()">전송</button>
    </div>
  </div>
  
  <!-- 왼쪽 HUD: 달력 UI -->
  <div id="left-hud">
    <h3>캘린더</h3>
    <div id="calendar-container">
      <div id="calendar-header">
        <button id="prev-month">◀</button>
        <span id="month-year-label"></span>
        <button id="next-month">▶</button>
        <select id="year-select"></select>
      </div>
      <div id="calendar-grid"></div>
    </div>
  </div>
  
  <!-- 말풍선 (3D 캐릭터 말풍선) -->
  <div id="speech-bubble"></div>
  
  <!-- 3D 캔버스 -->
  <canvas id="canvas"></canvas>
  
  <script>
    /* ====================================
       3D 씬 설정 (캐릭터, 배경, 날씨 효과 등)
    ==================================== */
    let currentWeather = "";
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById("canvas"), alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.position.set(5, 5, 10);
    camera.lookAt(0, 0, 0);
    
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
    directionalLight.position.set(5, 10, 7).normalize();
    scene.add(directionalLight);
    scene.add(new THREE.AmbientLight(0x333333));
    
    // 태양 객체
    const sunMaterial = new THREE.MeshStandardMaterial({ color: 0xffcc00, emissive: 0xff9900, transparent: true, opacity: 0 });
    const sun = new THREE.Mesh(new THREE.SphereGeometry(1.5, 64, 64), sunMaterial);
    scene.add(sun);
    
    // 달 객체
    const moonMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc, emissive: 0x222222, transparent: true, opacity: 1 });
    const moon = new THREE.Mesh(new THREE.SphereGeometry(1.2, 64, 64), moonMaterial);
    scene.add(moon);
    
    // 별, 반딧불 생성
    const stars = [], fireflies = [];
    for (let i = 0; i < 100; i++) {
      const star = new THREE.Mesh(new THREE.SphereGeometry(0.03, 8, 8), new THREE.MeshBasicMaterial({ color: 0xffffff }));
      star.position.set((Math.random()-0.5)*50, (Math.random()-0.5)*30, -10);
      scene.add(star);
      stars.push(star);
    }
    for (let i = 0; i < 30; i++) {
      const firefly = new THREE.Mesh(new THREE.SphereGeometry(0.05, 8, 8), new THREE.MeshBasicMaterial({ color: 0xffff99 }));
      firefly.position.set((Math.random()-0.5)*20, (Math.random()-0.5)*10, -5);
      scene.add(firefly);
      fireflies.push(firefly);
    }
    
    // 고해상도 콩크리트 바닥 (Y = -2)
    const floorGeometry = new THREE.PlaneGeometry(200, 200, 128, 128);
    const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x808080, roughness: 1, metalness: 0 });
    const floor = new THREE.Mesh(floorGeometry, floorMaterial);
    floor.rotation.x = -Math.PI/2;
    floor.position.y = -2;
    scene.add(floor);
    
    // 배경 그룹 (빌딩, 집, 가로등)
    const backgroundGroup = new THREE.Group();
    scene.add(backgroundGroup);
    function createBuilding(width, height, depth, color) {
      const geometry = new THREE.BoxGeometry(width, height, depth);
      const material = new THREE.MeshStandardMaterial({ color: color, roughness: 0.7, metalness: 0.1 });
      return new THREE.Mesh(geometry, material);
    }
    function createHouse(width, height, depth, baseColor, roofColor) {
      const houseGroup = new THREE.Group();
      const base = new THREE.Mesh(new THREE.BoxGeometry(width, height, depth),
                                  new THREE.MeshStandardMaterial({ color: baseColor, roughness: 0.8 }));
      base.position.y = -2 + height/2;
      houseGroup.add(base);
      const roof = new THREE.Mesh(new THREE.ConeGeometry(width * 0.8, height * 0.6, 4),
                                  new THREE.MeshStandardMaterial({ color: roofColor, roughness: 0.8 }));
      roof.position.y = -2 + height + (height * 0.6)/2;
      roof.rotation.y = Math.PI/4;
      houseGroup.add(roof);
      return houseGroup;
    }
    // 빌딩 배치 (5열×2행)
    for (let i = 0; i < 10; i++) {
      const width = Math.random() * 2 + 2;
      const height = Math.random() * 10 + 10;
      const depth = Math.random() * 2 + 2;
      const building = createBuilding(width, height, depth, 0x555555);
      const col = i % 5;
      const row = Math.floor(i / 5);
      const x = -20 + col * 10;
      const z = -15 - row * 10;
      building.position.set(x, -2 + height/2, z);
      backgroundGroup.add(building);
    }
    // 집 배치 (1행, 캐릭터 뒤쪽, Z = -5)
    for (let i = 0; i < 5; i++) {
      const width = Math.random() * 2 + 3;
      const height = Math.random() * 2 + 3;
      const depth = Math.random() * 2 + 3;
      const house = createHouse(width, height, depth, 0xa0522d, 0x8b0000);
      const x = -10 + i * 10;
      const z = -5;
      house.position.set(x, 0, z);
      backgroundGroup.add(house);
    }
    // 단일 가로등: 캐릭터 옆에 배치
    function createStreetlight() {
      const lightGroup = new THREE.Group();
      const pole = new THREE.Mesh(new THREE.CylinderGeometry(0.1, 0.1, 4, 8),
                                    new THREE.MeshBasicMaterial({ color: 0x333333 }));
      pole.position.y = 2;
      lightGroup.add(pole);
      const lamp = new THREE.Mesh(new THREE.SphereGeometry(0.2, 8, 8),
                                    new THREE.MeshBasicMaterial({ color: 0xffcc00 }));
      lamp.position.y = 4.2;
      lightGroup.add(lamp);
      const lampLight = new THREE.PointLight(0xffcc00, 1, 10);
      lampLight.position.set(0, 4.2, 0);
      lightGroup.add(lampLight);
      return lightGroup;
    }
    const characterStreetlight = createStreetlight();
    characterStreetlight.position.set(1, -2, 0);
    scene.add(characterStreetlight);
    
    // 날씨 효과 – 비
    let rainGroup = new THREE.Group();
    scene.add(rainGroup);
    function initRain() {
      const rainCount = 1000;
      const rainGeometry = new THREE.BufferGeometry();
      const positions = new Float32Array(rainCount * 3);
      for (let i = 0; i < rainCount; i++) {
        positions[i * 3] = Math.random() * 100 - 50;
        positions[i * 3 + 1] = Math.random() * 50;
        positions[i * 3 + 2] = Math.random() * 100 - 50;
      }
      rainGeometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
      const rainMaterial = new THREE.PointsMaterial({ color: 0xaaaaee, size: 0.1, transparent: true, opacity: 0.6 });
      const rainParticles = new THREE.Points(rainGeometry, rainMaterial);
      rainGroup.add(rainParticles);
    }
    initRain();
    rainGroup.visible = false;
    
    // 날씨 효과 – 구름 (하나의 구름)
    let houseCloudGroup = new THREE.Group();
    function createHouseCloud() {
      const cloud = new THREE.Group();
      const cloudMat = new THREE.MeshLambertMaterial({ color: 0xffffff, transparent: true, opacity: 0.9 });
      const sphere1 = new THREE.Mesh(new THREE.SphereGeometry(2, 32, 32), cloudMat);
      sphere1.position.set(0, 0, 0);
      const sphere2 = new THREE.Mesh(new THREE.SphereGeometry(1.8, 32, 32), cloudMat);
      sphere2.position.set(2.2, 0.7, 0);
      const sphere3 = new THREE.Mesh(new THREE.SphereGeometry(2.1, 32, 32), cloudMat);
      sphere3.position.set(-2.2, 0.5, 0);
      cloud.add(sphere1, sphere2, sphere3);
      cloud.userData.initialPos = cloud.position.clone();
      return cloud;
    }
    const singleCloud = createHouseCloud();
    houseCloudGroup.add(singleCloud);
    houseCloudGroup.position.set(0, 5, -10);
    scene.add(houseCloudGroup);
    function updateHouseClouds() {
      singleCloud.position.x += 0.02;
      if (singleCloud.position.x > 5) { singleCloud.position.x = -5; }
    }
    
    // 날씨 효과 – 번개
    let lightningLight = new THREE.PointLight(0xffffff, 0, 500);
    lightningLight.position.set(0, 50, 0);
    scene.add(lightningLight);
    function updateWeatherEffects() {
      if (currentWeather.indexOf("비") !== -1 || currentWeather.indexOf("소나기") !== -1) {
        rainGroup.visible = true;
      } else {
        rainGroup.visible = false;
      }
      if (currentWeather.indexOf("구름") !== -1) {
        houseCloudGroup.visible = true;
      } else {
        houseCloudGroup.visible = false;
      }
    }
    
    // 캐릭터 생성
    const characterGroup = new THREE.Group();
    const charBody = new THREE.Mesh(new THREE.BoxGeometry(1, 1.5, 0.5), new THREE.MeshStandardMaterial({ color: 0x00cc66 }));
    const head = new THREE.Mesh(new THREE.SphereGeometry(0.5, 32, 32), new THREE.MeshStandardMaterial({ color: 0xffcc66 }));
    head.position.y = 1.2;
    const eyeMat = new THREE.MeshBasicMaterial({ color: 0x000000 });
    const leftEye = new THREE.Mesh(new THREE.SphereGeometry(0.07, 16, 16), eyeMat);
    const rightEye = new THREE.Mesh(new THREE.SphereGeometry(0.07, 16, 16), eyeMat);
    leftEye.position.set(-0.2, 1.3, 0.45);
    rightEye.position.set(0.2, 1.3, 0.45);
    const mouth = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.05, 0.05), new THREE.MeshStandardMaterial({ color: 0xff3366 }));
    mouth.position.set(0, 1.1, 0.51);
    const leftBrow = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.05, 0.05), eyeMat);
    const rightBrow = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.05, 0.05), eyeMat);
    leftBrow.position.set(-0.2, 1.45, 0.45);
    rightBrow.position.set(0.2, 1.45, 0.45);
    const leftArm = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1, 0.2), charBody.material);
    const rightArm = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1, 0.2), charBody.material);
    leftArm.position.set(-0.7, 0.4, 0);
    rightArm.position.set(0.7, 0.4, 0);
    const legMat = new THREE.MeshStandardMaterial({ color: 0x3366cc });
    const leftLeg = new THREE.Mesh(new THREE.BoxGeometry(0.3, 1, 0.3), legMat);
    const rightLeg = new THREE.Mesh(new THREE.BoxGeometry(0.3, 1, 0.3), legMat);
    leftLeg.position.set(-0.35, -1, 0);
    rightLeg.position.set(0.35, -1, 0);
    characterGroup.add(charBody, head, leftEye, rightEye, mouth, leftBrow, rightBrow, leftArm, rightArm, leftLeg, rightLeg);
    characterGroup.position.y = -1;
    scene.add(characterGroup);
    const characterLight = new THREE.PointLight(0xffee88, 1, 15);
    scene.add(characterLight);
    
    /* ====================================
       채팅 관련 함수
    ==================================== */
    async function getWeather() { return "맑음"; }
    function appendToChatLog(message) {
      const chatLog = document.getElementById("chat-log");
      chatLog.innerHTML += "<div>" + message + "</div>";
      chatLog.scrollTop = chatLog.scrollHeight;
    }
    async function sendChat() {
      const inputEl = document.getElementById("chat-input");
      const input = inputEl.value.trim();
      let response = "";
      if (!input) return;
      const lowerInput = input.toLowerCase();
      if (lowerInput.includes("안녕")) {
        response = "안녕하세요, 주인님! 오늘 기분은 어떠세요?";
      } else if (lowerInput.includes("캐릭터 넌 누구야")) {
        response = "저는 당신의 개인 비서에요.";
      } else if (lowerInput.includes("일정")) {
        response = "캘린더는 좌측에서 확인하세요.";
      } else if (lowerInput.includes("날씨") && (lowerInput.includes("알려") || lowerInput.includes("어때"))) {
        const weather = await getWeather();
        response = `현재 날씨는 ${weather}입니다.`;
      } else if (lowerInput.includes("캐릭터 춤춰줘")) {
        response = "춤출게요!";
        const danceInterval = setInterval(() => {
          // 간단한 춤 동작 효과 (예: 팔 회전)
        }, 50);
        setTimeout(() => {
          clearInterval(danceInterval);
        }, 3000);
      } else {
        response = "죄송해요, 잘 이해하지 못했어요. 다시 한 번 말씀해주시겠어요?";
      }
      appendToChatLog("사용자: " + input);
      appendToChatLog("캐릭터: " + response);
      showSpeechBubbleInChunks(response);
      inputEl.value = "";
    }
    document.getElementById("chat-input").addEventListener("keydown", function(e) {
      if (e.key === "Enter") { sendChat(); }
    });
    setInterval(() => {
      const now = new Date();
      if (now.getHours() === 8 && now.getMinutes() === 0) {
        appendToChatLog("캐릭터: 주인님, 일어날 시간이에요!");
      } else if (now.getHours() === 12 && now.getMinutes() === 0) {
        appendToChatLog("캐릭터: 식사하실 시간이에요!");
      } else if (now.getHours() === 22 && now.getMinutes() === 0) {
        appendToChatLog("캐릭터: 주무실 시간이에요 zzzz");
      }
    }, 60000);
    
    /* ====================================
       애니메이션 루프 (3D 씬 업데이트)
    ==================================== */
    function animate() {
      requestAnimationFrame(animate);
      const now = new Date();
      const headWorldPos = new THREE.Vector3();
      head.getWorldPosition(headWorldPos);
      const orbitCenter = headWorldPos.clone().add(new THREE.Vector3(0, 2, 0));
      const totalMin = now.getHours() * 60 + now.getMinutes();
      const angle = (totalMin / 1440) * Math.PI * 2;
      const radius = 3;
      const sunPos = new THREE.Vector3(
        orbitCenter.x + Math.cos(angle) * radius,
        orbitCenter.y + Math.sin(angle) * radius,
        orbitCenter.z
      );
      sun.position.copy(sunPos);
      const moonAngle = angle + Math.PI;
      const moonPos = new THREE.Vector3(
        orbitCenter.x + Math.cos(moonAngle) * radius,
        orbitCenter.y + Math.sin(moonAngle) * radius,
        orbitCenter.z
      );
      moon.position.copy(moonPos);
      const t = now.getHours() + now.getMinutes() / 60;
      let sunOpacity = 0, moonOpacity = 0;
      if (t < 6) { sunOpacity = 0; moonOpacity = 1; }
      else if (t < 7) { let factor = (t - 6); sunOpacity = factor; moonOpacity = 1 - factor; }
      else if (t < 17) { sunOpacity = 1; moonOpacity = 0; }
      else if (t < 18) { let factor = (t - 17); sunOpacity = 1 - factor; moonOpacity = factor; }
      else { sunOpacity = 0; moonOpacity = 1; }
      sun.material.opacity = sunOpacity;
      moon.material.opacity = moonOpacity;
      const isDay = t >= 7 && t < 17;
      scene.background = new THREE.Color(isDay ? 0x87CEEB : 0x000033);
      stars.forEach(s => s.visible = !isDay);
      fireflies.forEach(f => f.visible = !isDay);
      characterStreetlight.traverse(child => {
        if (child instanceof THREE.PointLight) { child.intensity = isDay ? 0 : 1; }
      });
      characterLight.position.copy(characterGroup.position).add(new THREE.Vector3(0, 5, 0));
      characterLight.intensity = isDay ? 0 : 1;
      characterGroup.position.y = -1;
      characterGroup.rotation.x = 0;
      if (rainGroup.visible) {
        const rainPoints = rainGroup.children[0];
        const positions = rainPoints.geometry.attributes.position.array;
        for (let i = 0; i < positions.length; i += 3) {
          positions[i + 1] -= 0.5;
          if (positions[i + 1] < 0) { positions[i + 1] = Math.random() * 50 + 20; }
        }
        rainPoints.geometry.attributes.position.needsUpdate = true;
      }
      if (currentWeather.indexOf("번개") !== -1 || currentWeather.indexOf("뇌우") !== -1) {
        if (Math.random() < 0.001) {
          lightningLight.intensity = 5;
          setTimeout(() => { lightningLight.intensity = 0; }, 100);
        }
      }
      updateHouseClouds();
      characterStreetlight.position.set(characterGroup.position.x + 1, -2, characterGroup.position.z);
      updateBubblePosition();
      renderer.render(scene, camera);
    }
    animate();
    
    /* ====================================
       달력 UI
    ==================================== */
    let currentYear, currentMonth;
    function initCalendar() {
      const now = new Date();
      currentYear = now.getFullYear();
      currentMonth = now.getMonth();
      populateYearSelect();
      renderCalendar(currentYear, currentMonth);
      document.getElementById("prev-month").addEventListener("click", () => {
        currentMonth--;
        if (currentMonth < 0) { currentMonth = 11; currentYear--; }
        renderCalendar(currentYear, currentMonth);
      });
      document.getElementById("next-month").addEventListener("click", () => {
        currentMonth++;
        if (currentMonth > 11) { currentMonth = 0; currentYear++; }
        renderCalendar(currentYear, currentMonth);
      });
      document.getElementById("year-select").addEventListener("change", (e) => {
        currentYear = parseInt(e.target.value);
        renderCalendar(currentYear, currentMonth);
      });
    }
    function populateYearSelect() {
      const yearSelect = document.getElementById("year-select");
      yearSelect.innerHTML = "";
      for (let y = 2020; y <= 2070; y++) {
        const option = document.createElement("option");
        option.value = y;
        option.textContent = y;
        if (y === currentYear) option.selected = true;
        yearSelect.appendChild(option);
      }
    }
    function renderCalendar(year, month) {
      const monthNames = ["1월", "2월", "3월", "4월", "5월", "6월", "7월", "8월", "9월", "10월", "11월", "12월"];
      document.getElementById("month-year-label").textContent = `${year}년 ${monthNames[month]}`;
      const grid = document.getElementById("calendar-grid");
      grid.innerHTML = "";
      const daysOfWeek = ["일", "월", "화", "수", "목", "금", "토"];
      daysOfWeek.forEach((day) => {
        const th = document.createElement("div");
        th.style.fontWeight = "bold";
        th.style.textAlign = "center";
        th.textContent = day;
        grid.appendChild(th);
      });
      const firstDay = new Date(year, month, 1).getDay();
      const daysInMonth = new Date(year, month + 1, 0).getDate();
      for (let i = 0; i < firstDay; i++) {
        grid.appendChild(document.createElement("div"));
      }
      for (let d = 1; d <= daysInMonth; d++) {
        const cell = document.createElement("div");
        cell.innerHTML = `<div class="day-number">${d}</div>
                          <div class="event" id="event-${year}-${month+1}-${d}"></div>`;
        cell.addEventListener("click", () => {
          const eventText = prompt(`${year}-${month+1}-${d} 일정 입력:`);
          if (eventText) { addEventToDay(`${year}-${month+1}-${d}`, eventText); }
        });
        grid.appendChild(cell);
      }
    }
    function addEventToDay(dateStr, eventText) {
      const eventDiv = document.getElementById(`event-${dateStr}`);
      if (eventDiv) {
        if (eventDiv.textContent) { eventDiv.textContent += "; " + eventText; }
        else { eventDiv.textContent = eventText; }
      }
    }
    window.addEventListener("load", () => {
      initCalendar();
      appendToChatLog("캐릭터: 환영합니다! 무엇을 도와드릴까요?");
    });
    
    // 예제: 말풍선 위치 업데이트 (3D 캐릭터의 머리 위치 기준)
    function updateBubblePosition() {
      const bubble = document.getElementById("speech-bubble");
      const headWorldPos = new THREE.Vector3();
      head.getWorldPosition(headWorldPos);
      const screenPos = headWorldPos.project(camera);
      bubble.style.left = ((screenPos.x * 0.5 + 0.5) * window.innerWidth) + "px";
      bubble.style.top = ((1 - (screenPos.y * 0.5 + 0.5)) * window.innerHeight - 50) + "px";
    }
  </script>
</body>
</html>
