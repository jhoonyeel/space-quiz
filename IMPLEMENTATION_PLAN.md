# Issue #3: 타이머와 연속 정답 추적 기능 구현 계획

## 📋 개요

Issue #3는 **타이머(30초 카운트다운)** 및 **연속 정답 추적** 기능을 추가하는 요구사항입니다. 경쟁심 있는 플레이어들이 문제 해결 속도와 정확도를 개선할 수 있도록 지원합니다.

---

## ✅ 수용 기준 (Acceptance Criteria)

1. ✓ 각 문제마다 남은 초 단위 시간 표시 (기본값: 30초)
2. ✓ 타이머 시각적 변화: 초록색 → 노란색 → 빨간색
3. ✓ 시간이 다 되면 자동으로 다음 문제로 스킵
4. ✓ 퀴즈 진행 중 현재 연속 정답 개수 표시
5. ✓ 최종 결과에 표시: 총 점수, 달성한 최고 연속 정답 수, 소요 시간
6. ✓ 오답 시 연속 정답 카운트 리셋
7. ✓ 타이머와 연속 정답 표시가 명확하고 읽기 쉬워야 함

---

## 🏗️ 기술 아키텍처

### 상태 관리 변경

```javascript
// 기존 상태
let currentQuestion = 0;
let score = 0;
let isAnswered = false;
let userAnswers = [];
let selectedDifficulty = null;
let currentQuizData = [];

// 추가될 상태
let timerInterval = null;          // 타이머 인터벌 ID
let remainingTime = 30;            // 남은 초 (0~30)
let streakCount = 0;               // 현재 연속 정답 개수
let maxStreak = 0;                 // 최고 연속 정답 수
let quizStartTime = null;          // 퀴즈 시작 시간 (밀리초)
let quizElapsedTime = 0;           // 총 소요 시간 (초)
```

### 핵심 함수 추가/수정

#### 1. 타이머 관련 함수

**`startTimer()`** - 새로운 타이머 시작
```javascript
function startTimer() {
    remainingTime = 30;
    clearInterval(timerInterval);
    updateTimerDisplay();
    
    timerInterval = setInterval(() => {
        remainingTime--;
        updateTimerDisplay();
        
        if (remainingTime <= 0) {
            clearInterval(timerInterval);
            handleTimeUp();  // 시간 만료 처리
        }
    }, 1000);  // 1초마다 실행
}
```

**`updateTimerDisplay()`** - 타이머 UI 업데이트
```javascript
function updateTimerDisplay() {
    const timerDisplay = document.getElementById('timerDisplay');
    timerDisplay.textContent = remainingTime + 's';
    
    // 색상 변화: 초록색(20+초) → 노란색(10~20초) → 빨간색(0~10초)
    const timerElement = document.getElementById('timerContainer');
    
    if (remainingTime > 20) {
        timerElement.classList.remove('timer-warning', 'timer-danger');
        timerElement.classList.add('timer-normal');
    } else if (remainingTime > 10) {
        timerElement.classList.remove('timer-normal', 'timer-danger');
        timerElement.classList.add('timer-warning');
    } else {
        timerElement.classList.remove('timer-normal', 'timer-warning');
        timerElement.classList.add('timer-danger');
    }
}
```

**`handleTimeUp()`** - 시간 만료 처리
```javascript
function handleTimeUp() {
    if (isAnswered) return;
    
    isAnswered = true;
    const options = document.querySelectorAll('.option');
    options.forEach(option => option.disabled = true);
    
    // 시간 만료 피드백
    document.getElementById('feedback').textContent = '⏰ 시간이 끝났습니다!';
    
    // 연속 정답 카운트 리셋 (오답 처리)
    streakCount = 0;
    
    // 기록 추가
    const correctIndex = currentQuizData[currentQuestion].correct;
    userAnswers.push({
        questionIndex: currentQuestion,
        userAnswer: -1,  // -1은 시간 만료를 의미
        correctAnswer: correctIndex,
        isCorrect: false
    });
    
    setTimeout(nextQuestion, 1500);
}
```

**`stopTimer()`** - 타이머 중지
```javascript
function stopTimer() {
    if (timerInterval) {
        clearInterval(timerInterval);
        timerInterval = null;
    }
}
```

#### 2. 연속 정답 추적 함수

