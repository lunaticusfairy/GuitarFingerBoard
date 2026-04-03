<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'

const CHROMATIC_SCALE = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
const NATURAL_NOTES = ['A', 'B', 'C', 'D', 'E', 'F', 'G']
const PENTATONIC_ROOT_NOTE = 'A'
const STRING_TUNINGS = [
  { stringNumber: 1, openNote: 'E' },
  { stringNumber: 2, openNote: 'B' },
  { stringNumber: 3, openNote: 'G' },
  { stringNumber: 4, openNote: 'D' },
  { stringNumber: 5, openNote: 'A' },
  { stringNumber: 6, openNote: 'E' },
]
const POSITION_MARKER_FRETS = [3, 5, 7, 9, 12]
const VISIBLE_FRET_COUNT = 13
const OPEN_AREA_RATIO = 0.08
const CORRECT_NEXT_DELAY = 900
const WRONG_NEXT_DELAY = 1400
const QUESTION_TIME_LIMIT = 5
const FRET_LIST = Array.from({ length: VISIBLE_FRET_COUNT }, (_, index) => index + 1)
const PENTATONIC_BOXES = [
  {
    boxNumber: 1,
    fretsByString: {
      1: [5, 8],
      2: [5, 8],
      3: [5, 7],
      4: [5, 7],
      5: [5, 7],
      6: [5, 8],
    },
  },
  {
    boxNumber: 2,
    fretsByString: {
      1: [8, 10],
      2: [8, 10],
      3: [7, 9],
      4: [7, 10],
      5: [7, 10],
      6: [8, 10],
    },
  },
  {
    boxNumber: 3,
    fretsByString: {
      1: [10, 12],
      2: [10, 13],
      3: [9, 12],
      4: [10, 12],
      5: [10, 12],
      6: [10, 12],
    },
  },
  {
    boxNumber: 4,
    fretsByString: {
      1: [3],
      2: [1, 3],
      3: [2],
      4: [2],
      5: [3],
      6: [3],
    },
  },
  {
    boxNumber: 5,
    fretsByString: {
      1: [3, 5],
      2: [3, 5],
      3: [2, 5],
      4: [2, 5],
      5: [3, 5],
      6: [3, 5],
    },
  },
]

const noteIndexMap = Object.fromEntries(CHROMATIC_SCALE.map((note, index) => [note, index]))
const highestVisibleFretRatio = 1 - Math.pow(2, -VISIBLE_FRET_COUNT / 12)

const activeMode = ref('natural')
const selectedBox = ref(1)
const currentQuestion = ref(null)
const selectedAnswer = ref('')
const isAnswered = ref(false)
const lastAnswerCorrect = ref(false)
const feedbackMessage = ref('')
const isRunning = ref(false)
const timeLeft = ref(QUESTION_TIME_LIMIT)
const showResultModal = ref(false)
const lastSessionResult = ref(null)
const statsByMode = reactive({
  natural: createStats(),
  pentatonic: createStats(),
})

let autoNextTimer = null
let countdownTimer = null

function createStats() {
  return {
    totalAnswers: 0,
    correctAnswers: 0,
    currentStreak: 0,
    bestStreak: 0,
  }
}

function shuffle(list) {
  const copied = [...list]

  for (let index = copied.length - 1; index > 0; index -= 1) {
    const swapIndex = Math.floor(Math.random() * (index + 1))
    ;[copied[index], copied[swapIndex]] = [copied[swapIndex], copied[index]]
  }

  return copied
}

function getNoteAtFret(openNote, fret) {
  const startIndex = noteIndexMap[openNote]
  return CHROMATIC_SCALE[(startIndex + fret) % CHROMATIC_SCALE.length]
}

function getStringPercent(stringNumber) {
  return ((stringNumber - 1) / (STRING_TUNINGS.length - 1)) * 100
}

function getPositionLabel(fret) {
  return fret === 0 ? '개방현' : `${fret}프렛`
}

function getFretBoundaryRatio(fret) {
  if (fret <= 0) {
    return 0
  }

  const rawRatio = 1 - Math.pow(2, -fret / 12)
  return rawRatio / highestVisibleFretRatio
}

function getFretCenterRatio(fret) {
  const start = getFretBoundaryRatio(fret - 1)
  const end = getFretBoundaryRatio(fret)
  return (start + end) / 2
}

function getBoardRatio(scaleRatio) {
  return OPEN_AREA_RATIO + scaleRatio * (1 - OPEN_AREA_RATIO)
}

