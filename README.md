<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>3D 캐릭터 HUD 인터페이스 & 달력 & Wi‑Fi 알림</title>
  <style>
    body { margin: 0; font-family: Arial, sans-serif; overflow: hidden; }
    /* 오른쪽 HUD: 채팅창 및 핫스팟 연결 버튼 */
    #right-hud {
      position: absolute; top: 10px; right: 10px; padding: 10px;
      background: rgba(255,255,255,0.8); border-radius: 5px; z-index: 20;
      width: 300px;
    }
    /* 왼쪽 HUD: 달력 UI */
    #left-hud {
      position: absolute; top: 10px; left: 10px; padding: 10px;
      background: rgba(255,255,255,0.9); border-radius: 5px; z-index: 20;
      width: 320px; max-height: 90vh; overflow-y: auto;
    }
    /* 달력 UI 스타일 */
    #calendar-container { margin-top: 10px; }
    #calendar-header {
      display: flex; align-items: center; justify-content: space-between;
      margin-bottom: 5px;
    }
    #calendar-header button { padding: 2px 6px; font-size: 12px; }
    #month-year-label { font-weight: bold; font-size: 14px; }
    #year-select { font-size: 12px; padding: 2px; margin-left: 5px; }
    #calendar-grid {
      display: grid; grid-template-columns: repeat(7, 1fr); gap: 2px;
    }
    #calendar-grid div {
      border: 1px solid #ccc; min-height: 40px; font-size: 12px; padding: 2px;
      position: relative; cursor: pointer;
    }
    #calendar-grid div:hover { background: #f0f0f0; }
    .day-number { position: absolute; top: 2px; left: 2px; font-weight: bold; }
    .event { margin-top: 18px; font-size: 10px; color: #333; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    /* 오늘 일정 삭제 버튼 */
    #delete-today {
      display: block; width: 100%; padding: 4px; margin: 5px 0;
      font-size: 12px; background: #f66; color: #fff; border: none; border-radius: 3px;
      cursor: pointer;
    }
    /* 말풍선 (3D 캐릭터 말풍선) */
    #speech-bubble {
      position: absolute; background: white; padding: 5px 10px;
      border-radius: 10px; font-size: 12px; display: none; z-index: 30;
      white-space: pre-line;
    }
    /* 채팅 로그 및 입력 */
    #chat-log { height: 100px; overflow-y: scroll; border: 1px solid #ccc; padding: 5px; margin-top: 10px; }
    #chat-input { width: 100%; padding: 5px; }
    /* 핫스팟 연결 버튼 */
    #hotspot-btn {
      display: block; width: 100%; padding: 5px; margin-top: 10px;
      font-size: 14px; background: #28a745; color: #fff; border: none; border-radius: 3px;
      cursor: pointer;
    }
    /* 3D 캔버스 */
    #canvas { position: absolute; width: 100%; height: 100%; z-index: 1; }
  </style>
