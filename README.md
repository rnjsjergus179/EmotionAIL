
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>3D 캐릭터 HUD, 달력 & 말풍선 채팅</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { height: 100%; font-family: Arial, sans-serif; overflow: hidden; }
    
    /* 오른쪽 채팅창 */
    #right-hud {
      position: fixed;
      top: 10%;
      right: 1%;
      width: 20%;
      padding: 1%;
      background: rgba(255,255,255,0.8);
      border-radius: 5px;
      box-shadow: 0 4px 8px rgba(0,0,0,0.2);
      z-index: 20;
    }
    #chat-log {
      display: none;
      height: 100px;
      overflow-y: scroll;
      border: 1px solid #ccc;
      padding: 5px;
      margin-top: 10px;
      border-radius: 3px;
      background: #fff;
    }
    #chat-input-area {
      display: flex;
      margin-top: 10px;
    }
    #chat-input {
      flex: 1;
      padding: 5px;
      font-size: 14px;
    }
    
    /* 왼쪽 캘린더 */
    #left-hud {
      position: fixed;
      top: 10%;
      left: 1%;
      width: 20%;
      padding: 1%;
      background: rgba(255,255,255,0.9);
      border-radius: 5px;
      box-shadow: 0 4px 8px rgba(0,0,0,0.2);
      z-index: 20;
      max-height: 80vh;
      overflow-y: auto;
    }
    #left-hud h3 { margin-bottom: 5px; }
    #calendar-container { margin-top: 10px; }
    #calendar-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 5px;
    }
    #calendar-header button { padding: 2px 6px; font-size: 12px; cursor: pointer; }
    #month-year-label { font-weight: bold; font-size: 14px; }
    #year-select { font-size: 12px; padding: 2px; margin-left: 5px; }
    #calendar-actions {
      margin-top: 5px;
      text-align: center;
    }
    #calendar-actions button {
      margin: 2px;
      padding: 5px 8px;
      font-size: 12px;
      cursor: pointer;
    }
    #calendar-grid {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      gap: 2px;
    }
    #calendar-grid div {
      background: #fff;
      border: 1px solid #ccc;
      border-radius: 4px;
      min-height: 25px;
      font-size: 10px;
      padding: 2px;
      position: relative;
      cursor: pointer;
    }
    #calendar-grid div:hover { background: #e9e9e9; }
    .day-number {
      position: absolute;
      top: 2px;
      left: 2px;
      font-weight: bold;
      font-size: 10px;
    }
    .event {
      margin-top: 14px;
      font-size: 8px;
      color: #333;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    
    /* Three.js 캔버스 */
    #canvas {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      z-index: 1;
      display: block;
    }
    
    /* 말풍선 */
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
      box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    }
    
    /* 튜토리얼 오버레이 */
    #tutorial-overlay {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0, 0, 0, 0.7);
      color: white;
      display: flex;
      justify-content: center;
      align-items: center;
      z-index: 100;
      opacity: 0;
      transition: opacity 1s ease-in-out;
      pointer-events: none;
    }
    #tutorial-content {
      text-align: center;
      padding: 20px;
      background: rgba(255, 255, 255, 0.1);
      border-radius: 10px;
      max-width: 600px;
    }
    #tutorial-content h2 { margin-bottom: 15px; }
    #tutorial-content p { margin: 10px 0; font-size: 14px; }
    
    /* 버전 선택 */
    #version-select {
      position: fixed;
      bottom: 10px;
      left: 10px;
      z-index: 50;
    }
    #version-select select {
      padding: 5px;
      font-size: 12px;
    }

    @media (max-width: 480px) {
      #right-hud, #left-hud { width: 90%; left: 5%; right: 5%; top: 5%; }
    }
  </style>
  
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js"></script>
  
  <script>
    document.addEventListener("contextmenu", event => event.preventDefault());
    let blockUntil = 0;
    let danceInterval; // 캐릭터 춤 애니메이션 제어 변수
    // currentCity: 날씨 API 호출 시 사용될 지역명 (초기값 "Seoul")
    let currentCity = "Seoul";
    
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
      
      // 지역 변경 명령 처리: "인천 바꿔줘", "서울로 바꿔줘" 등
      const regionPattern = /(서울|인천|파주|부산|대구|광주|대전|울산|단양)(?:로)?\s*바꿔줘/;
      if (regionPattern.test(lowerInput)) {
        const match = lowerInput.match(regionPattern);
        if(match && match[1]){
          currentCity = match[1];
          response = `지역이 ${match[1]}(으)로 변경되었습니다.`;
          showSpeechBubbleInChunks(response);
          inputEl.value = "";
          return;
        }
      }
      
      // 기존 "지역 [지역명]" 명령 처리
      if (lowerInput.startsWith("지역 ")) {
        const newCity = lowerInput.replace("지역", "").trim();
        if(newCity) {
          if(newCity === "수도권") {
            const selectedCity = prompt("수도권 지역 중 선택하세요: 인천, 서울, 파주");
            if(selectedCity) {
              currentCity = selectedCity;
              response = `지역이 ${selectedCity}(으)로 변경되었습니다.`;
            } else {
              response = "지역 변경이 취소되었습니다.";
            }
          }
          else if(newCity === "지방") {
            const selectedCity = prompt("지방 지역 중 선택하세요: 부산, 대구, 광주");
            if(selectedCity) {
              currentCity = selectedCity;
              response = `지역이 ${selectedCity}(으)로 변경되었습니다.`;
            } else {
              response = "지역 변경이 취소되었습니다.";
            }
          }
          else {
            currentCity = newCity;
            response = `지역이 ${newCity}(으)로 변경되었습니다.`;
          }
        } else {
          response = "변경할 지역을 입력해주세요.";
        }
      }
      else if (lowerInput.includes("시간") || lowerInput.includes("몇시") || lowerInput.includes("현재시간")) {
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
        response = "안녕하세요, 주인님! 오늘 기분은 어떠세요?";
        characterGroup.children[7].rotation.z = Math.PI / 4;
        setTimeout(() => { characterGroup.children[7].rotation.z = 0; }, 1000);
      }
      else if (lowerInput.includes("캐릭터 넌 누구야")) {
        response = "저는 당신의 개인 비서에요.";
      }
      else if (lowerInput.includes("일정")) {
        response = "캘린더는 왼쪽에서 확인하세요.";
      }
      else if (lowerInput.includes("캐릭터 춤춰줘")) {
        response = "춤출게요!";
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
      // 춤 관련 키워드 처리
      else if (
        lowerInput.includes("춤") ||
        lowerInput.includes("춤춰") ||
        lowerInput.includes("춤춰줘") ||
        lowerInput.includes("춤춰봐") ||
        lowerInput.includes("춤사위")
      ) {
        response = "춤추겠습니다! 잠시만 기다려 주세요.";
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
        response = "죄송해요, 잘 이해하지 못했어요. 다시 한 번 말씀해주시겠어요?";
      }
      
      showSpeechBubbleInChunks(response);
      inputEl.value = "";
    }
    
    // currentCity 기반 날씨 조회 (텍스트 명령용)
    async function getWeather() {
      try {
        const url = `https://api.openweathermap.org/data/2.5/weather?q=${currentCity}&appid=${weatherKey}&units=metric&lang=kr`;
        const res = await fetch(url);
        if (!res.ok) throw new Error("날씨 API 호출 실패");
        const data = await res.json();
        const description = data.weather[0].description;
        const temp = data.main.temp;
        currentWeather = description;
        let extraComment = "";
        if (description.indexOf("흐림") !== -1 || description.indexOf("구름") !== -1) {
          extraComment = " 오늘은 약간 흐린 날씨네요 ☁️";
        }
        return `오늘 ${currentCity}의 날씨는 ${description}이며, 온도는 ${temp}°C입니다.${extraComment}`;
      } catch (err) {
        currentWeather = "";
        return "날씨 정보를 가져오지 못했습니다.";
      }
    }
    
    // 위도/경도 기반 날씨 조회 (말풍선 출력)
    async function getWeatherByCoords(lat, lng) {
      try {
        const url = `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lng}&appid=${weatherKey}&units=metric&lang=kr`;
        const res = await fetch(url);
        if (!res.ok) throw new Error("날씨 API 호출 실패");
        const data = await res.json();
        const description = data.weather[0].description;
        const temp = data.main.temp;
        let extraComment = "";
        if (description.indexOf("흐림") !== -1 || description.indexOf("구름") !== -1) {
          extraComment = " 오늘은 약간 흐린 날씨네요 ☁️";
        }
        const msg = `해당 위치의 날씨는 ${description}, 온도는 ${temp}°C입니다.${extraComment}`;
        showSpeechBubbleInChunks(msg);
      } catch (err) {
        showSpeechBubbleInChunks("날씨 정보를 가져오지 못했습니다.");
      }
    }
    
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
      <p><strong>캐릭터:</strong> 채팅창에 "안녕", "캐릭터 춤춰줘" 등 입력해 보세요.</p>
      <p>
        <strong>채팅창:</strong> 오른쪽에서 "날씨 알려줘", "파일 저장해줘" 등 명령할 수 있습니다.<br>
        또한, "지역 [지역명]" (예: "지역 수도권" 또는 "지역 부산") 또는 "인천 바꿔줘", "서울로 바꿔줘" 등<br>
        명령어를 입력하면 해당 지역으로 변경되어 이후 "날씨 알려줘" 명령 시 업데이트된 지역의 날씨가 조회됩니다.
      </p>
      <p><strong>캘린더:</strong> 왼쪽에서 날짜 클릭해 일정을 추가하거나, 버튼으로 저장/삭제하세요.</p>
    </div>
  </div>
  
  <div id="version-select">
    <select onchange="changeVersion(this.value)">
      <option value="latest">최신 버전</option>
      <option value="1.3">구버전 1.3</option>
    </select>
  </div>
  
  <canvas id="canvas"></canvas>
</body>
</html>