function getFretPercent(fret) {
  if (fret === 0) {
    return (OPEN_AREA_RATIO * 0.5) * 100
  }

  return getBoardRatio(getFretCenterRatio(fret)) * 100
}

function getFretBoundaryPercent(fret) {
  return getBoardRatio(getFretBoundaryRatio(fret)) * 100
}

const fretboard = STRING_TUNINGS.map(({ stringNumber, openNote }) => {
  const frets = [0, ...FRET_LIST].map((fret) => {
    const note = getNoteAtFret(openNote, fret)

    return {
      key: `${stringNumber}-${fret}`,
      stringNumber,
      fret,
      note,
      isNatural: NATURAL_NOTES.includes(note),
    }
  })

  return {
    stringNumber,
    frets,
  }
})

const fretCellMap = new Map(
  fretboard.flatMap((row) => row.frets.map((cell) => [cell.key, cell])),
)

const naturalQuizPool = fretboard.flatMap((row) => row.frets.filter((cell) => cell.isNatural))

const pentatonicBoxes = PENTATONIC_BOXES.map(({ boxNumber, fretsByString }) => {
  const cells = Object.entries(fretsByString)
    .flatMap(([stringNumber, frets]) =>
      frets
        .map((fret) => fretCellMap.get(`${stringNumber}-${fret}`))
        .filter(Boolean)
        .map((cell) => ({
          ...cell,
          isRoot: cell.note === PENTATONIC_ROOT_NOTE,
        })),
    )
    .sort((left, right) => {
      if (left.stringNumber === right.stringNumber) {
        return left.fret - right.fret
      }

      return left.stringNumber - right.stringNumber
    })

  return {
    boxNumber,
    cells,
  }
})

const currentStats = computed(() => statsByMode[activeMode.value])
const selectedPentatonicBox = computed(
  () => pentatonicBoxes.find((box) => box.boxNumber === selectedBox.value) ?? pentatonicBoxes[0],
)
const visibleBoxCells = computed(() =>
  activeMode.value === 'pentatonic' ? selectedPentatonicBox.value.cells : [],
)
const accuracy = computed(() => {
  if (!currentStats.value.totalAnswers) {
    return 0
  }

  return Math.round((currentStats.value.correctAnswers / currentStats.value.totalAnswers) * 100)
})

const resultHeadline = computed(() => {
  if (!lastSessionResult.value) {
    return ''
  }

  if (lastSessionResult.value.totalAnswers === 0) {
    return '아직 몸 풀기 전이에요'
  }

  if (lastSessionResult.value.accuracy >= 90) {
    return '와, 정말 대단해요'
  }

  if (lastSessionResult.value.accuracy >= 70) {
    return '아주 잘하고 있어요'
  }

  if (lastSessionResult.value.accuracy >= 40) {
    return '조금만 더 하면 잡혀요'
  }

  return '괜찮아요, 다시 해보면 돼요'
})

const resultModeLabel = computed(() => {
  if (!lastSessionResult.value) {
    return ''
  }

  return lastSessionResult.value.mode === 'natural'
    ? '자연음 퀴즈 결과'
    : `A minor pentatonic box ${lastSessionResult.value.boxNumber} 결과`
})

const resultCopy = computed(() => {
  if (!lastSessionResult.value) {
    return ''
  }

  if (lastSessionResult.value.totalAnswers === 0) {
    return '시작 버튼만 누르면 우주 하늘 기타 탐험이 바로 시작돼요.'
  }

  if (lastSessionResult.value.accuracy >= 90) {
    return '지판이 손에 꽤 익었어요. 지금 흐름이면 우주 하늘 앞에서 멋지게 뽐낼 수 있겠어요.'
  }

  if (lastSessionResult.value.accuracy >= 70) {
    return '감이 아주 좋아요. 몇 번만 더 하면 위치가 더 또렷하게 기억날 거예요.'
  }

  if (lastSessionResult.value.accuracy >= 40) {
    return '충분히 잘 가고 있어요. 조금 헷갈린 자리만 다시 보면 금방 더 좋아질 거예요.'
  }

  return '괜찮아요. 틀리면서 외우는 게 제일 빠를 때도 많아요. 바로 한 번 더 해봐요.'
})

function buildSessionResult() {
  const stats = statsByMode[activeMode.value]
  const totalAnswers = stats.totalAnswers
  const correctAnswers = stats.correctAnswers

  return {
    mode: activeMode.value,
    boxNumber: selectedBox.value,
    totalAnswers,
    correctAnswers,
    bestStreak: stats.bestStreak,
    accuracy: totalAnswers ? Math.round((correctAnswers / totalAnswers) * 100) : 0,
  }
}