</head>
<body>
  <!-- 오른쪽 HUD: 채팅창 및 핫스팟 연결 버튼 -->
  <div id="right-hud">
    <h3>채팅창</h3>
    <div id="chat-log"></div>
    <input type="text" id="chat-input" placeholder="채팅 입력..." />
    <button id="hotspot-btn">핫스팟 연결</button>
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
      <button id="delete-today">오늘 일정 삭제</button>
      <div id="calendar-grid"></div>
    </div>
  </div>
  
  <!-- 말풍선 (3D 캐릭터 말풍선) -->
  <div id="speech-bubble"></div>
  
  <!-- 3D 캔버스 -->
  <canvas id="canvas"></canvas>
  
  <!-- Three.js 라이브러리 -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js"></script>
  <script>
    /******************************
     * 1. 일정 전송 (Wi‑Fi 전송 예시)
     * Wi‑Fi 전송을 위한 엔드포인트 (실제 환경에 맞게 수정)
     ******************************/
    const WIFI_ENDPOINT = "http://192.168.4.1/push";  // 예시 엔드포인트

    async function sendSchedule(message) {
      try {
        const response = await fetch(WIFI_ENDPOINT, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ message })
        });
        console.log("Wi‑Fi 전송 응답:", response.status);
      } catch (err) {
        console.error("Wi‑Fi 전송 오류:", err);
      }
    }

    /******************************
     * 2. 핫스팟 연결 UI
     * 우측 HUD의 "핫스팟 연결" 버튼을 클릭하면, 연결되었다고 가정하고 오늘 일정(저장된)을 전송
     ******************************/
    let isHotspotConnected = false;
    async function connectHotspot() {
      // 실제 핫스팟 연결 코드는 네트워크 환경에 따라 구현되어야 함
      // 여기서는 단순히 연결되었다고 가정하고 isHotspotConnected를 true로 설정
      isHotspotConnected = true;
      alert("핫스팟에 연결되었습니다.");
      // 오늘 일정 자동 전송
      const now = new Date();
      const dateStr = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`;
      const events = localStorage.getItem("events_" + dateStr) || "";
      if (events) {
        await sendSchedule(`[${dateStr}] ${events}`);
      }
    }
    document.getElementById("hotspot-btn").addEventListener("click", connectHotspot);

    /******************************
     * 3. 3D 씬 설정 (캐릭터, 배경, 날씨 효과 등)
     ******************************/
    let currentWeather = "";
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), alpha: true });
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
    const floorMesh = new THREE.Mesh(floorGeometry, floorMaterial);
    floorMesh.rotation.x = -Math.PI/2;
    floorMesh.position.y = -2;
    scene.add(floorMesh);

    // 배경 그룹 (빌딩, 집, 가로등)
    const backgroundGroup = new THREE.Group();
    scene.add(backgroundGroup);
    function createBuilding(w, h, d, color) {
      const geom = new THREE.BoxGeometry(w, h, d);
      const mat = new THREE.MeshStandardMaterial({ color: color, roughness: 0.7, metalness: 0.1 });
      return new THREE.Mesh(geom, mat);
    }
    function createHouse(w, h, d, baseColor, roofColor) {
      const group = new THREE.Group();
      const base = new THREE.Mesh(new THREE.BoxGeometry(w, h, d),
                                  new THREE.MeshStandardMaterial({ color: baseColor, roughness: 0.8 }));
      base.position.y = -2 + h/2;
      group.add(base);
      const roof = new THREE.Mesh(new THREE.ConeGeometry(w*0.8, h*0.6, 4),
                                  new THREE.MeshStandardMaterial({ color: roofColor, roughness: 0.8 }));
      roof.position.y = -2 + h + (h*0.6)/2;
      roof.rotation.y = Math.PI/4;
      group.add(roof);
      return group;
    }
    // 빌딩 배치 (5열×2행)
    for (let i = 0; i < 10; i++) {
      const w = Math.random() * 2 + 2;
      const h = Math.random() * 10 + 10;
      const d = Math.random() * 2 + 2;
      const building = createBuilding(w, h, d, 0x555555);
      const col = i % 5;
      const row = Math.floor(i / 5);
      const x = -20 + col * 10;
      const z = -15 - row * 10;
      building.position.set(x, -2 + h/2, z);
      backgroundGroup.add(building);
    }
    // 집 배치 (1행, 캐릭터 뒤쪽, Z = -5)
    for (let i = 0; i < 5; i++) {
      const w = Math.random() * 2 + 3;
      const h = Math.random() * 2 + 3;
      const d = Math.random() * 2 + 3;
      const house = createHouse(w, h, d, 0xa0522d, 0x8b0000);
      const x = -10 + i * 10;
      house.position.set(x, 0, -5);
      backgroundGroup.add(house);
    }
    
    // 단일 가로등: 캐릭터 바로 옆 (바닥 고정)
    function createStreetlight() {
      const group = new THREE.Group();
      const pole = new THREE.Mesh(new THREE.CylinderGeometry(0.1, 0.1, 4, 8),
                                  new THREE.MeshBasicMaterial({ color: 0x333333 }));
      pole.position.y = 2;
      group.add(pole);
      const lamp = new THREE.Mesh(new THREE.SphereGeometry(0.2, 8, 8),
                                  new THREE.MeshBasicMaterial({ color: 0xffcc00 }));
      lamp.position.y = 4.2;
      group.add(lamp);
      const lampLight = new THREE.PointLight(0xffcc00, 1, 10);
      lampLight.position.set(0, 4.2, 0);
      group.add(lampLight);
      return group;
    }
    const characterStreetlight = createStreetlight();
    characterStreetlight.position.set(1, -2, 0);
    scene.add(characterStreetlight);
    
    // 날씨 효과 – 비
    let rainGroup = new THREE.Group();
    scene.add(rainGroup);
    function initRain() {
      const cnt = 1000;
      const geometry = new THREE.BufferGeometry();
      const positions = new Float32Array(cnt * 3);
      for (let i = 0; i < cnt; i++) {
        positions[i*3] = Math.random() * 100 - 50;
        positions[i*3+1] = Math.random() * 50;
        positions[i*3+2] = Math.random() * 100 - 50;
      }
      geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      const material = new THREE.PointsMaterial({ color: 0xaaaaee, size: 0.1, transparent: true, opacity: 0.6 });
      const particles = new THREE.Points(geometry, material);
      rainGroup.add(particles);
    }
    initRain();
    rainGroup.visible = false;

    // 날씨 효과 – 구름 (단 하나의 고해상도 구름)
    let houseCloudGroup = new THREE.Group();
    function createHouseCloud() {
      const cloud = new THREE.Group();
      const mat = new THREE.MeshLambertMaterial({ color: 0xffffff, transparent: true, opacity: 0.9 });
      const s1 = new THREE.Mesh(new THREE.SphereGeometry(2, 32, 32), mat);
      s1.position.set(0, 0, 0);
      const s2 = new THREE.Mesh(new THREE.SphereGeometry(1.8, 32, 32), mat);
      s2.position.set(2.2, 0.7, 0);
      const s3 = new THREE.Mesh(new THREE.SphereGeometry(2.1, 32, 32), mat);
      s3.position.set(-2.2, 0.5, 0);
      cloud.add(s1, s2, s3);
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
      // 이 예제에서는 currentWeather를 사용하지 않으므로 별도 조정 없음.
    }

    // 캐릭터 생성
    const characterGroup = new THREE.Group();
    const charBody = new THREE.Mesh(new THREE.BoxGeometry(1, 1.5, 0.5),
                                      new THREE.MeshStandardMaterial({ color: 0x00cc66 }));
    const head = new THREE.Mesh(new THREE.SphereGeometry(0.5, 32, 32),
                                  new THREE.MeshStandardMaterial({ color: 0xffcc66 }));
    head.position.y = 1.2;
    const eyeMat = new THREE.MeshBasicMaterial({ color: 0x000000 });
    const leftEye = new THREE.Mesh(new THREE.SphereGeometry(0.07, 16, 16), eyeMat);
    const rightEye = new THREE.Mesh(new THREE.SphereGeometry(0.07, 16, 16), eyeMat);
    leftEye.position.set(-0.2, 1.3, 0.45);
    rightEye.position.set(0.2, 1.3, 0.45);
    const mouth = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.05, 0.05),
                                 new THREE.MeshStandardMaterial({ color: 0xff3366 }));
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

    // 말풍선 관련 함수
    const bubble = document.getElementById('speech-bubble');
    function updateBubblePosition() {
      const headPos = new THREE.Vector3();
      head.getWorldPosition(headPos);
      const screenPos = headPos.project(camera);
      bubble.style.left = `${(screenPos.x * 0.5 + 0.5) * window.innerWidth}px`;
      bubble.style.top = `${(1 - (screenPos.y * 0.5 + 0.5)) * window.innerHeight - 50}px`;
    }
    function showSpeechBubbleInChunks(text, chunkSize = 15, delay = 3000) {
      const chunks = [];
      for (let i = 0; i < text.length; i += chunkSize) {
        chunks.push(text.slice(i, i + chunkSize));
      }
      let index = 0;
      function showNextChunk() {
        if (index < chunks.length) {
          bubble.textContent = chunks[index];
          bubble.style.display = 'block';
          index++;
          setTimeout(showNextChunk, delay);
        } else {
          setTimeout(() => bubble.style.display = 'none', 3000);
        }
      }
      showNextChunk();
    }

    // 채팅 전송: 엔터 시 3D 캐릭터 말풍선 출력 및 캘린더에 일정 추가 (Wi‑Fi 전송 포함)
    async function sendChat() {
      const inputEl = document.getElementById('chat-input');
      const msg = inputEl.value.trim();
      if (!msg) return;
      let response = "";
      const lower = msg.toLowerCase();
      if (lower.includes("안녕")) {
        response = "안녕하세요, 주인님! 오늘 기분은 어떠세요?";
        characterGroup.children[7].rotation.z = Math.PI/4;
        setTimeout(() => { characterGroup.children[7].rotation.z = 0; }, 1000);
      } else if (lower.includes("캐릭터 넌 누구야")) {
        response = "저는 당신의 개인 비서에요 😁";
      } else if (lower.includes("일정")) {
        response = "캘린더는 좌측에서 확인하세요.";
      } else if (lower.includes("날씨") && (lower.includes("알려") || lower.includes("어때"))) {
        const weather = await getWeather();
        response = `현재 날씨는 ${weather}입니다.`;
      } else if (lower.includes("캐릭터 춤춰줘")) {
        response = "춤출게요!";
        const danceInterval = setInterval(() => {
          characterGroup.children[7].rotation.z = Math.sin(Date.now() * 0.01) * Math.PI/4;
          head.rotation.y = Math.sin(Date.now() * 0.01) * Math.PI/8;
        }, 50);
        setTimeout(() => {
          clearInterval(danceInterval);
          characterGroup.children[7].rotation.z = 0;
          head.rotation.y = 0;
        }, 3000);
      } else {
        response = "죄송해요, 잘 이해하지 못했어요. 다시 한 번 말씀해주시겠어요?";
      }
      showSpeechBubbleInChunks(response);
      inputEl.value = '';
    }
    document.getElementById('chat-input').addEventListener('keydown', (e) => {
      if (e.key === 'Enter') { sendChat(); }
    });

    // 자동 메시지 (시간대별)
    setInterval(() => {
      const now = new Date();
      if (now.getHours() === 8 && now.getMinutes() === 0) {
        showSpeechBubbleInChunks('주인님, 일어날 시간이에요!');
      } else if (now.getHours() === 12 && now.getMinutes() === 0) {
        showSpeechBubbleInChunks('식사하실 시간이에요!');
      } else if (now.getHours() === 22 && now.getMinutes() === 0) {
        showSpeechBubbleInChunks('주무실 시간이에요 zzzz');
      }
    }, 60000);

    // 애니메이션 루프 (3D 씬 업데이트)
    function animate() {
      requestAnimationFrame(animate);
      updateBubblePosition();
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
      const moonPos = new THREE.Vector3(
        orbitCenter.x + Math.cos(angle + Math.PI) * radius,
        orbitCenter.y + Math.sin(angle + Math.PI) * radius,
        orbitCenter.z
      );
      moon.position.copy(moonPos);
      const t = now.getHours() + now.getMinutes()/60;
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
      // 단일 가로등(캐릭터 옆) 불빛: 아침/낮 꺼지고 밤에만 켜짐
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
          positions[i+1] -= 0.5;
          if (positions[i+1] < 0) { positions[i+1] = Math.random() * 50 + 20; }
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
      // 캐릭터 옆 단일 가로등 위치 업데이트 (캐릭터 기준 X offset 1)
      characterStreetlight.position.set(
        characterGroup.position.x + 1,
        -2,
        characterGroup.position.z
      );
      renderer.render(scene, camera);
    }
    animate();

    window.addEventListener('load', () => {
      showSpeechBubbleInChunks('환영합니다 개인 AI비서입니다. 무엇을 도와드릴까요');
      initCalendar();
    });

    /******************************
       4. 달력 UI (왼쪽 HUD)
         - 2020년부터 2070년까지 선택 가능
    ******************************/
    let currentYear, currentMonth;
    function initCalendar() {
      const now = new Date();
      currentYear = now.getFullYear();
      currentMonth = now.getMonth();
      populateYearSelect();
      renderCalendar(currentYear, currentMonth);
      document.getElementById('prev-month').addEventListener('click', () => {
        currentMonth--;
        if (currentMonth < 0) { currentMonth = 11; currentYear--; }
        renderCalendar(currentYear, currentMonth);
      });
      document.getElementById('next-month').addEventListener('click', () => {
        currentMonth++;
        if (currentMonth > 11) { currentMonth = 0; currentYear++; }
        renderCalendar(currentYear, currentMonth);
      });
      document.getElementById('year-select').addEventListener('change', (e) => {
        currentYear = parseInt(e.target.value);
        renderCalendar(currentYear, currentMonth);
      });
      // 오늘 일정 삭제 버튼
      document.getElementById('delete-today').addEventListener('click', () => {
        const now = new Date();
        const dateStr = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`;
        localStorage.removeItem("events_" + dateStr);
        const eventDiv = document.getElementById(`event-${dateStr}`);
        if (eventDiv) { eventDiv.textContent = ""; }
        alert(`오늘(${dateStr}) 일정이 삭제되었습니다.`);
      });
    }
    function populateYearSelect() {
      const yearSelect = document.getElementById('year-select');
      yearSelect.innerHTML = "";
      for (let y = 2020; y <= 2070; y++) {
        const option = document.createElement('option');
        option.value = y;
        option.textContent = y;
        if (y === currentYear) option.selected = true;
        yearSelect.appendChild(option);
      }
    }
    function renderCalendar(year, month) {
      const monthNames = ["1월","2월","3월","4월","5월","6월","7월","8월","9월","10월","11월","12월"];
      document.getElementById('month-year-label').textContent = `${year}년 ${monthNames[month]}`;
      const grid = document.getElementById('calendar-grid');
      grid.innerHTML = "";
      const daysOfWeek = ["일","월","화","수","목","금","토"];
      daysOfWeek.forEach(day => {
        const th = document.createElement("div");
        th.style.fontWeight = "bold";
        th.style.textAlign = "center";
        th.textContent = day;
        grid.appendChild(th);
      });
      const firstDay = new Date(year, month, 1).getDay();
      const daysInMonth = new Date(year, month + 1, 0).getDate();
      for (let i = 0; i < firstDay; i++){
        grid.appendChild(document.createElement("div"));
      }
      for (let d = 1; d <= daysInMonth; d++){
        const dateStr = `${year}-${month+1}-${d}`;
        const savedEvents = localStorage.getItem("events_" + dateStr) || "";
        const cell = document.createElement("div");
        cell.innerHTML = `<div class="day-number">${d}</div><div class="event" id="event-${dateStr}">${savedEvents}</div>`;
        cell.addEventListener("click", () => {
          const eventText = prompt(`${year}-${month+1}-${d} 일정 입력:`);
          if (eventText) { addEventToDay(dateStr, eventText); }
        });
        grid.appendChild(cell);
      }
    }
    function addEventToDay(dateStr, eventText) {
      let events = localStorage.getItem("events_" + dateStr);
      if (events) { events += "; " + eventText; }
      else { events = eventText; }
      localStorage.setItem("events_" + dateStr, events);
      const eventDiv = document.getElementById(`event-${dateStr}`);
      if (eventDiv) { eventDiv.textContent = events; }
      // 일정 추가 시 Wi‑Fi 전송 (예시)
      sendSchedule(`[${dateStr}] ${eventText}`);
    }
  </script>
</body>
</html>