**`updateStreakDisplay()`** - 연속 정답 UI 업데이트
```javascript
function updateStreakDisplay() {
    const streakDisplay = document.getElementById('streakDisplay');
    streakDisplay.textContent = streakCount;
    
    // 연속 정답이 있으면 강조 표시
    const streakContainer = document.getElementById('streakContainer');
    if (streakCount > 0) {
        streakContainer.classList.add('streak-active');
    } else {
        streakContainer.classList.remove('streak-active');
    }
}
```

**`updateStreak(isCorrect)`** - 정답 여부에 따라 연속 정답 업데이트
```javascript
function updateStreak(isCorrect) {
    if (isCorrect) {
        streakCount++;
        if (streakCount > maxStreak) {
            maxStreak = streakCount;
        }
    } else {
        streakCount = 0;
    }
    updateStreakDisplay();
}
```

#### 3. 기존 함수 수정

**`selectDifficulty(level)` 수정** - 타이머/시간 초기화
```javascript
function selectDifficulty(level) {
    selectedDifficulty = level;
    currentQuizData = quizDataByDifficulty[level];
    currentQuestion = 0;
    score = 0;
    isAnswered = false;
    userAnswers = [];
    streakCount = 0;           // ← 추가
    maxStreak = 0;             // ← 추가
    quizStartTime = Date.now(); // ← 추가
    
    document.getElementById('difficultySelection').classList.remove('active');
    document.getElementById('quizContainer').style.display = 'block';
    document.getElementById('results').classList.remove('active');
    
    displayQuestion();
}
```

**`displayQuestion()` 수정** - 타이머 시작
```javascript
function displayQuestion() {
    isAnswered = false;
    const quiz = currentQuizData[currentQuestion];
    document.getElementById('question').textContent = quiz.question;
    document.getElementById('feedback').textContent = '';
    
    // 선택지 렌더링 (기존 코드)
    const optionsContainer = document.getElementById('options');
    optionsContainer.innerHTML = '';
    quiz.options.forEach((option, index) => {
        const button = document.createElement('button');
        button.className = 'option';
        button.textContent = option;
        button.onclick = () => selectAnswer(index);
        optionsContainer.appendChild(button);
    });
    
    updateProgress();
    startTimer();  // ← 타이머 시작
}
```

**`selectAnswer(index)` 수정** - 타이머 중지 및 연속 정답 추적
```javascript
function selectAnswer(index) {
    if (isAnswered) return;
    isAnswered = true;
    
    stopTimer();  // ← 타이머 중지
    
    const options = document.querySelectorAll('.option');
    const correctIndex = currentQuizData[currentQuestion].correct;
    
    options.forEach(option => option.disabled = true);
    
    const isCorrect = index === correctIndex;
    userAnswers.push({
        questionIndex: currentQuestion,
        userAnswer: index,
        correctAnswer: correctIndex,
        isCorrect: isCorrect
    });
    
    if (isCorrect) {
        options[index].classList.add('correct');
        document.getElementById('feedback').textContent = '✅ 정답!';
        score++;
    } else {
        options[index].classList.add('incorrect');
        options[correctIndex].classList.add('correct');
        document.getElementById('feedback').textContent = '❌ 오답!';
    }
    
    updateStreak(isCorrect);  // ← 연속 정답 추적
    
    setTimeout(nextQuestion, 1500);
}
```

**`nextQuestion()` 수정** - 총 소요 시간 계산
```javascript
function nextQuestion() {
    currentQuestion++;
    if (currentQuestion < currentQuizData.length) {
        displayQuestion();
    } else {
        quizElapsedTime = Math.floor((Date.now() - quizStartTime) / 1000);  // ← 추가
        stopTimer();  // ← 타이머 중지
        showResults();
    }
}
```

**`showResults()` 수정** - 최고 연속 정답 및 소요 시간 표시
```javascript
function showResults() {
    document.getElementById('quizContainer').style.display = 'none';
    document.getElementById('results').classList.add('active');
    
    const percentage = Math.round((score / currentQuizData.length) * 100);
    const difficultyLabels = {
        easy: '쉬움',
        medium: '중간',
        hard: '어려움'
    };
    
    const difficultyLabel = difficultyLabels[selectedDifficulty];
    document.getElementById('finalScore').textContent = 
        `[${difficultyLabel}] ${score}/${currentQuizData.length} (${percentage}%)`;
    
    // 최고 기록 표시 ← 추가
    const maxStreakDisplay = document.getElementById('maxStreakDisplay');
    maxStreakDisplay.textContent = `최고 연속 정답: ${maxStreak}`;
    
    // 소요 시간 표시 ← 추가
    const elapsedTimeDisplay = document.getElementById('elapsedTimeDisplay');
    elapsedTimeDisplay.textContent = `소요 시간: ${quizElapsedTime}초`;
    
    // 기존 결과 표시 코드...
    let emoji, message;
    if (percentage === 100) {
        emoji = '🌟';
        message = '완벽해요! 당신은 우주 전문가입니다!';
    } else if (percentage >= 80) {
        emoji = '🎉';
        message = '대단해요! 우주에 대해 많이 알고 있네요!';
    } else if (percentage >= 60) {
        emoji = '👍';
        message = '잘했어요! 더 알아갈 게 있네요!';
    } else {
        emoji = '🤔';
        message = '다시 도전해 보세요! 다음엔 더 잘할 수 있어요!';
    }
    
    document.getElementById('emoji').textContent = emoji;
    document.getElementById('resultMessage').textContent = message;
    
    // 상세 결과 렌더링 (기존 코드)...
}
```