function clearAutoNextTimer() {
  if (autoNextTimer) {
    clearTimeout(autoNextTimer)
    autoNextTimer = null
  }
}

function clearCountdownTimer() {
  if (countdownTimer) {
    clearInterval(countdownTimer)
    countdownTimer = null
  }
}

function clearActiveTimers() {
  clearAutoNextTimer()
  clearCountdownTimer()
}

function buildNaturalChoices(correctNote) {
  const distractors = shuffle(NATURAL_NOTES.filter((note) => note !== correctNote)).slice(0, 3)
  return shuffle([correctNote, ...distractors])
}

function buildPentatonicChoices() {
  return ['루트(A)', '루트 아님']
}

function getCurrentPool() {
  if (activeMode.value === 'natural') {
    return naturalQuizPool
  }

  return selectedPentatonicBox.value.cells
}

function nextQuestion() {
  clearActiveTimers()

  if (!isRunning.value) {
    currentQuestion.value = null
    return
  }

  const pool = getCurrentPool()

  if (!pool.length) {
    currentQuestion.value = null
    return
  }

  const previousKey = currentQuestion.value?.key
  let nextCell = pool[Math.floor(Math.random() * pool.length)]

  while (pool.length > 1 && nextCell.key === previousKey) {
    nextCell = pool[Math.floor(Math.random() * pool.length)]
  }

  if (activeMode.value === 'natural') {
    currentQuestion.value = {
      ...nextCell,
      quizType: 'natural',
      choices: buildNaturalChoices(nextCell.note),
    }
  } else {
    currentQuestion.value = {
      ...nextCell,
      quizType: 'pentatonic',
      isRoot: nextCell.note === PENTATONIC_ROOT_NOTE,
      choices: buildPentatonicChoices(),
    }
  }

  selectedAnswer.value = ''
  isAnswered.value = false
  lastAnswerCorrect.value = false
  feedbackMessage.value = ''
  timeLeft.value = QUESTION_TIME_LIMIT
  startCountdown()
}

function scheduleNextQuestion(delay) {
  clearActiveTimers()
  autoNextTimer = setTimeout(() => {
    nextQuestion()
  }, delay)
}

function answerQuestion(choice) {
  if (!isRunning.value || !currentQuestion.value || isAnswered.value) {
    return
  }

  clearCountdownTimer()

  const stats = currentStats.value
  let isCorrect = false

  if (activeMode.value === 'natural') {
    isCorrect = choice === currentQuestion.value.note
    feedbackMessage.value = isCorrect
      ? `정답입니다. ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}은 ${currentQuestion.value.note}입니다.`
      : `오답입니다. 정답은 ${currentQuestion.value.note}이고, 위치는 ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}입니다.`
  } else {
    const expectedAnswer = currentQuestion.value.isRoot ? '루트(A)' : '루트 아님'
    isCorrect = choice === expectedAnswer
    feedbackMessage.value = isCorrect
      ? currentQuestion.value.isRoot
        ? `정답입니다. ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}은 루트 A입니다.`
        : `정답입니다. ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}은 루트가 아닙니다.`
      : currentQuestion.value.isRoot
        ? `오답입니다. 이 위치는 루트 A입니다. ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}을 기억해 두세요.`
        : `오답입니다. 이 위치는 루트가 아닙니다. A minor pentatonic 박스 안의 다른 점들과 함께 비교해 보세요.`
  }

  selectedAnswer.value = choice
  isAnswered.value = true
  lastAnswerCorrect.value = isCorrect
  stats.totalAnswers += 1

  if (isCorrect) {
    stats.correctAnswers += 1
    stats.currentStreak += 1
    stats.bestStreak = Math.max(stats.bestStreak, stats.currentStreak)
    scheduleNextQuestion(CORRECT_NEXT_DELAY)
  } else {
    stats.currentStreak = 0
    scheduleNextQuestion(WRONG_NEXT_DELAY)
  }
}

function resetCurrentModeStats() {
  Object.assign(statsByMode[activeMode.value], createStats())
}

