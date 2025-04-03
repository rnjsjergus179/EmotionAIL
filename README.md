
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>3D 캐릭터 HUD, 달력 & 말풍선 채팅</title>
  <style>
    /* 기존 스타일 유지 */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { height: 100%; font-family: Arial, sans-serif; overflow: hidden; }
    /* 나머지 스타일은 그대로 */
  </style>
  
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js"></script>
  
  <script>
    document.addEventListener("contextmenu", event => event.preventDefault());
    let blockUntil = 0;
    let danceInterval; // 춤 애니메이션 인터벌 변수 선언
    
    // 춤 관련 키워드 배열
    const danceKeywords = ["춤춰", "춤추게", "춤출수있게", "춤", "댄스", "dance"];
    
    // 인사말 응답 배열
    const greetingResponses = [
      "안녕하세요, 주인님! 오늘 기분은 어떠세요?",
      "안녕하세요! 좋은 하루 보내세요.",
      "반갑습니다, 주인님!",
      "안녕하세요! 오늘은 어떤 계획이 있으신가요?"
    ];
    
    // 기본(이해하지 못한 경우) 응답 배열
    const defaultResponses = [
      "죄송해요, 잘 이해하지 못했어요. 다시 한 번 말씀해주시겠어요?",
      "무슨 말씀이신지 잘 모르겠어요.",
      "다시 한 번 말씀해주시면 감사하겠습니다.",
      "저는 아직 배우는 중이라 잘 모르겠어요."
    ];

    document.addEventListener("copy", function(e) {
      e.preventDefault();
      let selectedText = window.getSelection().toString();
      selectedText = selectedText.replace(/396bfaf4974ab9c336b3fb46e15242da/g, "HIDDEN");
      e.clipboardData.setData("text/plain", selectedText);
      if (Date.now() < blockUntil) return;
      blockUntil = Date.now() + 3600000;
      showSpeechBubbleInChunks("1시간동안 차단됩니다.");
    });
    
    const weatherKey = "396bfaf4974ab9c336b3fb46e15242da";
    let currentWeather = "";
    
    function saveFile() {
      const content = "파일 저장 완료";
      const filename = "saved_file.txt";
      const blob = new Blob([content], { type: "text/plain;charset=utf-8" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = filename;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    }
    
    function saveCalendar() {
      const daysInMonth = new Date(currentYear, currentMonth+1, 0).getDate();
      const calendarData = {};
      for (let d = 1; d <= daysInMonth; d++){
        const eventDiv = document.getElementById(`event-${currentYear}-${currentMonth+1}-${d}`);
        if (eventDiv && eventDiv.textContent.trim() !== "") {
          calendarData[`${currentYear}-${currentMonth+1}-${d}`] = eventDiv.textContent;
        }
      }
      const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(calendarData, null, 2));
      const dlAnchorElem = document.createElement("a");
      dlAnchorElem.setAttribute("href", dataStr);
      dlAnchorElem.setAttribute("download", "calendar_events.json");
      dlAnchorElem.style.display = "none";
      document.body.appendChild(dlAnchorElem);
      dlAnchorElem.click();
      document.body.removeChild(dlAnchorElem);
    }
    
    async function sendChat() {
      const inputEl = document.getElementById("chat-input");
      const input = inputEl.value.trim();
      
      if (Date.now() < blockUntil) {
        showSpeechBubbleInChunks("1시간동안 차단됩니다.");
        inputEl.value = "";
        return;
      }
      
      if (!input) return;
      
      let response = "";
      const lowerInput = input.toLowerCase();
      
      if (lowerInput.includes("시간") || lowerInput.includes("몇시") || lowerInput.includes("현재시간")) {
        const now = new Date();
        const hours = now.getHours();
        const minutes = now.getMinutes();
        response = `현재 시간은 ${hours}시 ${minutes}분입니다.`;
      }
      else if (lowerInput.includes("파일 저장해줘")) {
        response = "네, 알겠습니다. 파일 저장하겠습니다.";
        saveFile();
      }
      else if ((lowerInput.includes("캘린더") && lowerInput.includes("저장")) ||
               lowerInput.includes("일정저장") ||
               lowerInput.includes("하루일과저장")) {
        response = "네, 알겠습니다. 캘린더 저장하겠습니다.";
        saveCalendar();
      }
      else if (lowerInput.includes("날씨") &&
         (lowerInput.includes("알려") || lowerInput.includes("어때") ||
          lowerInput.includes("뭐야") || lowerInput.includes("어떻게") || lowerInput.includes("맑아"))) {
        response = await getWeather();
      }
      else if (lowerInput.includes("기분") && lowerInput.includes("좋아")) {
        response = "정말요!? 저도 정말 기분좋아요😁";
        const originalEyeColor = leftEye.material.color.getHex();
        leftEye.material.color.set(0xffff00);
        rightEye.material.color.set(0xffff00);
        setTimeout(() => {
          leftEye.material.color.set(originalEyeColor);
          rightEye.material.color.set(originalEyeColor);
        }, 500);
        const originalLeftBrowRotation = leftBrow.rotation.x;
        const originalRightBrowRotation = rightBrow.rotation.x;
        const eyebrowInterval = setInterval(() => {
          const angle = Math.sin(Date.now() * 0.005) * 0.3;
          leftBrow.rotation.x = originalLeftBrowRotation + angle;
          rightBrow.rotation.x = originalRightBrowRotation + angle;
        }, 50);
        setTimeout(() => {
          clearInterval(eyebrowInterval);
          leftBrow.rotation.x = originalLeftBrowRotation;
          rightBrow.rotation.x = originalRightBrowRotation;
        }, 3000);
      }
      else if (lowerInput.includes("안녕")) {
        // 인사말에 대해 랜덤 응답 선택
        response = greetingResponses[Math.floor(Math.random() * greetingResponses.length)];
        characterGroup.children[7].rotation.z = Math.PI / 4;
        setTimeout(() => { characterGroup.children[7].rotation.z = 0; }, 1000);
      }
      else if (lowerInput.includes("캐릭터 넌 누구야")) {
        response = "저는 당신의 개인 비서에요.";
      }
      else if (lowerInput.includes("일정")) {
        response = "캘린더는 왼쪽에서 확인하세요.";
      }
      else if (danceKeywords.some(keyword => lowerInput.includes(keyword))) {
        // 춤 키워드 감지 시 춤 동작 실행
        response = "춤을 추겠습니다!";
        if (danceInterval) clearInterval(danceInterval);
        danceInterval = setInterval(() => {
          characterGroup.children[7].rotation.z = Math.sin(Date.now() * 0.01) * Math.PI / 4;
          head.rotation.y = Math.sin(Date.now() * 0.01) * Math.PI / 8;
        }, 50);
        setTimeout(() => {
          clearInterval(danceInterval);
          characterGroup.children[7].rotation.z = 0;
          head.rotation.y = 0;
        }, 3000);
      }
      else if (lowerInput.includes("하루일정 삭제해줘") || lowerInput.includes("일정 삭제")) {
        const dayStr = prompt("삭제할 하루일정의 날짜(일)를 입력하세요 (예: 15):");
        if (dayStr) {
          const dayNum = parseInt(dayStr);
          const eventDiv = document.getElementById(`event-${currentYear}-${currentMonth+1}-${dayNum}`);
          if (eventDiv) {
            eventDiv.textContent = "";
            response = `${currentYear}-${currentMonth+1}-${dayNum} 일정이 삭제되었습니다.`;
          } else {
            response = "해당 날짜의 셀이 없습니다. 현재 달에 있는 날짜를 입력해주세요.";
          }
        } else {
          response = "날짜를 입력하지 않았습니다.";
        }
      }
      else if (lowerInput.includes("입력하게 보여줘") || lowerInput.includes("일정 입력")) {
        const dayStr = prompt("일정을 입력할 날짜(일)를 입력하세요 (예: 15):");
        if (dayStr) {
          const dayNum = parseInt(dayStr);
          const eventDiv = document.getElementById(`event-${currentYear}-${currentMonth+1}-${dayNum}`);
          if (eventDiv) {
            const eventText = prompt(`${currentYear}-${currentMonth+1}-${dayNum} 일정 입력:`);
            if (eventText) {
              if (eventDiv.textContent) {
                eventDiv.textContent += "; " + eventText;
              } else {
                eventDiv.textContent = eventText;
              }
              response = `${currentYear}-${currentMonth+1}-${dayNum}에 일정이 추가되었습니다.`;
            } else {
              response = "일정을 입력하지 않았습니다.";
            }
          } else {
            response = "해당 날짜의 셀이 없습니다. 현재 달에 있는 날짜를 입력해주세요.";
          }
        } else {
          response = "날짜를 입력하지 않았습니다.";
        }
      }
      else {
        // 이해하지 못한 경우 랜덤 응답 선택
        response = defaultResponses[Math.floor(Math.random() * defaultResponses.length)];
      }
      
      showSpeechBubbleInChunks(response);
      inputEl.value = "";
    }
    
    async function getWeather() {
      try {
        const city = "Seoul";
        const url = `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${weatherKey}&units=metric&lang=kr`;
        const res = await fetch(url);
        if (!res.ok) throw new Error("날씨 API 호출 실패");
        const data = await res.json();
        const description = data.weather[0].description;
        const temp = data.main.temp;
        currentWeather = description;
        return `오늘 ${city}의 날씨는 ${description}이며, 온도는 ${temp}°C입니다.`;
      } catch (err) {
        currentWeather = "";
        return "날씨 정보를 가져오지 못했습니다.";
      }
    }
    
    // 나머지 함수는 변경 없음
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
    
    function updateLightning() {
      if (currentWeather.indexOf("번개") !== -1 || currentWeather.indexOf("뇌우") !== -1) {
        if (Math.random() < 0.001) {
          lightningLight.intensity = 5;
          setTimeout(() => { lightningLight.intensity = 0; }, 100);
        }
      }
    }
    
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
    
    window.addEventListener("DOMContentLoaded", function() {
      document.getElementById("chat-input").addEventListener("keydown", function(e) {
        if (e.key === "Enter") sendChat();
      });
    });
    
    window.addEventListener("resize", function(){
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  </script>
</head>
<body>
  <!-- 기존 HTML 구조 유지 -->
  <div id="right-hud">
    <h3>채팅창</h3>
    <div id="chat-log"></div>
    <div id="chat-input-area">
      <input type="text" id="chat-input" placeholder="채팅 입력..." />
    </div>
  </div>
  
  <div id="left-hud">
    <h3>캘린더</h3>
    <div id="calendar-container">
      <div id="calendar-header">
        <button id="prev-month">◀</button>
        <span id="month-year-label"></span>
        <button id="next-month">▶</button>
        <select id="year-select"></select>
      </div>
      <div id="calendar-actions">
        <button id="delete-day-event">하루일정 삭제</button>
        <button id="save-calendar">바탕화면 저장</button>
      </div>
      <div id="calendar-grid"></div>
    </div>
  </div>
  
  <div id="speech-bubble"></div>
  
  <div id="tutorial-overlay">
    <div id="tutorial-content">
      <h2>사용법 안내</h2>
      <p><strong>캐릭터:</strong> 채팅창에 "안녕", "캐릭터 춤춰줘" 등을 입력해 보세요.</p>
      <p><strong>채팅창:</strong> 오른쪽에서 "날씨 알려줘", "파일 저장해줘" 등으로 명령할 수 있습니다.</p>
      <p><strong>캘린더:</strong> 왼쪽에서 날짜를 클릭해 일정을 추가하거나, 버튼으로 저장/삭제하세요.</p>
    </div>
  </div>
  
  <div id="version-select">
    <select onchange="changeVersion(this.value)">
      <option value="latest">최신 버전</option>
      <option value="1.3">구버전 1.3</option>
    </select>
  </div>
  
  <canvas id="canvas"></canvas>
  
  <script>
    // Three.js 및 애니메이션 관련 코드 (변경 없음)
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById("canvas"), alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.position.set(5, 5, 10);
    camera.lookAt(0, 0, 0);
    
    // 나머지 Three.js 초기화 및 애니메이션 코드 유지
    /* 생략 */
    
    function animate() {
      requestAnimationFrame(animate);
      /* 기존 animate 함수 내용 유지 */
      renderer.render(scene, camera);
    }
    animate();
    
    let currentYear, currentMonth;
    function initCalendar() {
      /* 기존 캘린더 초기화 코드 유지 */
    }
    /* 나머지 캘린더 관련 함수 유지 */
    
    window.addEventListener("load", () => {
      initCalendar();
      showTutorial();
    });
    
    /* 나머지 함수 유지 */
  </script>
</body>
</html