---

## 🎨 HTML 마크업 추가

### 점수 정보 영역 확장

**기존 코드:**
```html
<div class="score-info">
    <span>진행도</span>
    <span class="score-counter"><span id="current">1</span>/10</span>
</div>
```

**변경할 코드:**
```html
<div class="score-info">
    <span>진행도</span>
    <span class="score-counter"><span id="current">1</span>/10</span>
</div>

<!-- 타이머 추가 -->
<div class="timer-info">
    <div id="timerContainer" class="timer-container timer-normal">
        <span class="timer-label">⏱️ 시간</span>
        <span class="timer-display" id="timerDisplay">30s</span>
    </div>
    <div id="streakContainer" class="streak-container">
        <span class="streak-label">🔥 연속</span>
        <span class="streak-display" id="streakDisplay">0</span>
    </div>
</div>
```

### 결과 화면 확장

**추가 요소 (`.results` 내부):**
```html
<div class="results-details">
    <div class="detail-item">
        <span class="detail-label" id="maxStreakDisplay">최고 연속 정답: 0</span>
    </div>
    <div class="detail-item">
        <span class="detail-label" id="elapsedTimeDisplay">소요 시간: 0초</span>
    </div>
</div>
```

---

## 🎨 CSS 스타일링 추가

### 타이머 컨테이너 스타일

```css
.timer-info {
    display: flex;
    justify-content: space-between;
    gap: 15px;
    margin-bottom: 20px;
    animation: slideIn 0.3s ease;
}

.timer-container,
.streak-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 12px 16px;
    border-radius: 8px;
    background: #f0f0f0;
    border: 2px solid #e0e0e0;
    transition: all 0.3s ease;
}

@media (prefers-color-scheme: dark) {
    .timer-container,
    .streak-container {
        background: #2a2a3e;
        border-color: #333;
    }
}

.timer-label,
.streak-label {
    font-size: 12px;
    color: #666;
    margin-bottom: 4px;
    font-weight: 600;
}

@media (prefers-color-scheme: dark) {
    .timer-label,
    .streak-label {
        color: #aaa;
    }
}

.timer-display,
.streak-display {
    font-size: 24px;
    font-weight: bold;
}

/* 타이머 색상 상태 */
.timer-container.timer-normal {
    border-color: #10b981;
    background: #ecfdf5;
}

@media (prefers-color-scheme: dark) {
    .timer-container.timer-normal {
        background: #064e3b;
        border-color: #10b981;
    }
}

.timer-container.timer-normal .timer-display {
    color: #10b981;
}

.timer-container.timer-warning {
    border-color: #f59e0b;
    background: #fffbeb;
    animation: timerWarningPulse 0.6s ease-in-out infinite;
}

@media (prefers-color-scheme: dark) {
    .timer-container.timer-warning {
        background: #5a3a1f;
        border-color: #f59e0b;
    }
}

.timer-container.timer-warning .timer-display {
    color: #f59e0b;
}

.timer-container.timer-danger {
    border-color: #ef4444;
    background: #fef2f2;
    animation: timerDangerPulse 0.3s ease-in-out infinite;
}

@media (prefers-color-scheme: dark) {
    .timer-container.timer-danger {
        background: #7f1d1d;
        border-color: #ef4444;
    }
}

.timer-container.timer-danger .timer-display {
    color: #ef4444;
}

/* 연속 정답 강조 */
.streak-container {
    border-color: #8b5cf6;
    background: #f5f3ff;
}

@media (prefers-color-scheme: dark) {
    .streak-container {
        background: #3a2a4a;
        border-color: #8b5cf6;
    }
}

.streak-container .streak-display {
    color: #7c3aed;
}

.streak-container.streak-active {
    animation: streakPulse 0.4s ease-in-out;
    box-shadow: 0 0 15px rgba(139, 92, 246, 0.5);
}

@media (prefers-color-scheme: dark) {
    .streak-container.streak-active {
        box-shadow: 0 0 15px rgba(139, 92, 246, 0.4);
    }
}

/* 애니메이션 */
@keyframes timerWarningPulse {
    0%, 100% {
        box-shadow: 0 0 0 0 rgba(245, 158, 11, 0.3);
    }
    50% {
        box-shadow: 0 0 0 8px rgba(245, 158, 11, 0);
    }
}

@keyframes timerDangerPulse {
    0%, 100% {
        box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.5);
    }
    50% {
        box-shadow: 0 0 0 6px rgba(239, 68, 68, 0);
    }
}

@keyframes streakPulse {
    0% {
        transform: scale(1);
    }
    50% {
        transform: scale(1.1);
    }
    100% {
        transform: scale(1);
    }
}

/* 결과 화면에서 상세 정보 표시 */
.results-details {
    margin: 20px 0 30px 0;
    padding: 16px;
    background: #f5f5f5;
    border-radius: 8px;
    border-left: 4px solid #667eea;
}

@media (prefers-color-scheme: dark) {
    .results-details {
        background: #2a2a3e;
        border-left-color: #8b9eff;
    }
}

.detail-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 0;
    font-size: 16px;
}

.detail-item:not(:last-child) {
    margin-bottom: 12px;
}

.detail-label {
    color: #1a1a2e;
    font-weight: 600;
}

@media (prefers-color-scheme: dark) {
    .detail-label {
        color: #e0e0e0;
    }
}
```