function markTimedOutAnswer() {
  if (!isRunning.value || !currentQuestion.value || isAnswered.value) {
    return
  }

  const stats = currentStats.value

  isAnswered.value = true
  lastAnswerCorrect.value = false
  selectedAnswer.value = ''
  stats.totalAnswers += 1
  stats.currentStreak = 0

  if (activeMode.value === 'natural') {
    feedbackMessage.value = `시간 초과입니다. 정답은 ${currentQuestion.value.note}이고, 위치는 ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}입니다.`
  } else {
    feedbackMessage.value = currentQuestion.value.isRoot
      ? `시간 초과입니다. 이 위치는 루트 A입니다.`
      : '시간 초과입니다. 이 위치는 루트가 아닙니다.'
  }

  scheduleNextQuestion(WRONG_NEXT_DELAY)
}

function startCountdown() {
  clearCountdownTimer()

  countdownTimer = setInterval(() => {
    if (!isRunning.value || isAnswered.value) {
      clearCountdownTimer()
      return
    }

    if (timeLeft.value <= 1) {
      timeLeft.value = 0
      clearCountdownTimer()
      markTimedOutAnswer()
      return
    }

    timeLeft.value -= 1
  }, 1000)
}

function stopQuiz(showResults = true) {
  const shouldShowResult = showResults && isRunning.value

  isRunning.value = false
  clearActiveTimers()
  currentQuestion.value = null
  selectedAnswer.value = ''
  isAnswered.value = false
  lastAnswerCorrect.value = false
  feedbackMessage.value = ''
  timeLeft.value = QUESTION_TIME_LIMIT

  if (shouldShowResult) {
    lastSessionResult.value = buildSessionResult()
    showResultModal.value = true
  }
}

function startQuiz() {
  showResultModal.value = false
  lastSessionResult.value = null
  resetCurrentModeStats()
  isRunning.value = true
  timeLeft.value = QUESTION_TIME_LIMIT
  nextQuestion()
}

function closeResultModal() {
  showResultModal.value = false
}

function switchMode(mode) {
  if (activeMode.value === mode) {
    return
  }

  showResultModal.value = false
  lastSessionResult.value = null
  activeMode.value = mode
  stopQuiz(false)
}

function selectBox(boxNumber) {
  if (selectedBox.value === boxNumber) {
    return
  }

  selectedBox.value = boxNumber

  if (activeMode.value === 'pentatonic') {
    if (isRunning.value) {
      nextQuestion()
    }
  }
}

function getStringStyle(stringNumber) {
  return {
    top: `${getStringPercent(stringNumber)}%`,
  }
}

function getFretLineStyle(fret) {
  return {
    left: `${getFretBoundaryPercent(fret)}%`,
  }
}

function getPositionMarkerStyle(fret) {
  return {
    left: `${getFretPercent(fret)}%`,
  }
}

function getFretNumberStyle(fret) {
  return {
    left: `${getFretPercent(fret)}%`,
  }
}

function getQuestionMarkerStyle(cell) {
  return {
    left: `${getFretPercent(cell.fret)}%`,
    top: `${getStringPercent(cell.stringNumber)}%`,
  }
}

onMounted(() => {
  stopQuiz(false)
})

onBeforeUnmount(() => {
  clearActiveTimers()
})
</script>

