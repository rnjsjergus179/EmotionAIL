
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>3D 캐릭터 HUD, 캘린더, 음성 채팅 & 말풍선</title>
  <style>
    /* 기본 스타일 */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { height: 100%; font-family: 'Courier New', monospace; overflow: hidden; }
    
    /* 오른쪽 채팅창 HUD */
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
    #region-select {
      width: 100%;
      padding: 5px;
      font-size: 14px;
      margin-bottom: 10px;
    }
    /* 채팅 로그 (필요시 보이도록 설정) */
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
    /* 채팅 입력창 */
    #chat-input {
      flex: 1;
      padding: 5px;
      font-size: 14px;
    }
    
    /* HUD-6: 음성 입력 영역 */
    #hud-6 {
      position: fixed;
      top: 45%;
      right: 1%;
      width: 20%;
      padding: 5px;
      background: rgba(255,255,255,0.95);
      border-radius: 5px;
      box-shadow: 0 4px 8px rgba(0,0,0,0.2);
      z-index: 25;
      text-align: center;
    }
    #hud-6 button {
      padding: 8px 12px;
      font-size: 14px;
      border: none;
      border-radius: 4px;
      background: #00ffcc;
      color: #000;
      cursor: pointer;
      transition: background 0.3s;
    }
    #hud-6 button:hover {
      background: #00cc99;
    }
    
    /* 왼쪽 캘린더 HUD */
    #left-hud {
      position: fixed;
      top: 10%;
      left: 1%;
      width: 20%;
      padding: 1%;
      background: rgba(0, 0, 0, 0.7);
      border: 2px solid #00ffcc;
      border-radius: 10px;
      box-shadow: 0 0 15px rgba(0,255,204,0.5);
      z-index: 20;
      max-height: 80vh;
      overflow-y: auto;
      color: #00ffcc;
    }
    #left-hud h3 { 
      margin-bottom: 5px; 
      text-shadow: 0 0 5px #00ffcc;
    }
    #calendar-container { margin-top: 10px; }
    #calendar-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 5px;
    }
    #calendar-header button { 
      padding: 2px 6px; 
      font-size: 12px; 
      cursor: pointer; 
      background: #00ffcc; 
      color: #000; 
      border: none; 
      border-radius: 3px; 
      box-shadow: 0 0 5px #00ffcc; 
      transition: all 0.3s; 
    }
    #calendar-header button:hover { 
      background: #00cc99; 
      box-shadow: 0 0 10px #00ffcc; 
    }
    #month-year-label { 
      font-weight: bold; 
      font-size: 14px; 
      text-shadow: 0 0 5px #00ffcc;
    }
    #year-select { 
      font-size: 12px; 
      padding: 2px; 
      margin-left: 5px; 
      background: #333; 
      color: #00ffcc; 
      border: 1px solid #00ffcc; 
      border-radius: 3px; 
    }
    #calendar-actions {
      margin-top: 5px;
      text-align: center;
    }
    #calendar-actions button {
      margin: 2px;
      padding: 5px 8px;
      font-size: 12px;
      cursor: pointer;
      background: #00ffcc;
      color: #000;
      border: none;
      border-radius: 3px;
      box-shadow: 0 0 5px #00ffcc;
      transition: all 0.3s;
    }
    #calendar-actions button:hover {
      background: #00cc99;
      box-shadow: 0 0 10px #00ffcc;
    }
    #calendar-grid {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      gap: 2px;
    }
    #calendar-grid div {
      background: rgba(255,255,255,0.1);
      border: 1px solid #00ffcc;
      border-radius: 4px;
      min-height: 25px;
      font-size: 10px;
      padding: 2px;
      position: relative;
      cursor: pointer;
      transition: all 0.3s;
    }
    #calendar-grid div:hover { 
      background: rgba(0,255,204,0.3);
      box-shadow: 0 0 5px #00ffcc;
    }
    .day-number {
      position: absolute;
      top: 2px;
      left: 2px;
      font-weight: bold;
      font-size: 10px;
      color: #00ffcc;
      text-shadow: 0 0 3px #00ffcc;
    }
    .event {
      margin-top: 14px;
      font-size: 8px;
      color: #00ffcc;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      text-shadow: 0 0 3px #00ffcc;
    }
    
    /* 메인 캔버스와 말풍선 */
    #canvas {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      z-index: 1;
      display: block;
    }
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
    
    /* 지도 또는 유튜브가 표시되는 영역 */
    #hud-3 {
      position: fixed;
      top: 70%;
      right: 1%;
      width: 20%;
      height: 20%;
      padding: 1%;
      background: rgba(255,255,255,0.9);
      border-radius: 5px;
      box-shadow: 0 4px 8px rgba(0,0,0,0.2);
      z-index: 20;
      overflow: hidden;
    }
    
    /* 버전 선택 메뉴 */
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
      #right-hud, #left-hud, #hud-3, #hud-6 { width: 90%; left: 5%; right: 5%; top: 5%; }
    }
  </style>
  
  <!-- Three.js 라이브러리 -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js"></script>
  
  <script>
    /* 전역 키워드 객체 – 자동완성 및 채팅 처리용 */
    const KEYWORDS = {
      greetings: ["안녕", "안녕하세요", "안녕 하세", "안녕하시오", "안녕한갑네"],
      sleep: ["잘자", "좋은꿈", "좋은 꿈", "잘자요", "잘자시게", "잘자리요", "잘자라니께"],
      youtube: ["유튜브", "유트브", "유튜브알려줘", "유튭", "유튜브랑", "유튜브나와줘"],
      twitter: ["트위터", "트위터 보여주게", "트위터 틔위터검색", "트위터보여", "트위터보여줘봐"],
      naver: ["네이버", "네이버 보여줘", "네이버 보여주게", "네이버 검색"],
      weather: ["날씨알려줘", "날씨알려주게", "날씨좀알려줘", "날씨 알려줘", "날씨 좀 알려줘", "날씨 어때", "날씨 맑아"],
      calendar: ["일정 알려줘"],
      time: ["시간 알려줘"],
      map: ["지도 보여줘", "교통정보"]
    };
    
    /* 전역 변수 */
    document.addEventListener("contextmenu", event => event.preventDefault());
    let blockUntil = 0;
    let currentCity = "서울";
    let currentWeather = "";
    const weatherKey = "2caa7fa4a66f2f8d150f1da93d306261";
    const regionMap = {
      "서울": "Seoul",
      "인천": "Incheon",
      "수원": "Suwon",
      "고양": "Goyang",
      "성남": "Seongnam",
      "용인": "Yongin",
      "부천": "Bucheon",
      "안양": "Anyang",
      "의정부": "Uijeongbu",
      "광명": "Gwangmyeong",
      "안산": "Ansan",
      "파주": "Paju",
      "부산": "Busan",
      "대구": "Daegu",
      "광주": "Gwangju",
      "대전": "Daejeon",
      "울산": "Ulsan",
      "제주": "Jeju",
      "전주": "Jeonju",
      "청주": "Cheongju",
      "포항": "Pohang",
      "여수": "Yeosu",
      "김해": "Gimhae"
    };
    const regionList = Object.keys(regionMap);
    
    document.addEventListener("copy", function(e) {
      e.preventDefault();
      let selectedText = window.getSelection().toString();
      selectedText = selectedText.replace(/2caa7fa4a66f2f8d150f1da93d306261/g, "HIDDEN");
      e.clipboardData.setData("text/plain", selectedText);
      if (Date.now() < blockUntil) return;
      blockUntil = Date.now() + 3600000;
      showSpeechBubbleInChunks("1시간동안 차단됩니다.");
    });
    
    /* 캘린더, 파일 저장 관련 함수들 */
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
      for (let d = 1; d <= daysInMonth; d++) {
        const eventDiv = document.getElementById(`event-${currentYear}-${currentMonth+1}-${d}`);
        if (eventDiv && eventDiv.textContent.trim() !== "") {
          calendarData[`${currentYear}-${currentMonth+1}-${d}`] = eventDiv.textContent;
        }
      }
      localStorage.setItem("calendarEvents", JSON.stringify(calendarData));
      const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(calendarData, null, 2));
      const dlAnchorElem = document.createElement("a");
      dlAnchorElem.setAttribute("href", dataStr);
      dlAnchorElem.setAttribute("download", "calendar_events.json");
      dlAnchorElem.style.display = "none";
      document.body.appendChild(dlAnchorElem);
      dlAnchorElem.click();
      document.body.removeChild(dlAnchorElem);
    }
    
    function deleteCalendarEvent(day) {
      const eventDiv = document.getElementById(`event-${currentYear}-${currentMonth+1}-${day}`);
      if (eventDiv) {
        eventDiv.textContent = "";
        const calendarData = JSON.parse(localStorage.getItem("calendarEvents") || "{}");
        delete calendarData[`${currentYear}-${currentMonth+1}-${day}`];
        localStorage.setItem("calendarEvents", JSON.stringify(calendarData));
        return `${currentYear}-${currentMonth+1}-${day} 일정이 삭제되었습니다.`;
      } else {
        return "해당 날짜에 일정이 없습니다.";
      }
    }
    
    function getCalendarEvents(dateStr = null) {
      const calendarData = JSON.parse(localStorage.getItem("calendarEvents") || "{}");
      if (!Object.keys(calendarData).length) {
        return "저장된 일정이 없습니다. 먼저 캘린더를 저장해주세요.";
      }
      
      if (dateStr) {
        if (calendarData[dateStr]) {
          return `${dateStr}의 일정: ${calendarData[dateStr]}`;
        } else {
          return `${dateStr}에는 일정이 없습니다.`;
        }
      } else {
        const currentMonthStr = `${currentYear}-${currentMonth+1}`;
        let events = [];
        for (let key in calendarData) {
          if (key.startsWith(currentMonthStr)) {
            events.push(`${key}: ${calendarData[key]}`);
          }
        }
        if (events.length) {
          return `현재 월(${currentMonthStr})의 일정:\n${events.join("\n")}`;
        } else {
          return `현재 월(${currentMonthStr})에는 일정이 없습니다.`;
        }
      }
    }
    
    function updateMap() {
      const englishCity = regionMap[currentCity] || "Seoul";
      document.getElementById("map-iframe").src = `https://www.google.com/maps?q=${encodeURIComponent(englishCity)}&output=embed`;
    }
    
    /* 날씨 API 호출 – 지도와 동일하게 영어 도시명을 사용 */
    async function getWeather() {
      try {
        const englishCity = regionMap[currentCity] || "Seoul";
        const url = `https://api.openweathermap.org/data/2.5/weather?q=${encodeURIComponent(englishCity)}&appid=${weatherKey}&units=metric&lang=kr`;
        const res = await fetch(url);
        if (!res.ok) throw new Error("날씨 API 호출 실패");
        const data = await res.json();
        currentWeather = data.weather[0].description;
        const message = `오늘 ${currentCity}의 날씨는 ${data.weather[0].description}이고, 기온은 ${data.main.temp}°C입니다.`;
        return { message };
      } catch (error) {
        console.error(error);
        currentWeather = "";
        return { message: "날씨 정보를 가져오는데 실패했습니다." };
      }
    }
    
    /* 날씨 효과 업데이트 – 비 효과와 구름 효과를 각각 독립적으로 표시 */
    function updateWeatherEffects() {
      if (!currentWeather) return;
      // 비 효과: "비" 또는 "소나기"가 포함되면
      if (currentWeather.includes("비") || currentWeather.includes("소나기")) {
        rainGroup.visible = true;
        cloudRainGroup.visible = true;
      } else {
        rainGroup.visible = false;
        cloudRainGroup.visible = false;
      }
      // 구름 효과: "구름" 또는 "흐림"이 포함되면
      if (currentWeather.includes("구름") || currentWeather.includes("흐림")) {
        houseCloudGroup.visible = true;
      } else {
        houseCloudGroup.visible = false;
      }
    }
    
    function updateLightning() {
      if (currentWeather.includes("번개") || currentWeather.includes("뇌우")) {
        if (Math.random() < 0.001) {
          lightningLight.intensity = 5;
          setTimeout(() => { lightningLight.intensity = 0; }, 100);
        }
      }
    }
    
    async function updateWeatherAndEffects(sendMessage = true) {
      const weatherData = await getWeather();
      if (sendMessage) {
        showSpeechBubbleInChunks(weatherData.message);
      }
      updateWeatherEffects();
    }
    
    function changeRegion(value) {
      currentCity = value;
      updateMap();
      updateWeatherAndEffects();
      showSpeechBubbleInChunks(`지역이 ${value}(으)로 변경되었습니다.`);
    }
    
    function startSpeechRecognition() {
      if (!('webkitSpeechRecognition' in window)) {
        alert("이 브라우저는 음성 인식을 지원하지 않습니다.");
        return;
      }
      const recognition = new webkitSpeechRecognition();
      recognition.lang = "ko-KR";
      recognition.interimResults = false;
      recognition.maxAlternatives = 1;
      recognition.start();
      recognition.onresult = function(event) {
        const transcript = event.results[0][0].transcript.trim();
        document.getElementById("chat-input").value = transcript;
      };
      recognition.onerror = function(event) {
        console.error("음성 인식 오류:", event.error);
      };
    }
    
    /* 채팅 처리 – KEYWORDS 객체를 이용해 중복 키워드를 통합 */
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
      
      // 지역 변경 처리
      if (lowerInput.startsWith("지역 ")) {
        const newCity = lowerInput.replace("지역", "").trim();
        if (newCity) {
          if (regionList.includes(newCity)) {
            currentCity = newCity;
            document.getElementById("region-select").value = newCity;
            response = `좋아요, 지역을 ${newCity}(으)로 변경할게요!`;
            updateMap();
            await updateWeatherAndEffects();
          } else {
            response = "죄송해요, 그 지역은 지원하지 않아요. 드롭다운 메뉴에서 선택해주세요.";
          }
        } else {
          response = "변경할 지역을 입력해 주세요.";
        }
      } else if (regionList.includes(input)) {
        currentCity = input;
        document.getElementById("region-select").value = input;
        response = `좋아요, 지역을 ${input}(으)로 변경할게요!`;
        updateMap();
        await updateWeatherAndEffects();
      }
      
      // 각 키워드 그룹별 처리 (KEYWORDS 사용)
      if (!response && KEYWORDS.youtube.some(keyword => lowerInput.includes(keyword))) {
        response = "유튜브를 보여드릴게요! 잠시만 기다려 주세요.";
        showSpeechBubbleInChunks(response);
        setTimeout(() => { window.location.href = "https://www.youtube.com/"; }, 2000);
        inputEl.value = "";
        return;
      }
      if (!response && KEYWORDS.twitter.some(keyword => lowerInput.includes(keyword))) {
        response = "트위터(현재 X)를 보여드릴게요! 잠시만 기다려 주세요.";
        showSpeechBubbleInChunks(response);
        setTimeout(() => { window.location.href = "https://x.com/login?lang=ko"; }, 2000);
        inputEl.value = "";
        return;
      }
      if (!response && KEYWORDS.naver.some(keyword => lowerInput.includes(keyword))) {
        response = "네이버를 보여드릴게요! 잠시만 기다려 주세요.";
        showSpeechBubbleInChunks(response);
        setTimeout(() => { window.location.href = "https://m.naver.com/"; }, 2000);
        inputEl.value = "";
        return;
      }
      if (!response && KEYWORDS.greetings.some(keyword => lowerInput.includes(keyword))) {
        response = "안녕하세요! 만나서 반갑습니다. 오늘 하루 어떠셨나요?";
      }
      if (!response && KEYWORDS.sleep.some(keyword => lowerInput.includes(keyword))) {
        response = "편안한 밤 되세요, 좋은 꿈 꾸세요~";
      }
      if (!response && KEYWORDS.weather.some(keyword => lowerInput.includes(keyword))) {
        await updateWeatherAndEffects();
        inputEl.value = "";
        return;
      }
      if (!response && lowerInput.includes("일정") && lowerInput.includes("알려줘")) {
        const dateMatch = input.match(/\d{4}-\d{1,2}-\d{1,2}/);
        if (dateMatch) {
          const dateStr = dateMatch[0];
          response = getCalendarEvents(dateStr);
        } else {
          response = getCalendarEvents();
        }
      }
      if (!response && KEYWORDS.time.some(keyword => lowerInput.includes(keyword))) {
        const now = new Date();
        const hours = now.getHours();
        const minutes = now.getMinutes();
        response = `현재 시간은 ${hours}시 ${minutes}분입니다.`;
      }
      if (!response && KEYWORDS.map.some(keyword => lowerInput.includes(keyword))) {
        response = "지도를 보여드릴게요!";
        showSpeechBubbleInChunks(response);
        setTimeout(() => { window.location.href = "https://www.google.com/maps"; }, 2000);
        inputEl.value = "";
        return;
      }
      
      // 감정 및 일반 대화 응답 처리
      if (!response) {
        if (lowerInput.includes("기분") || lowerInput.includes("슬프") || lowerInput.includes("우울") ||
            lowerInput.includes("짜증") || lowerInput.includes("화난") || lowerInput.includes("분노") ||
            lowerInput.includes("놀람") || lowerInput.includes("피곤")) {
          const sadResponses = [
            "정말 마음이 아프시네요. 제가 도와드릴 수 있다면 좋겠어요.",
            "그런 날도 있죠. 힘내시고 천천히 쉬어가세요.",
            "마음이 많이 힘들어 보이네요. 꼭 회복되시길 바랄게요."
          ];
          const happyResponses = [
            "오늘 정말 즐거워 보이세요! 기분 좋은 일이 가득하길 바랍니다.",
            "당신의 미소가 전해지네요. 행복한 하루 보내세요!",
            "기쁨이 넘치는 하루, 함께 기뻐요!"
          ];
          const angryResponses = [
            "화가 치밀어 오시네요. 잠시 심호흡을 해보세요.",
            "분노를 조금 내려놓고, 잠깐의 휴식 어떠세요?",
            "감정이 많이 격해지신 것 같아요. 차분해지시길 바랄게요."
          ];
          const surprisedResponses = [
            "오, 놀라운 소식이네요! 자세히 말씀해 주세요.",
            "정말 놀라워요! 당신의 이야기에 귀 기울이고 있어요."
          ];
          const tiredResponses = [
            "피곤해 보이시네요. 푹 쉬시고 내일 더 힘내세요.",
            "오늘 하루 수고 많으셨어요. 편안한 밤 되세요."
          ];
          if (lowerInput.includes("슬프") || lowerInput.includes("우울")) {
            response = sadResponses[Math.floor(Math.random() * sadResponses.length)];
          } else if (lowerInput.includes("기쁘") || lowerInput.includes("행복")) {
            response = happyResponses[Math.floor(Math.random() * happyResponses.length)];
          } else if (lowerInput.includes("화난") || lowerInput.includes("분노") || lowerInput.includes("짜증")) {
            response = angryResponses[Math.floor(Math.random() * angryResponses.length)];
          } else if (lowerInput.includes("놀람")) {
            response = surprisedResponses[Math.floor(Math.random() * surprisedResponses.length)];
          } else if (lowerInput.includes("피곤")) {
            response = tiredResponses[Math.floor(Math.random() * tiredResponses.length)];
          } else {
            const neutralResponses = [
              "그렇군요, 좀 더 자세히 말씀해 주실 수 있을까요?",
              "알겠습니다. 추가로 궁금한 점이 있으신가요?",
              "흥미로운 이야기네요. 더 이야기해 주세요!"
            ];
            response = neutralResponses[Math.floor(Math.random() * neutralResponses.length)];
          }
        } else {
          const generalResponses = [
            "정말 흥미로운 이야기네요. 더 들려주세요!",
            "알겠습니다. 혹시 다른 궁금한 점은 없으신가요?",
            "그렇군요. 당신의 의견을 듣고 있으니 저도 많이 배워요.",
            "그렇게 느끼실 수 있겠네요. 함께 이야기 나눠봐요!"
          ];
          response = generalResponses[Math.floor(Math.random() * generalResponses.length)];
        }
      }
      
      showSpeechBubbleInChunks(response);
      inputEl.value = "";
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
      // 자동 완성용 datalist – KEYWORDS의 모든 값을 결합
      const chatInput = document.getElementById("chat-input");
      chatInput.setAttribute("list", "chat-keywords");
      const autoCompleteList = document.createElement("datalist");
      autoCompleteList.id = "chat-keywords";
      const allKeywords = Object.values(KEYWORDS).flat();
      allKeywords.forEach(kw => {
        const option = document.createElement("option");
        option.value = kw;
        autoCompleteList.appendChild(option);
      });
      document.body.appendChild(autoCompleteList);
      
      document.getElementById("chat-input").addEventListener("keydown", function(e) {
        if (e.key === "Enter") sendChat();
      });
      
      const regionSelect = document.getElementById("region-select");
      regionList.forEach(region => {
        const option = document.createElement("option");
        option.value = region;
        option.textContent = `${region} (${regionMap[region]})`;
        if (region === currentCity) option.selected = true;
        regionSelect.appendChild(option);
      });
    });
    
    window.addEventListener("resize", function(){
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
    
    window.addEventListener("load", async () => {
      initCalendar();
      updateMap();
      await updateWeatherAndEffects();
    });
    
    function changeVersion(version) {
      if (version === "1.3") {
        window.location.href = "https://aipersonalassistant.neocities.org/";
      } else if (version === "latest") {
        window.location.reload();
      }
    }
  </script>
</head>
<body>
  <div id="right-hud">
    <h3>채팅창</h3>
    <select id="region-select" onchange="changeRegion(this.value)">
      <option value="" disabled>지역 선택</option>
    </select>
    <div id="chat-log"></div>
    <div id="chat-input-area">
      <input type="text" id="chat-input" placeholder="채팅 입력..." />
    </div>
  </div>
  
  <!-- HUD-6: 음성 입력 영역 -->
  <div id="hud-6">
    <button onclick="startSpeechRecognition()">🎤 음성 입력</button>
  </div>
  
  <div id="hud-3">
    <iframe id="map-iframe" src="https://www.google.com/maps?q=Seoul&output=embed" frameborder="0" style="width:100%; height:100%; border:0;" allowfullscreen></iframe>
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
  
  <div id="version-select">
    <select onchange="changeVersion(this.value)">
      <option value="latest">최신 버전 (1.7)</option>
      <option value="1.3">구버전 1.3</option>
    </select>
  </div>
  
  <canvas id="canvas"></canvas>
  
  <script>
    /* Three.js Scene, 카메라, 렌더러 및 애니메이션 설정 */
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ canvas: document.getElementById("canvas"), alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.position.set(5, 5, 10);
    camera.lookAt(0, 0, 0);
    
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
    directionalLight.position.set(5, 10, 7).normalize();
    scene.add(directionalLight);
    scene.add(new THREE.AmbientLight(0x333333));
    
    const sunMaterial = new THREE.MeshStandardMaterial({ color: 0xffcc00, emissive: 0xff9900, transparent: true, opacity: 0 });
    const sun = new THREE.Mesh(new THREE.SphereGeometry(1.5, 64, 64), sunMaterial);
    scene.add(sun);
    
    const moonMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc, emissive: 0x222222, transparent: true, opacity: 1 });
    const moon = new THREE.Mesh(new THREE.SphereGeometry(1.2, 64, 64), moonMaterial);
    scene.add(moon);
    
    const stars = [], fireflies = [];
    for (let i = 0; i < 200; i++) {
      const star = new THREE.Mesh(new THREE.SphereGeometry(0.03, 8, 8), new THREE.MeshBasicMaterial({ color: 0xffffff }));
      star.position.set((Math.random()-0.5)*100, (Math.random()-0.5)*60, -20);
      scene.add(star);
      stars.push(star);
    }
    for (let i = 0; i < 60; i++) {
      const firefly = new THREE.Mesh(new THREE.SphereGeometry(0.05, 8, 8), new THREE.MeshBasicMaterial({ color: 0xffff99 }));
      firefly.position.set((Math.random()-0.5)*40, (Math.random()-0.5)*20, -10);
      scene.add(firefly);
      fireflies.push(firefly);
    }
    
    const floorGeometry = new THREE.PlaneGeometry(400, 400, 128, 128);
    const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x808080, roughness: 1, metalness: 0 });
    const floor = new THREE.Mesh(floorGeometry, floorMaterial);
    floor.rotation.x = -Math.PI/2;
    floor.position.y = -2;
    scene.add(floor);
    
    const backgroundGroup = new THREE.Group();
    scene.add(backgroundGroup);
    function createBuilding(width, height, depth, color) {
      const buildingGroup = new THREE.Group();
      const geometry = new THREE.BoxGeometry(width, height, depth);
      const material = new THREE.MeshStandardMaterial({ color: color, roughness: 0.7, metalness: 0.1 });
      const building = new THREE.Mesh(geometry, material);
      buildingGroup.add(building);
      
      const windowMat = new THREE.MeshStandardMaterial({ color: 0x87CEEB });
      for (let y = 3; y < height - 1; y += 2) {
        for (let x = -width/2 + 0.5; x < width/2; x += 1) {
          const window = new THREE.Mesh(new THREE.BoxGeometry(0.4, 0.8, 0.1), windowMat);
          window.position.set(x, y - height/2, depth/2 + 0.01);
          buildingGroup.add(window);
        }
      }
      const doorMat = new THREE.MeshStandardMaterial({ color: 0x8B4513 });
      const door = new THREE.Mesh(new THREE.BoxGeometry(1, 2, 0.1), doorMat);
      door.position.set(0, -height/2 + 1, depth/2 + 0.01);
      buildingGroup.add(door);
      
      return buildingGroup;
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
      
      const windowMat = new THREE.MeshStandardMaterial({ color: 0xFFFFE0 });
      const window1 = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.1), windowMat);
      window1.position.set(-width/4, -2 + height/2, depth/2 + 0.01);
      const window2 = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.1), windowMat);
      window2.position.set(width/4, -2 + height/2, depth/2 + 0.01);
      houseGroup.add(window1, window2);
      
      const doorMat = new THREE.MeshStandardMaterial({ color: 0x8B4513 });
      const door = new THREE.Mesh(new THREE.BoxGeometry(1, 1.5, 0.1), doorMat);
      door.position.set(0, -2 + height/4, depth/2 + 0.01);
      houseGroup.add(door);
      
      return houseGroup;
    }
    for (let i = 0; i < 20; i++) {
      const width = Math.random() * 4 + 4;
      const height = Math.random() * 20 + 20;
      const depth = Math.random() * 4 + 4;
      const building = createBuilding(width, height, depth, 0x555555);
      const col = i % 10;
      const row = Math.floor(i / 10);
      const x = -50 + col * 10;
      const z = -30 - row * 20;
      building.position.set(x, -2 + height/2, z);
      backgroundGroup.add(building);
    }
    for (let i = 0; i < 10; i++) {
      const width = Math.random() * 4 + 6;
      const height = Math.random() * 4 + 6;
      const depth = Math.random() * 4 + 6;
      const house = createHouse(width, height, depth, 0xa0522d, 0x8b0000);
      const x = -40 + i * 10;
      const z = -10;
      house.position.set(x, 0, z);
      backgroundGroup.add(house);
    }
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
    
    let rainGroup = new THREE.Group();
    scene.add(rainGroup);
    function initRain() {
      const rainCount = 2000;
      const rainGeometry = new THREE.BufferGeometry();
      const positions = new Float32Array(rainCount * 3);
      for (let i = 0; i < rainCount; i++) {
        positions[i * 3] = Math.random() * 200 - 100;
        positions[i * 3 + 1] = Math.random() * 100;
        positions[i * 3 + 2] = Math.random() * 200 - 100;
      }
      rainGeometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
      const rainMaterial = new THREE.PointsMaterial({ color: 0xaaaaee, size: 0.1, transparent: true, opacity: 0.6 });
      const rainParticles = new THREE.Points(rainGeometry, rainMaterial);
      rainGroup.add(rainParticles);
    }
    initRain();
    rainGroup.visible = false;
    
    let houseCloudGroup = new THREE.Group();
    scene.add(houseCloudGroup);
    function createHouseCloud() {
      const cloud = new THREE.Group();
      const cloudMat = new THREE.MeshLambertMaterial({ color: 0xffffff, transparent: true, opacity: 0.9 });
      const sphere1 = new THREE.Mesh(new THREE.SphereGeometry(0.5, 32, 32), cloudMat);
      sphere1.position.set(0, 0, 0);
      const sphere2 = new THREE.Mesh(new THREE.SphereGeometry(0.4, 32, 32), cloudMat);
      sphere2.position.set(0.6, 0.2, 0);
      const sphere3 = new THREE.Mesh(new THREE.SphereGeometry(0.5, 32, 32), cloudMat);
      sphere3.position.set(-0.6, 0.1, 0);
      cloud.add(sphere1, sphere2, sphere3);
      cloud.scale.set(2, 2, 2);
      cloud.userData.initialPos = cloud.position.clone();
      return cloud;
    }
    const singleCloud = createHouseCloud();
    houseCloudGroup.add(singleCloud);
    houseCloudGroup.position.set(0, 2, 0);
    
    let cloudRainGroup = new THREE.Group();
    function initCloudRain() {
      const cloudRainCount = 100;
      const geometry = new THREE.BufferGeometry();
      const positions = new Float32Array(cloudRainCount * 3);
      for (let i = 0; i < cloudRainCount; i++) {
        positions[i * 3] = (Math.random()-0.5) * 1.5;
        positions[i * 3 + 1] = Math.random() * 0.2;
        positions[i * 3 + 2] = (Math.random()-0.5) * 1.5;
      }
      geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
      const material = new THREE.PointsMaterial({ color: 0xaaaaee, size: 0.05, transparent: true, opacity: 0.8 });
      const particles = new THREE.Points(geometry, material);
      cloudRainGroup.add(particles);
    }
    initCloudRain();
    cloudRainGroup.visible = false;
    houseCloudGroup.add(cloudRainGroup);
    
    function updateHouseClouds() {
      const headWorldPos = new THREE.Vector3();
      head.getWorldPosition(headWorldPos);
      houseCloudGroup.position.x = headWorldPos.x + Math.sin(Date.now() * 0.001) * 1;
      houseCloudGroup.position.y = headWorldPos.y + 2.5;
      houseCloudGroup.position.z = headWorldPos.z;
    }
    
    let lightningLight = new THREE.PointLight(0xffffff, 0, 500);
    lightningLight.position.set(0, 50, 0);
    scene.add(lightningLight);
    
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
    
    function createTree() {
      const treeGroup = new THREE.Group();
      const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.2, 0.2, 2, 16),
                                     new THREE.MeshStandardMaterial({ color: 0x8B4513 }));
      trunk.position.y = -1;
      const foliage = new THREE.Mesh(new THREE.ConeGeometry(1, 3, 16),
                                       new THREE.MeshStandardMaterial({ color: 0x228B22 }));
      foliage.position.y = 0.5;
      treeGroup.add(trunk, foliage);
      return treeGroup;
    }
    
    for (let i = 0; i < 10; i++) {
      const tree = createTree();
      tree.position.set(-50 + i * 10, -2, -15);
      scene.add(tree);
    }
    
    function animate() {
      requestAnimationFrame(animate);
      
      const now = new Date();
      const headWorldPos = new THREE.Vector3();
      head.getWorldPosition(headWorldPos);
      const totalMin = now.getHours() * 60 + now.getMinutes();
      const angle = (totalMin / 1440) * Math.PI * 2;
      const radius = 3;
      const sunPos = new THREE.Vector3(
        headWorldPos.x + Math.cos(angle) * radius,
        headWorldPos.y + Math.sin(angle) * radius,
        headWorldPos.z
      );
      sun.position.copy(sunPos);
      
      const moonAngle = angle + Math.PI;
      const moonPos = new THREE.Vector3(
        headWorldPos.x + Math.cos(moonAngle) * radius,
        headWorldPos.y + Math.sin(moonAngle) * radius,
        headWorldPos.z
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
      
      const isDay = (t >= 7 && t < 17);
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
      
      updateWeatherEffects();
      updateHouseClouds();
      updateLightning();
      characterStreetlight.position.set(characterGroup.position.x + 1, -2, characterGroup.position.z);
      updateBubblePosition();
      
      if (cloudRainGroup.visible) {
        const particles = cloudRainGroup.children[0];
        let positions = particles.geometry.attributes.position.array;
        for (let i = 0; i < positions.length; i += 3) {
          positions[i+1] -= 0.02;
          if (positions[i+1] < -0.3) {
            positions[i+1] = Math.random() * 0.2;
          }
        }
        particles.geometry.attributes.position.needsUpdate = true;
      }
      
      renderer.render(scene, camera);
    }
    animate();
    
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
      document.getElementById("delete-day-event").addEventListener("click", () => {
        const dayStr = prompt("삭제할 하루일정의 날짜(일)를 입력하세요 (예: 15):");
        if(dayStr) {
          const dayNum = parseInt(dayStr);
          const eventDiv = document.getElementById(`event-${currentYear}-${currentMonth+1}-${dayNum}`);
          if(eventDiv) {
            eventDiv.textContent = "";
            alert(`${currentYear}-${currentMonth+1}-${dayNum} 일정이 삭제되었습니다. 다시 입력할 수 있습니다.`);
          }
        }
      });
      document.getElementById("save-calendar").addEventListener("click", () => {
        saveCalendar();
      });
    }
    
    function populateYearSelect() {
      const yearSelect = document.getElementById("year-select");
      yearSelect.innerHTML = "";
      for(let y = 2020; y <= 2070; y++){
        const option = document.createElement("option");
        option.value = y;
        option.textContent = y;
        if(y === currentYear) option.selected = true;
        yearSelect.appendChild(option);
      }
    }
    
    function renderCalendar(year, month) {
      const monthNames = ["1월","2월","3월","4월","5월","6월","7월","8월","9월","10월","11월","12월"];
      document.getElementById("month-year-label").textContent = `${year}년 ${monthNames[month]}`;
      const grid = document.getElementById("calendar-grid");
      grid.innerHTML = "";
      const daysOfWeek = ["일","월","화","수","목","금","토"];
      daysOfWeek.forEach((day) => {
        const th = document.createElement("div");
        th.style.fontWeight = "bold";
        th.style.textAlign = "center";
        th.textContent = day;
        th.style.color = "#00ffcc";
        th.style.textShadow = "0 0 3px #00ffcc";
        grid.appendChild(th);
      });
      const firstDay = new Date(year, month, 1).getDay();
      const daysInMonth = new Date(year, month+1, 0).getDate();
      for(let i = 0; i < firstDay; i++){
        grid.appendChild(document.createElement("div"));
      }
      for(let d = 1; d <= daysInMonth; d++){
        const cell = document.createElement("div");
        cell.innerHTML = `<div class="day-number">${d}</div>
                          <div class="event" id="event-${year}-${month+1}-${d}"></div>`;
        cell.addEventListener("click", () => {
          const eventText = prompt(`${year}-${month+1}-${d} 일정 입력:`);
          if(eventText) {
            const eventDiv = document.getElementById(`event-${year}-${month+1}-${d}`);
            if(eventDiv.textContent) {
              eventDiv.textContent += "; " + eventText;
            } else {
              eventDiv.textContent = eventText;
            }
          }
        });
        grid.appendChild(cell);
      }
    }
    
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