---

## 🧪 테스트 포인트 (수동 검증)

### 기본 타이머 기능

- [ ] 퀴즈 시작 시 타이머가 30초부터 시작
- [ ] 매초 1씩 감소
- [ ] 타이머 색상 변화: 초록색(20+) → 노란색(10~20) → 빨간색(0~10)
- [ ] 타이머 색상 변화 시 시각적 펄스/애니메이션 재생

### 타이머 만료 처리

- [ ] 시간 만료 시 "⏰ 시간이 끝났습니다!" 메시지 표시
- [ ] 타이머 만료 시 연속 정답 카운트 리셋 (0으로 변경)
- [ ] 타이머 만료 시 자동으로 1.5초 뒤 다음 문제로 진행
- [ ] 기록에 시간 만료로 표시

### 연속 정답 추적

- [ ] 정답 선택 시 연속 정답 개수 +1 증가
- [ ] 오답 또는 타임아웃 시 연속 정답 개수 0으로 리셋
- [ ] 최고 연속 정답 수 추적 (퀴즈 진행 중 실시간 업데이트 안 함)
- [ ] 연속 정답 개수 표시가 명확하고 가독성 좋음

### 결과 화면

- [ ] 최고 연속 정답 수 표시 (예: "최고 연속 정답: 5")
- [ ] 총 소요 시간 표시 (예: "소요 시간: 127초")
- [ ] 다크 모드에서도 두 정보 모두 명확하게 표시
- [ ] 다시 시작하기 버튼 클릭 후 타이머/연속 정답 리셋 (초기값으로)

### 다크/라이트 모드

- [ ] 라이트 모드: 타이머/연속 정답 박스가 명확한 배경색
- [ ] 다크 모드: 타이머/연속 정답 박스가 다크 테마와 조화
- [ ] 타이머 색상 변화(초록→노란→빨강)가 두 모드 모두에서 구분 가능
- [ ] 결과 화면 상세 정보 박스가 모두 읽기 쉬운 색상

### 엣지 케이스

- [ ] 선택지를 매우 빨리 누른 후 타이머 영향 없음
- [ ] 모든 문제에서 올바른 연속 정답 수 전체 퀴즈 길이(10) 도달 가능
- [ ] 매우 느린 기기에서도 타이머 동작 일관성 있음

---

## 🔄 단계별 구현 순서

### Phase 1: 상태 관리 및 기본 타이머 (1시간)
1. 전역 상태 변수 추가 (`timerInterval`, `remainingTime`, `streakCount`, `maxStreak`, `quizStartTime`, `quizElapsedTime`)
2. `startTimer()` 함수 구현
3. `updateTimerDisplay()` 함수 구현
4. `stopTimer()` 함수 구현
5. `selectDifficulty()` 수정 (상태 초기화)
6. `displayQuestion()` 수정 (타이머 시작)
7. HTML 마크업 추가 (타이머 컨테이너)