<template>
  <main class="app-shell">
    <section class="card fretboard-card">
      <div class="board-topbar">
        <div class="board-topbar-left">
          <div class="mode-toggle">
            <button
              type="button"
              class="mode-button"
              :class="{ active: activeMode === 'natural' }"
              @click="switchMode('natural')"
            >
              자연음
            </button>
            <button
              type="button"
              class="mode-button"
              :class="{ active: activeMode === 'pentatonic' }"
              @click="switchMode('pentatonic')"
            >
              A minor pentatonic
            </button>
          </div>

          <span class="board-title">사랑하는 우주 하늘을 위한 기타지판퀴즈</span>

          <div class="session-controls">
            <button type="button" class="session-button start" @click="startQuiz">
              시작
            </button>
            <button
              type="button"
              class="session-button stop"
              :disabled="!isRunning"
              @click="stopQuiz"
            >
              종료
            </button>
          </div>
        </div>

        <div class="stats-inline">
          <span>정답 <strong>{{ currentStats.correctAnswers }}</strong></span>
          <span>총문제 <strong>{{ currentStats.totalAnswers }}</strong></span>
          <span>정답률 <strong>{{ accuracy }}%</strong></span>
          <span>최고연속 <strong>{{ currentStats.bestStreak }}</strong></span>
          <span>남은초 <strong>{{ isRunning ? timeLeft : '-' }}</strong></span>
        </div>
      </div>

      <div v-if="activeMode === 'pentatonic'" class="box-toolbar">
        <span class="box-toolbar-label">Box</span>
        <div class="box-toolbar-buttons">
          <button
            v-for="box in pentatonicBoxes"
            :key="box.boxNumber"
            type="button"
            class="box-button"
            :class="{ active: selectedBox === box.boxNumber }"
            @click="selectBox(box.boxNumber)"
          >
            {{ box.boxNumber }}
          </button>
        </div>
      </div>

      <div class="board-scroll">
        <div class="fretboard-figure">
          <div class="string-labels">
            <div class="label-layer">
              <div
                v-for="row in fretboard"
                :key="`label-${row.stringNumber}`"
                class="string-row-label"
                :style="getStringStyle(row.stringNumber)"
              >
                <span class="string-number">{{ row.stringNumber }}</span>
              </div>
            </div>
          </div>

          <div class="board-column">
            <div class="board-area">
              <div class="board-layer">
                <div class="nut-line"></div>

                <div
                  v-for="fret in FRET_LIST"
                  :key="`fret-line-${fret}`"
                  class="fret-line"
                  :style="getFretLineStyle(fret)"
                ></div>

                <div
                  v-for="row in fretboard"
                  :key="`string-${row.stringNumber}`"
                  class="string-line"
                  :style="getStringStyle(row.stringNumber)"
                ></div>

                <div
                  v-for="fret in POSITION_MARKER_FRETS"
                  :key="`position-${fret}`"
                  class="position-marker"
                  :style="getPositionMarkerStyle(fret)"
                >
                  <template v-if="fret === 12">
                    <span class="position-dot upper"></span>
                    <span class="position-dot lower"></span>
                  </template>
                  <span v-else class="position-dot single"></span>
                </div>

                <div
                  v-for="cell in visibleBoxCells"
                  :key="`box-cell-${cell.key}`"
                  class="box-note-marker"
                  :style="getQuestionMarkerStyle(cell)"
                ></div>

                <div
                  v-if="currentQuestion"
                  class="question-marker"
                  :class="{
                    correct: isAnswered && lastAnswerCorrect,
                    wrong: isAnswered && !lastAnswerCorrect,
                  }"
                  :style="getQuestionMarkerStyle(currentQuestion)"
                ></div>
              </div>
            </div>

            <div class="fret-numbers">
              <span
                v-for="fret in FRET_LIST"
                :key="`fret-number-${fret}`"
                class="fret-number"
                :style="getFretNumberStyle(fret)"
              >
                {{ fret }}
              </span>
            </div>
          </div>
        </div>
      </div>
    </section>

    <section class="card quiz-card">
      <div
        v-if="currentQuestion && isRunning"
        class="choices-grid"
        :class="{ 'is-binary': activeMode === 'pentatonic' }"
      >
        <button
          v-for="(choice, index) in currentQuestion.choices"
          :key="choice"
          type="button"
          class="choice-button"
          :class="{
            'is-correct':
              isAnswered &&
              ((activeMode === 'natural' && choice === currentQuestion.note) ||
                (activeMode === 'pentatonic' &&
                  ((currentQuestion.isRoot && choice === '루트(A)') ||
                    (!currentQuestion.isRoot && choice === '루트 아님')))),
            'is-wrong':
              isAnswered &&
              choice === selectedAnswer &&
              !(
                (activeMode === 'natural' && choice === currentQuestion.note) ||
                (activeMode === 'pentatonic' &&
                  ((currentQuestion.isRoot && choice === '루트(A)') ||
                    (!currentQuestion.isRoot && choice === '루트 아님')))
              ),
            'is-muted':
              isAnswered &&
              choice !== selectedAnswer &&
              !(
                (activeMode === 'natural' && choice === currentQuestion.note) ||
                (activeMode === 'pentatonic' &&
                  ((currentQuestion.isRoot && choice === '루트(A)') ||
                    (!currentQuestion.isRoot && choice === '루트 아님')))
              ),
          }"
          :disabled="isAnswered"
          @click="answerQuestion(choice)"
        >
          <span class="choice-index">{{ index + 1 }}</span>
          <span class="choice-note">{{ choice }}</span>
        </button>
      </div>

      <div
        v-if="isAnswered && !lastAnswerCorrect"
        class="feedback-box error"
      >
        {{ feedbackMessage }}
      </div>
    </section>

    <div
      v-if="showResultModal && lastSessionResult"
      class="result-modal-backdrop"
      @click.self="closeResultModal"
    >
      <div class="result-modal">
        <p class="result-kicker">{{ resultModeLabel }}</p>
        <h2 class="result-title">{{ resultHeadline }}</h2>
        <p class="result-copy">{{ resultCopy }}</p>

        <div class="result-grid">
          <article class="result-card">
            <span>정답</span>
            <strong>{{ lastSessionResult.correctAnswers }}</strong>
          </article>
          <article class="result-card">
            <span>총 문제</span>
            <strong>{{ lastSessionResult.totalAnswers }}</strong>
          </article>
          <article class="result-card">
            <span>정답률</span>
            <strong>{{ lastSessionResult.accuracy }}%</strong>
          </article>
          <article class="result-card">
            <span>최고 연속</span>
            <strong>{{ lastSessionResult.bestStreak }}</strong>
          </article>
        </div>

        <div class="result-actions">
          <button type="button" class="result-button secondary" @click="closeResultModal">
            닫기
          </button>
          <button type="button" class="result-button primary" @click="startQuiz">
            다시 시작
          </button>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
.app-shell {
  width: min(980px, calc(100% - 20px));
  margin: 0 auto;
  padding: 12px 0 20px;
}

.card {
  border-radius: 28px;
  border: 1px solid rgba(76, 52, 31, 0.12);
  background:
    radial-gradient(circle at top right, rgba(245, 204, 143, 0.28), transparent 32%),
    linear-gradient(180deg, rgba(255, 250, 242, 0.96), rgba(246, 238, 226, 0.98));
  box-shadow:
    0 18px 48px rgba(92, 67, 38, 0.12),
    inset 0 1px 0 rgba(255, 255, 255, 0.7);
}

.card {
  padding: 12px;
  margin-top: 10px;
}

.board-topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 6px 12px;
  margin-bottom: 6px;
}

.board-topbar-left {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
  flex-wrap: wrap;
}

.board-title {
  color: #2f2015;
  font-size: 0.76rem;
  font-weight: 700;
  line-height: 1.2;
  white-space: nowrap;
}

.session-controls {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.session-button {
  min-height: 22px;
  padding: 0 8px;
  border: 0;
  border-radius: 999px;
  font: inherit;
  font-size: 0.68rem;
  cursor: pointer;
}

.session-button.start {
  color: #fff8f1;
  background: linear-gradient(135deg, #bb5a2f, #e38b2d);
}

.session-button.stop {
  color: #5a4331;
  background: rgba(255, 255, 255, 0.78);
  border: 1px solid rgba(121, 88, 56, 0.14);
}

.session-button:disabled {
  opacity: 0.5;
  cursor: default;
}

.mode-toggle {
  display: inline-flex;
  gap: 6px;
  margin-bottom: 0;
  padding: 3px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(129, 88, 47, 0.12);
}

.mode-button {
  min-height: 24px;
  padding: 0 10px;
  border: 0;
  border-radius: 999px;
  background: transparent;
  color: #6a5038;
  font: inherit;
  font-size: 0.72rem;
  cursor: pointer;
}

.mode-button.active {
  color: #fff9f2;
  background: linear-gradient(135deg, #ba6133, #de8b34);
}

.stats-inline {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: flex-end;
  margin-left: auto;
  color: #6a5038;
  font-size: 0.72rem;
  line-height: 1.2;
}

.stats-inline span {
  white-space: nowrap;
}

.stats-inline strong {
  color: #1f140d;
  font-size: 0.82rem;
}

.box-toolbar {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}

.box-toolbar-label {
  font-size: 0.76rem;
  font-weight: 700;
  color: #6a5038;
}

.box-toolbar-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.box-button {
  min-width: 34px;
  height: 30px;
  border: 0;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.76);
  color: #5f4735;
  border: 1px solid rgba(121, 88, 56, 0.12);
  font: inherit;
  font-size: 0.8rem;
  cursor: pointer;
}

.box-button.active {
  color: #fff9f2;
  background: linear-gradient(135deg, #ba6133, #de8b34);
  border-color: transparent;
}

.section-head {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: flex-start;
  margin-bottom: 14px;
}

.section-head.compact {
  margin-bottom: 8px;
}

.board-scroll {
  overflow: hidden;
}

.fretboard-figure {
  width: 100%;
  display: grid;
  grid-template-columns: 28px minmax(0, 1fr);
  gap: 8px;
  align-items: start;
}

.string-labels,
.board-area {
  --board-height: 150px;
  --board-pad: 12px;
  --open-space: 8%;
  position: relative;
  height: var(--board-height);
}

.label-layer,
.board-layer {
  position: absolute;
  inset: var(--board-pad) 0;
}

.string-row-label {
  position: absolute;
  right: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  width: 100%;
  transform: translateY(-50%);
}

.string-number {
  min-width: 12px;
  font-size: 0.8rem;
  font-weight: 700;
  color: #422e1f;
  text-align: right;
}

.board-column {
  min-width: 0;
}

.board-area {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.38), rgba(255, 255, 255, 0.12));
  border-radius: 14px;
}

.nut-line,
.fret-line,
.string-line,
.position-marker,
.question-marker,
.box-note-marker {
  position: absolute;
}

.nut-line {
  left: var(--open-space);
  top: -2px;
  bottom: -2px;
  width: 5px;
  border-radius: 999px;
  background: #17100b;
}

.fret-line {
  top: 0;
  bottom: 0;
  width: 3px;
  background: rgba(29, 20, 14, 0.92);
  transform: translateX(-50%);
}

.string-line {
  left: 0;
  right: 0;
  height: 1px;
  transform: translateY(-50%);
  background: rgba(26, 18, 12, 0.92);
}

.position-marker {
  top: 0;
  bottom: 0;
  width: 20px;
  transform: translateX(-50%);
}

.position-dot {
  position: absolute;
  left: 50%;
  width: 9px;
  height: 9px;
  border-radius: 999px;
  transform: translate(-50%, -50%);
  background: rgba(121, 104, 90, 0.3);
}

.position-dot.single {
  top: 50%;
}

.position-dot.upper {
  top: 30%;
}

.position-dot.lower {
  top: 70%;
}

.box-note-marker {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  transform: translate(-50%, -50%);
  background: rgba(118, 102, 87, 0.38);
}

.question-marker {
  width: 15px;
  height: 15px;
  border-radius: 999px;
  transform: translate(-50%, -50%);
  background: radial-gradient(circle at 30% 30%, #ff9e93, #ff4035 65%, #da251e);
  box-shadow:
    0 0 0 2px rgba(255, 255, 255, 0.72),
    0 4px 8px rgba(189, 62, 46, 0.2);
}

.question-marker.correct {
  background: radial-gradient(circle at 30% 30%, #a7f4bf, #49b56f 68%, #2e8f53);
  box-shadow:
    0 0 0 2px rgba(255, 255, 255, 0.72),
    0 4px 8px rgba(73, 181, 111, 0.2);
}

.question-marker.wrong {
  background: radial-gradient(circle at 30% 30%, #ffb0a8, #df584c 68%, #b53c31);
}

.fret-numbers {
  position: relative;
  height: 14px;
  margin-top: 2px;
}

.fret-number {
  position: absolute;
  top: 0;
  transform: translateX(-50%);
  font-size: 0.76rem;
  color: #4e3b2d;
  white-space: nowrap;
}

.best-streak {
  display: inline-flex;
  align-items: center;
  min-height: 30px;
  padding: 0 12px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.76);
  border: 1px solid rgba(120, 87, 56, 0.12);
  color: #6a5038;
  font-size: 0.82rem;
}

.quiz-card {
  padding: 10px 12px;
}

.choices-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 6px;
  margin-top: 0;
}

.choices-grid.is-binary {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.choice-button {
  min-height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 0 6px;
  border: 0;
  border-radius: 10px;
  background: #fff8ec;
  color: #25170e;
  font: inherit;
  cursor: pointer;
  transition:
    transform 0.16s ease,
    box-shadow 0.16s ease,
    background 0.16s ease;
  box-shadow: 0 10px 18px rgba(92, 67, 38, 0.08);
}

.choice-button:hover:enabled {
  transform: translateY(-2px);
  box-shadow: 0 14px 22px rgba(92, 67, 38, 0.12);
}

.choice-button:focus-visible,
.mode-button:focus-visible,
.box-button:focus-visible,
.session-button:focus-visible,
.result-button:focus-visible {
  outline: 3px solid rgba(255, 164, 65, 0.6);
  outline-offset: 2px;
}

.choice-button.is-correct {
  background: #dbf2dd;
  color: #1d4c2a;
}

.choice-button.is-wrong {
  background: #fbe0dc;
  color: #8b2b1d;
}

.choice-button.is-muted {
  background: #f0e6d7;
  color: #8b7862;
}

.choice-button:disabled {
  cursor: default;
}

.choice-index {
  width: 18px;
  height: 18px;
  border-radius: 999px;
  display: grid;
  place-items: center;
  background: rgba(129, 88, 47, 0.12);
  font-weight: 700;
  font-size: 0.62rem;
}

.choice-note {
  font-size: clamp(0.76rem, 1.2vw, 0.96rem);
  font-weight: 800;
  letter-spacing: 0.02em;
}

.feedback-box {
  min-height: 0;
  margin-top: 6px;
  padding: 8px 10px;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.76);
  border: 1px solid rgba(119, 87, 56, 0.12);
  color: #5e4633;
  line-height: 1.4;
  font-size: 0.82rem;
}

.feedback-box.error {
  background: #fff1ee;
  color: #7f3427;
  border-color: rgba(193, 93, 71, 0.18);
}

.result-modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 20;
  display: grid;
  place-items: center;
  padding: 20px;
  background: rgba(38, 25, 16, 0.38);
  backdrop-filter: blur(6px);
}

.result-modal {
  width: min(460px, 100%);
  padding: 22px;
  border-radius: 26px;
  border: 1px solid rgba(76, 52, 31, 0.12);
  background:
    radial-gradient(circle at top right, rgba(245, 204, 143, 0.3), transparent 34%),
    linear-gradient(180deg, rgba(255, 252, 246, 0.98), rgba(246, 236, 222, 0.98));
  box-shadow:
    0 24px 50px rgba(44, 27, 16, 0.18),
    inset 0 1px 0 rgba(255, 255, 255, 0.8);
}

.result-kicker {
  margin: 0;
  color: #8a6540;
  font-size: 0.72rem;
  letter-spacing: 0.18em;
  text-transform: uppercase;
}

.result-title {
  margin: 8px 0 0;
  color: #2b1d13;
  font-size: clamp(1.4rem, 2.6vw, 2rem);
  line-height: 1.05;
}

.result-copy {
  margin: 10px 0 0;
  color: #6a5038;
  font-size: 0.9rem;
  line-height: 1.5;
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  margin-top: 16px;
}

.result-card {
  padding: 14px;
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(121, 88, 56, 0.1);
}

.result-card span {
  display: block;
  color: #876646;
  font-size: 0.76rem;
}

.result-card strong {
  display: block;
  margin-top: 6px;
  color: #24170f;
  font-size: 1.4rem;
}

.result-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 18px;
}