### Phase 2: 타이머 UI 및 색상 (1시간)
1. CSS 스타일링 추가 (`.timer-container`, `.timer-normal`, `.timer-warning`, `.timer-danger`)
2. 애니메이션 추가 (`timerWarningPulse`, `timerDangerPulse`)
3. 다크 모드 대응 CSS 추가

### Phase 3: 시간 만료 처리 (30분)
1. `handleTimeUp()` 함수 구현
2. `stopTimer()` 호출 타이밍 수정
3. 기록 추가 로직 (시간 만료 처리)
4. 다음 문제 자동 진행 (1.5초 지연)

### Phase 4: 연속 정답 추적 (1시간)
1. `updateStreakDisplay()` 함수 구현
2. `updateStreak()` 함수 구현
3. `selectAnswer()` 수정 (연속 정답 호출)
4. HTML 마크업 추가 (연속 정답 컨테이너)
5. CSS 스타일링 추가 (`.streak-container`, `.streak-active`)
6. 애니메이션 추가 (`streakPulse`)

### Phase 5: 결과 화면 개선 (1시간)
1. `nextQuestion()` 수정 (총 소요 시간 계산)
2. `showResults()` 수정 (최고 연속 정답, 소요 시간 표시)
3. HTML 마크업 추가 (결과 상세 정보)
4. CSS 스타일링 추가 (`.results-details`, `.detail-item`)
5. 다크 모드 대응 CSS 추가

### Phase 6: 테스트 및 버그 수정 (1시간)
1. 전체 흐름 통합 테스트
2. 엣지 케이스 확인
3. 다크 모드 검증
4. 성능 최적화 (불필요한 리렌더링 제거)
5. 버그 수정

**총 예상 시간:** 약 5.5~6시간

---

## 📝 주요 고려사항

### 1. 타이머와 사용자 상호작용
- 타이머 만료 후에도 선택지를 누르면 작동하지 않도록 `isAnswered` 플래그 활용
- `handleTimeUp()` 호출 시 옵션 비활성화 필수

### 2. 연속 정답 카운트 리셋 시점
- 오답 선택 시 (클릭)
- 타이머 만료 시 (자동 진행)
- 재시작 시 (0으로 초기화)

### 3. 타이머 정확성
- `setInterval()` 사용 (1000ms 간격)
- 브라우저 최소화 또는 탭 비활성화 시 영향을 받을 수 있음 (일반적인 브라우저 동작)

### 4. 다크 모드 일관성
- 모든 새로운 요소에 `@media (prefers-color-scheme: dark)` 블록 추가
- 색상 대비 검증 필수

### 5. 접근성 (Accessibility)
- 타이머 및 연속 정답 수를 명확한 숫자로 표시
- 색상만으로 상태를 나타내지 않도록 주의 (라벨 텍스트 함께 제시)
- 이모지 + 텍스트 레이블 조합 사용

### 6. 모바일 반응성
- 타이머/연속 정답 박스가 좁은 화면에서도 균형 잡혀 보임
- Flexbox 사용으로 유연성 보장

---

## 🎯 예상 최종 결과물

### 퀴즈 진행 중 화면
```
🚀 우주 탐사 퀴즈

진행도                                                    1/10

⏱️ 시간          🔥 연속
[초록색]          [보라색]
  25s               2

[진행률 바]

질문 텍스트

[선택지 1]
[선택지 2]
[선택지 3]
[선택지 4]

✅ 정답!
```

### 결과 화면
```
🌟

[쉬움] 10/10 (100%)

완벽해요! 당신은 우주 전문가입니다!

┌─────────────────────────┐
│ 최고 연속 정답: 10       │
│ 소요 시간: 127초         │
└─────────────────────────┘

상세 결과
[기존 항목 리뷰]

[다시 시작하기]
```

---

## 📊 버전 관리

- **현재 파일:** `index.html`
- **단일 파일 유지:** 모든 변경사항 한 파일에 통합
- **커밋 메시지:** "feat: Add timer and streak tracking features"

---

## ✨ 완료 조건

모든 수용 기준이 충족되고:
- [ ] 수동 테스트 완료 (모든 체크포인트 통과)
- [ ] 다크/라이트 모드 모두 검증
- [ ] 코드 리뷰 통과
- [ ] GitHub Issue #3 완료 마크