.result-button {
  min-height: 38px;
  padding: 0 16px;
  border: 0;
  border-radius: 999px;
  font: inherit;
  font-size: 0.82rem;
  cursor: pointer;
}

.result-button.primary {
  color: #fff8f1;
  background: linear-gradient(135deg, #bb5a2f, #e38b2d);
}

.result-button.secondary {
  color: #5a4331;
  background: rgba(255, 255, 255, 0.84);
  border: 1px solid rgba(121, 88, 56, 0.14);
}

@media (max-width: 980px) {
  .app-shell {
    width: min(100% - 16px, 100%);
    padding: 10px 0 18px;
  }

  .card {
    padding: 10px;
  }

  .section-head {
    flex-direction: column;
  }
}

@media (max-width: 640px) {
  .mode-toggle {
    flex-wrap: wrap;
    border-radius: 14px;
  }

  .board-topbar {
    align-items: flex-start;
  }

  .board-topbar-left {
    gap: 6px;
  }

  .board-title {
    font-size: 0.64rem;
    white-space: normal;
  }

  .session-button {
    min-height: 20px;
    padding: 0 7px;
    font-size: 0.62rem;
  }

  .fretboard-figure {
    grid-template-columns: 24px minmax(0, 1fr);
    gap: 6px;
  }

  .string-labels,
  .board-area {
    --board-height: 128px;
    --board-pad: 10px;
  }

  .string-number,
  .fret-number {
    font-size: 0.74rem;
  }

  .quiz-card {
    padding: 8px 10px;
  }

  .choices-grid {
    gap: 4px;
  }

  .choice-button {
    min-height: 34px;
    padding: 0 4px;
  }

  .choice-index {
    width: 16px;
    height: 16px;
    font-size: 0.56rem;
  }

  .choice-note {
    font-size: 0.72rem;
  }

  .box-note-marker {
    width: 7px;
    height: 7px;
  }

  .question-marker {
    width: 13px;
    height: 13px;
  }

  .stats-inline {
    gap: 6px;
    font-size: 0.64rem;
    margin-left: 0;
    width: 100%;
    justify-content: flex-end;
  }

  .result-modal {
    padding: 18px;
    border-radius: 22px;
  }

  .result-grid {
    gap: 8px;
  }

  .result-card {
    padding: 12px;
    border-radius: 16px;
  }
}
</style>
