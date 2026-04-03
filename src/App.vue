<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'

const CHROMATIC_SCALE = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
const NATURAL_NOTES = ['A', 'B', 'C', 'D', 'E', 'F', 'G']
const PENTATONIC_TYPE_CONFIG = {
  minor: {
    buttonLabel: 'Minor Pentatonic',
    scaleSuffix: 'minor',
    intervals: [0, 3, 5, 7, 10],
    degreeLabels: ['R', 'b3', '4', '5', 'b7'],
    commonKeys: ['A', 'E', 'D', 'G', 'C'],
  },
  major: {
    buttonLabel: 'Major Pentatonic',
    scaleSuffix: 'major',
    intervals: [0, 2, 4, 7, 9],
    degreeLabels: ['R', '2', '3', '5', '6'],
    commonKeys: ['C', 'G', 'D', 'A', 'E'],
  },
}
const STRING_TUNINGS = [
  { stringNumber: 1, openNote: 'E' },
  { stringNumber: 2, openNote: 'B' },
  { stringNumber: 3, openNote: 'G' },
  { stringNumber: 4, openNote: 'D' },
  { stringNumber: 5, openNote: 'A' },
  { stringNumber: 6, openNote: 'E' },
]
const OPEN_STRING_MIDI = {
  1: 64,
  2: 59,
  3: 55,
  4: 50,
  5: 45,
  6: 40,
}
const POSITION_MARKER_FRETS = [3, 5, 7, 9, 12]
const VISIBLE_FRET_COUNT = 13
const OPEN_AREA_RATIO = 0.08
const CORRECT_NEXT_DELAY = 900
const WRONG_NEXT_DELAY = 1400
const HARD_QUESTION_TIME_LIMIT = 5
const EASY_QUESTION_TIME_LIMIT = 10
const EASY_HINT_REVEAL_AFTER = 7
const BONUS_ROUND_INTERVAL = 5
const BONUS_ROUND_TIME_LIMIT = 8
const FRET_LIST = Array.from({ length: VISIBLE_FRET_COUNT }, (_, index) => index + 1)
const RESULT_MESSAGES = {
  idle: {
    headlines: ['준비 완료', '워밍업 대기 중', '시작하면 바로 감이 와요'],
    copies: [
      '아직 기록이 없어요. 가볍게 한 판 시작해서 오늘 지판 감각부터 깨워봐요.',
      '첫 판은 항상 탐색전이에요. 시작 버튼만 누르면 바로 손이 풀리기 시작할 거예요.',
      '지금은 몸 풀기 전이에요. 몇 문제만 지나도 어디가 헷갈리는지 금방 보일 거예요.',
    ],
  },
  excellent: {
    headlines: ['오늘 완전 감 잡았어요', '지판 감각이 제대로 올라왔어요', '이 정도면 정말 잘하고 있어요'],
    copies: [
      '눈으로 보고 바로 떠올리는 속도가 좋네요. 지금 페이스면 다음 판은 더 빠르게도 갈 수 있어요.',
      '정확도와 흐름이 둘 다 좋았어요. 손과 귀가 지판을 같이 기억하기 시작한 느낌이에요.',
      '이 정도면 외운 게 아니라 몸에 붙고 있는 단계예요. 조금만 더 하면 훨씬 편해질 거예요.',
    ],
  },
  good: {
    headlines: ['좋아요, 흐름이 올라오고 있어요', '이제 점점 손에 붙고 있어요', '지금 페이스 좋습니다'],
    copies: [
      '헷갈리는 자리 몇 군데만 다시 보면 다음 판에서 훨씬 또렷해질 거예요.',
      '좋은 흐름이에요. 지금 틀린 자리만 조금 더 보면 정확도가 금방 올라갈 거예요.',
      '이미 감각은 들어왔어요. 몇 번만 더 반복하면 지판이 훨씬 편하게 보일 거예요.',
    ],
  },
  warming: {
    headlines: ['조금씩 잡히고 있어요', '방향은 맞아요', '지금부터 더 재밌어집니다'],
    copies: [
      '틀린 자리만 다시 보면 다음 판이 확 달라질 거예요. 지금부터가 진짜 시작이에요.',
      '아직 섞여 보일 수 있지만 괜찮아요. 자주 나오는 자리부터 하나씩 손에 붙을 거예요.',
      '반응 속도보다 익숙해지는 과정이 더 중요해요. 한 판 더 가면 훨씬 정리될 거예요.',
    ],
  },
  retry: {
    headlines: ['괜찮아요, 지금부터예요', '처음엔 누구나 헷갈려요', '한 판 더 하면 달라져요'],
    copies: [
      '처음엔 헷갈리는 게 당연해요. 방금 틀린 자리부터 다시 보면 금방 감이 생길 거예요.',
      '틀리면서 외우는 속도가 생각보다 빠릅니다. 부담 없이 한 번 더 해보면 바로 달라질 수 있어요.',
      '지금은 외워가는 중이에요. 천천히 한 판 더 가면 지판이 훨씬 덜 낯설게 느껴질 거예요.',
    ],
  },
}
const PENTATONIC_POSITION_LAYOUTS = [
  {
    positionKey: 'C',
    positionLabel: 'C pos',
    color: '#8c67f3',
    minFret: 0,
    maxFret: 3,
  },
  {
    positionKey: 'A',
    positionLabel: 'A pos',
    color: '#4ba86c',
    minFret: 2,
    maxFret: 5,
  },
  {
    positionKey: 'G',
    positionLabel: 'G pos',
    color: '#e25757',
    minFret: 5,
    maxFret: 8,
  },
  {
    positionKey: 'E',
    positionLabel: 'E pos',
    color: '#e5c14b',
    minFret: 7,
    maxFret: 10,
  },
  {
    positionKey: 'D',
    positionLabel: 'D pos',
    color: '#4e8df0',
    minFret: 9,
    maxFret: 13,
  },
]

const noteIndexMap = Object.fromEntries(CHROMATIC_SCALE.map((note, index) => [note, index]))
const highestVisibleFretRatio = 1 - Math.pow(2, -VISIBLE_FRET_COUNT / 12)

const activeMode = ref('natural')
const difficultyMode = ref('easy')
const pentatonicType = ref('minor')
const selectedPentatonicKeys = reactive({
  minor: PENTATONIC_TYPE_CONFIG.minor.commonKeys[0],
  major: PENTATONIC_TYPE_CONFIG.major.commonKeys[0],
})
const selectedPosition = ref('C')
const currentQuestion = ref(null)
const selectedAnswer = ref('')
const isAnswered = ref(false)
const lastAnswerCorrect = ref(false)
const feedbackMessage = ref('')
const isRunning = ref(false)
const timeLeft = ref(EASY_QUESTION_TIME_LIMIT)
const bonusRound = ref(null)
const showResultModal = ref(false)
const lastSessionResult = ref(null)
const statsByMode = reactive({
  natural: createStats(),
  pentatonic: createStats(),
})

let autoNextTimer = null
let countdownTimer = null
let audioContext = null

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

function getStableMessageIndex(seed, length) {
  let hash = 0

  for (const character of seed) {
    hash = (hash * 31 + character.charCodeAt(0)) % 2147483647
  }

  return hash % length
}

function pickStableMessage(messages, seed) {
  return messages[getStableMessageIndex(seed, messages.length)]
}

function getResultTier(result) {
  if (result.totalAnswers === 0) {
    return 'idle'
  }

  if (result.accuracy >= 90) {
    return 'excellent'
  }

  if (result.accuracy >= 70) {
    return 'good'
  }

  if (result.accuracy >= 40) {
    return 'warming'
  }

  return 'retry'
}

function buildResultNarration(result) {
  const tier = getResultTier(result)
  const messages = RESULT_MESSAGES[tier]
  const seedBase = [
    result.mode,
    result.scaleLabel ?? 'natural',
    result.positionKey ?? 'none',
    result.totalAnswers,
    result.correctAnswers,
    result.bestStreak,
  ].join('-')

  return {
    headline: pickStableMessage(messages.headlines, `headline-${seedBase}`),
    copy: pickStableMessage(messages.copies, `copy-${seedBase}`),
  }
}

function getNoteAtFret(openNote, fret) {
  const startIndex = noteIndexMap[openNote]
  return CHROMATIC_SCALE[(startIndex + fret) % CHROMATIC_SCALE.length]
}

function getNoteFromInterval(rootNote, interval) {
  return CHROMATIC_SCALE[(noteIndexMap[rootNote] + interval) % CHROMATIC_SCALE.length]
}

function buildScaleNotes(rootNote, intervals) {
  return intervals.map((interval) => getNoteFromInterval(rootNote, interval))
}

function getAudioContext() {
  if (typeof window === 'undefined') {
    return null
  }

  const AudioContextClass = window.AudioContext || window.webkitAudioContext

  if (!AudioContextClass) {
    return null
  }

  if (!audioContext) {
    audioContext = new AudioContextClass()
  }

  return audioContext
}

function getCellFrequency(cell) {
  const midiNote = OPEN_STRING_MIDI[cell.stringNumber] + cell.fret
  return 440 * Math.pow(2, (midiNote - 69) / 12)
}

function playCellNote(cell) {
  const context = getAudioContext()

  if (!context || !cell) {
    return
  }

  const schedulePlayback = () => {
    const now = context.currentTime
    const frequency = getCellFrequency(cell)
    const oscillator = context.createOscillator()
    const overtoneOscillator = context.createOscillator()
    const filter = context.createBiquadFilter()
    const gainNode = context.createGain()

    oscillator.type = 'triangle'
    oscillator.frequency.setValueAtTime(frequency, now)

    overtoneOscillator.type = 'sine'
    overtoneOscillator.frequency.setValueAtTime(frequency * 2, now)
    overtoneOscillator.detune.setValueAtTime(4, now)

    filter.type = 'lowpass'
    filter.frequency.setValueAtTime(2400, now)
    filter.Q.setValueAtTime(0.6, now)

    gainNode.gain.setValueAtTime(0.0001, now)
    gainNode.gain.exponentialRampToValueAtTime(0.16, now + 0.02)
    gainNode.gain.exponentialRampToValueAtTime(0.05, now + 0.16)
    gainNode.gain.exponentialRampToValueAtTime(0.0001, now + 0.95)

    oscillator.connect(filter)
    overtoneOscillator.connect(filter)
    filter.connect(gainNode)
    gainNode.connect(context.destination)

    oscillator.start(now)
    overtoneOscillator.start(now)
    oscillator.stop(now + 1)
    overtoneOscillator.stop(now + 1)

    oscillator.onended = () => {
      oscillator.disconnect()
      overtoneOscillator.disconnect()
      filter.disconnect()
      gainNode.disconnect()
    }
  }

  if (context.state === 'suspended') {
    context.resume().then(schedulePlayback).catch(() => {})
    return
  }

  schedulePlayback()
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

const currentPentatonicConfig = computed(() => PENTATONIC_TYPE_CONFIG[pentatonicType.value])
const currentPentatonicKey = computed(() => selectedPentatonicKeys[pentatonicType.value])
const currentPentatonicScaleLabel = computed(
  () => `${currentPentatonicKey.value} ${currentPentatonicConfig.value.scaleSuffix}`,
)
const currentPentatonicScaleNotes = computed(() =>
  buildScaleNotes(currentPentatonicKey.value, currentPentatonicConfig.value.intervals),
)
const currentPentatonicTargets = computed(() =>
  currentPentatonicScaleNotes.value.map((note, index) => ({
    degreeLabel: currentPentatonicConfig.value.degreeLabels[index],
    note,
    value: `${currentPentatonicConfig.value.degreeLabels[index]}:${note}`,
  })),
)
const pentatonicTargetByNote = computed(
  () => new Map(currentPentatonicTargets.value.map((target) => [target.note, target])),
)
const commonPentatonicKeys = computed(() => currentPentatonicConfig.value.commonKeys)
const pentatonicPositions = computed(() =>
  PENTATONIC_POSITION_LAYOUTS.map((position) => {
    const cells = fretboard
      .flatMap((row) =>
        row.frets
          .filter(
            (cell) =>
              cell.fret >= position.minFret &&
              cell.fret <= position.maxFret &&
              currentPentatonicScaleNotes.value.includes(cell.note),
          )
          .map((cell) => {
            const target = pentatonicTargetByNote.value.get(cell.note)

            return {
              ...cell,
              degreeLabel: target?.degreeLabel ?? '',
              choiceValue: target?.value ?? cell.note,
              isRoot: cell.note === currentPentatonicKey.value,
            }
          }),
      )
      .sort((left, right) => {
        if (left.stringNumber === right.stringNumber) {
          return left.fret - right.fret
        }

        return left.stringNumber - right.stringNumber
      })

    return {
      ...position,
      cells,
    }
  }),
)

const currentStats = computed(() => statsByMode[activeMode.value])
const currentQuestionTimeLimit = computed(() =>
  difficultyMode.value === 'easy' ? EASY_QUESTION_TIME_LIMIT : HARD_QUESTION_TIME_LIMIT,
)
const selectedPentatonicPosition = computed(
  () =>
    pentatonicPositions.value.find((position) => position.positionKey === selectedPosition.value) ??
    pentatonicPositions.value[0],
)
const visiblePositionCells = computed(() =>
  activeMode.value === 'pentatonic' ? selectedPentatonicPosition.value?.cells ?? [] : [],
)
const visiblePositionZones = computed(() =>
  activeMode.value === 'pentatonic' && selectedPentatonicPosition.value
    ? [selectedPentatonicPosition.value]
    : [],
)
const visibleRootCells = computed(() => visiblePositionCells.value.filter((cell) => cell.isRoot))
const isBonusRoundActive = computed(() => Boolean(bonusRound.value))
const bonusProgressLabel = computed(() => {
  if (!bonusRound.value) {
    return ''
  }

  return `${bonusRound.value.foundKeys.length}/${bonusRound.value.rootKeys.length}`
})
const showQuestionNoteHint = computed(() => {
  if (
    difficultyMode.value !== 'easy' ||
    !currentQuestion.value ||
    isBonusRoundActive.value ||
    !isRunning.value ||
    isAnswered.value
  ) {
    return false
  }

  return timeLeft.value <= EASY_QUESTION_TIME_LIMIT - EASY_HINT_REVEAL_AFTER
})
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

  return lastSessionResult.value.headline
})

const resultModeLabel = computed(() => {
  if (!lastSessionResult.value) {
    return ''
  }

  return lastSessionResult.value.mode === 'natural'
    ? '자연음 퀴즈 결과'
    : `${lastSessionResult.value.scaleLabel} ${lastSessionResult.value.positionLabel} 도수/음이름 퀴즈 결과`
})

const resultCopy = computed(() => {
  if (!lastSessionResult.value) {
    return ''
  }

  return lastSessionResult.value.copy
})

function buildSessionResult() {
  const stats = statsByMode[activeMode.value]
  const totalAnswers = stats.totalAnswers
  const correctAnswers = stats.correctAnswers
  const result = {
    mode: activeMode.value,
    pentatonicType: pentatonicType.value,
    scaleLabel: currentPentatonicScaleLabel.value,
    positionKey: selectedPosition.value,
    positionLabel: selectedPentatonicPosition.value.positionLabel,
    totalAnswers,
    correctAnswers,
    bestStreak: stats.bestStreak,
    accuracy: totalAnswers ? Math.round((correctAnswers / totalAnswers) * 100) : 0,
  }
  const narration = buildResultNarration(result)

  return {
    ...result,
    ...narration,
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
  return shuffle([correctNote, ...distractors]).map((note) => ({
    value: note,
    note,
  }))
}

function buildPentatonicChoices(correctChoiceValue) {
  const correctTarget = currentPentatonicTargets.value.find((target) => target.value === correctChoiceValue)
  const distractors = shuffle(
    currentPentatonicTargets.value.filter((target) => target.value !== correctChoiceValue),
  ).slice(0, 3)

  return shuffle([correctTarget, ...distractors].filter(Boolean)).map((target) => ({
    value: target.value,
    degreeLabel: target.degreeLabel,
    note: target.note,
  }))
}

function getCurrentPool() {
  if (activeMode.value === 'natural') {
    return naturalQuizPool
  }

  return selectedPentatonicPosition.value?.cells ?? []
}

function isCorrectChoice(choice, question = currentQuestion.value) {
  if (!question) {
    return false
  }

  const choiceValue = choice?.value ?? choice

  return choiceValue === question.correctAnswer
}

function shouldTriggerBonusRound(stats) {
  return (
    activeMode.value === 'pentatonic' &&
    stats.correctAnswers > 0 &&
    stats.correctAnswers % BONUS_ROUND_INTERVAL === 0 &&
    visibleRootCells.value.length > 0
  )
}

function startBonusRound() {
  clearActiveTimers()

  bonusRound.value = {
    rootKeys: visibleRootCells.value.map((cell) => cell.key),
    foundKeys: [],
  }
  currentQuestion.value = null
  selectedAnswer.value = ''
  isAnswered.value = false
  lastAnswerCorrect.value = false
  feedbackMessage.value = `보너스입니다. ${selectedPentatonicPosition.value.positionLabel} 안의 루트 ${currentPentatonicKey.value}를 모두 찾아보세요.`
  timeLeft.value = BONUS_ROUND_TIME_LIMIT
  startCountdown()
}

function finishBonusRound(success, timedOut = false) {
  const foundCount = bonusRound.value?.foundKeys.length ?? 0
  const rootCount = bonusRound.value?.rootKeys.length ?? 0

  clearCountdownTimer()
  bonusRound.value = null
  currentQuestion.value = null
  selectedAnswer.value = ''
  isAnswered.value = true
  lastAnswerCorrect.value = success
  feedbackMessage.value = success
    ? `보너스 성공입니다. ${selectedPentatonicPosition.value.positionLabel}에서 루트 ${currentPentatonicKey.value}를 모두 찾았어요.`
    : `보너스 종료입니다. 루트 ${currentPentatonicKey.value}를 ${foundCount}/${rootCount} 찾았어요.`

  scheduleNextQuestion(success ? CORRECT_NEXT_DELAY : WRONG_NEXT_DELAY)
}

function handlePositionCellClick(cell) {
  if (!isBonusRoundActive.value || !bonusRound.value) {
    return
  }

  if (!cell.isRoot) {
    feedbackMessage.value = `여긴 ${cell.degreeLabel} / ${cell.note}예요. 루트 ${currentPentatonicKey.value}를 다시 찾아보세요.`
    return
  }

  if (bonusRound.value.foundKeys.includes(cell.key)) {
    return
  }

  const foundKeys = [...bonusRound.value.foundKeys, cell.key]
  bonusRound.value = {
    ...bonusRound.value,
    foundKeys,
  }

  playCellNote(cell)
  feedbackMessage.value = `좋아요. 루트 ${currentPentatonicKey.value}를 ${foundKeys.length}/${bonusRound.value.rootKeys.length} 찾았어요.`

  if (foundKeys.length === bonusRound.value.rootKeys.length) {
    finishBonusRound(true)
  }
}

function nextQuestion() {
  clearActiveTimers()
  bonusRound.value = null

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
      correctAnswer: nextCell.note,
      choices: buildNaturalChoices(nextCell.note),
    }
  } else {
    currentQuestion.value = {
      ...nextCell,
      quizType: 'pentatonic',
      scaleLabel: currentPentatonicScaleLabel.value,
      correctAnswer: nextCell.choiceValue,
      choices: buildPentatonicChoices(nextCell.choiceValue),
    }
  }

  selectedAnswer.value = ''
  isAnswered.value = false
  lastAnswerCorrect.value = false
  feedbackMessage.value = ''
  timeLeft.value = currentQuestionTimeLimit.value
  startCountdown()
}

function scheduleNextQuestion(delay) {
  clearActiveTimers()
  autoNextTimer = setTimeout(() => {
    nextQuestion()
  }, delay)
}

function answerQuestion(choice) {
  if (!isRunning.value || !currentQuestion.value || isAnswered.value || isBonusRoundActive.value) {
    return
  }

  clearCountdownTimer()

  const stats = currentStats.value
  const choiceValue = choice?.value ?? choice
  let isCorrect = false

  if (activeMode.value === 'natural') {
    isCorrect = choiceValue === currentQuestion.value.correctAnswer
    feedbackMessage.value = isCorrect
      ? `정답입니다. ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}은 ${currentQuestion.value.note}입니다.`
      : `오답입니다. 정답은 ${currentQuestion.value.note}이고, 위치는 ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}입니다.`
  } else {
    isCorrect = choiceValue === currentQuestion.value.correctAnswer
    feedbackMessage.value = isCorrect
      ? `정답입니다. ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}은 ${currentQuestion.value.degreeLabel} / ${currentQuestion.value.note}입니다.`
      : `오답입니다. 정답은 ${currentQuestion.value.degreeLabel} / ${currentQuestion.value.note}이고, 위치는 ${currentQuestion.value.stringNumber}번 줄 ${getPositionLabel(currentQuestion.value.fret)}입니다.`
  }

  selectedAnswer.value = choiceValue
  isAnswered.value = true
  lastAnswerCorrect.value = isCorrect
  stats.totalAnswers += 1

  if (isCorrect) {
    playCellNote(currentQuestion.value)
    stats.correctAnswers += 1
    stats.currentStreak += 1
    stats.bestStreak = Math.max(stats.bestStreak, stats.currentStreak)

    if (shouldTriggerBonusRound(stats)) {
      clearActiveTimers()
      autoNextTimer = setTimeout(() => {
        startBonusRound()
      }, CORRECT_NEXT_DELAY)
      return
    }

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
  if (!isRunning.value) {
    return
  }

  if (isBonusRoundActive.value) {
    finishBonusRound(false, true)
    return
  }

  if (!currentQuestion.value || isAnswered.value) {
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
    feedbackMessage.value = `시간 초과입니다. 정답은 ${currentQuestion.value.degreeLabel} / ${currentQuestion.value.note}입니다.`
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
  bonusRound.value = null
  currentQuestion.value = null
  selectedAnswer.value = ''
  isAnswered.value = false
  lastAnswerCorrect.value = false
  feedbackMessage.value = ''
  timeLeft.value = currentQuestionTimeLimit.value

  if (shouldShowResult) {
    lastSessionResult.value = buildSessionResult()
    showResultModal.value = true
  }
}

function startQuiz() {
  showResultModal.value = false
  lastSessionResult.value = null
  resetCurrentModeStats()
  bonusRound.value = null
  isRunning.value = true
  timeLeft.value = currentQuestionTimeLimit.value
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

function refreshPentatonicQuiz() {
  showResultModal.value = false
  lastSessionResult.value = null
  bonusRound.value = null

  if (activeMode.value === 'pentatonic' && isRunning.value) {
    startQuiz()
  }
}

function handlePentatonicModeClick() {
  if (activeMode.value === 'pentatonic') {
    pentatonicType.value = pentatonicType.value === 'minor' ? 'major' : 'minor'
    refreshPentatonicQuiz()
    return
  }

  switchMode('pentatonic')
}

function selectPentatonicKey(key) {
  if (selectedPentatonicKeys[pentatonicType.value] === key) {
    return
  }

  selectedPentatonicKeys[pentatonicType.value] = key
  refreshPentatonicQuiz()
}

function selectPosition(positionKey) {
  if (selectedPosition.value === positionKey) {
    return
  }

  selectedPosition.value = positionKey

  if (activeMode.value === 'pentatonic' && isRunning.value) {
    nextQuestion()
  }
}

function selectDifficulty(mode) {
  if (difficultyMode.value === mode) {
    return
  }

  difficultyMode.value = mode

  if (isRunning.value && currentQuestion.value && !isAnswered.value && !isBonusRoundActive.value) {
    timeLeft.value = currentQuestionTimeLimit.value
    startCountdown()
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

function getPositionZoneStyle(position) {
  const left = position.minFret === 0 ? 0 : getFretBoundaryPercent(position.minFret - 1)
  const right = getFretBoundaryPercent(position.maxFret)

  return {
    left: `${left}%`,
    width: `${right - left}%`,
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

function getPositionCellStyle(cell) {
  return {
    ...getQuestionMarkerStyle(cell),
    '--position-marker-color': selectedPentatonicPosition.value.color,
  }
}

function isFoundBonusRoot(cell) {
  return bonusRound.value?.foundKeys.includes(cell.key) ?? false
}

onMounted(() => {
  stopQuiz(false)
})

onBeforeUnmount(() => {
  clearActiveTimers()

  if (audioContext && audioContext.state !== 'closed') {
    audioContext.close()
    audioContext = null
  }
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
              @click="handlePentatonicModeClick"
            >
              {{ currentPentatonicConfig.buttonLabel }}
            </button>
          </div>

          <span class="board-title">사랑하는 우주 하늘을 위한 기타지판퀴즈</span>

          <div class="difficulty-toggle">
            <button
              type="button"
              class="difficulty-button"
              :class="{ active: difficultyMode === 'easy' }"
              @click="selectDifficulty('easy')"
            >
              Easy
            </button>
            <button
              type="button"
              class="difficulty-button"
              :class="{ active: difficultyMode === 'hard' }"
              @click="selectDifficulty('hard')"
            >
              Hard
            </button>
          </div>

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
        <div class="box-toolbar-group">
          <span class="box-toolbar-label">Pos</span>
          <div class="box-toolbar-buttons">
            <button
              v-for="position in pentatonicPositions"
              :key="position.positionKey"
              type="button"
              class="box-button"
              :class="{ active: selectedPosition === position.positionKey }"
              :style="{
                '--position-color': position.color,
                '--position-soft': `${position.color}22`,
              }"
              @click="selectPosition(position.positionKey)"
            >
              {{ position.positionLabel }}
            </button>
          </div>
        </div>

        <div class="scale-toolbar">
          <span class="box-toolbar-label">Key</span>
          <div class="scale-toolbar-buttons">
            <button
              v-for="key in commonPentatonicKeys"
              :key="`${pentatonicType}-${key}`"
              type="button"
              class="scale-button"
              :class="{ active: currentPentatonicKey === key }"
              @click="selectPentatonicKey(key)"
            >
              {{ key }}
            </button>
          </div>
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
                <div
                  v-for="position in visiblePositionZones"
                  :key="`zone-${position.positionKey}`"
                  class="position-zone"
                  :class="{ active: selectedPosition === position.positionKey }"
                  :style="{
                    ...getPositionZoneStyle(position),
                    '--position-color': position.color,
                    '--position-soft': `${position.color}22`,
                  }"
                ></div>

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

                <button
                  v-for="cell in visiblePositionCells"
                  :key="`position-cell-${cell.key}`"
                  type="button"
                  class="box-note-marker"
                  :class="{
                    'is-bonus-active': isBonusRoundActive,
                    'is-found': isFoundBonusRoot(cell),
                    'is-root-note': activeMode === 'pentatonic' && !isBonusRoundActive && cell.isRoot,
                  }"
                  :style="getPositionCellStyle(cell)"
                  :disabled="!isBonusRoundActive"
                  @click="handlePositionCellClick(cell)"
                ></button>

                <div
                  v-if="currentQuestion && !isBonusRoundActive"
                  class="question-marker"
                  :class="{
                    correct: isAnswered && lastAnswerCorrect,
                    wrong: isAnswered && !lastAnswerCorrect,
                    'with-note': showQuestionNoteHint,
                  }"
                  :style="getQuestionMarkerStyle(currentQuestion)"
                >
                  <span v-if="showQuestionNoteHint" class="question-marker-note">
                    {{ currentQuestion.note }}
                  </span>
                </div>
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
        v-if="isBonusRoundActive"
        class="bonus-box"
      >
        <strong>보너스 라운드</strong>
        <span>{{ selectedPentatonicPosition.positionLabel }} 안의 루트 {{ currentPentatonicKey }}를 모두 찾아보세요.</span>
        <span class="bonus-progress">{{ bonusProgressLabel }}</span>
        <span v-if="feedbackMessage" class="bonus-hint">{{ feedbackMessage }}</span>
      </div>

      <div
        v-else-if="currentQuestion && isRunning"
        class="choices-grid"
      >
        <button
          v-for="(choice, index) in currentQuestion.choices"
          :key="choice.value"
          type="button"
          class="choice-button"
          :class="{
            'is-correct': isAnswered && isCorrectChoice(choice),
            'is-wrong': isAnswered && choice.value === selectedAnswer && !isCorrectChoice(choice),
            'is-muted': isAnswered && choice.value !== selectedAnswer && !isCorrectChoice(choice),
          }"
          :disabled="isAnswered"
          @click="answerQuestion(choice)"
        >
          <span class="choice-index">{{ index + 1 }}</span>
          <span
            class="choice-note"
            :class="{ 'is-pentatonic': activeMode === 'pentatonic' }"
          >
            <template v-if="activeMode === 'pentatonic'">
              <span class="choice-degree">{{ choice.degreeLabel }}</span>
              <span class="choice-subnote">{{ choice.note }}</span>
            </template>
            <template v-else>
              {{ choice.note }}
            </template>
          </span>
        </button>
      </div>

      <div
        v-if="(isAnswered && !lastAnswerCorrect) || (!currentQuestion && !isBonusRoundActive && feedbackMessage)"
        class="feedback-box"
        :class="{ error: !lastAnswerCorrect, success: lastAnswerCorrect && !currentQuestion }"
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
  <footer>
  <p>© 2026 달빛요정. All rights reserved.</p>
  <a href="/privacy.html" target="_blank" style="font-size: 0.8rem; color: #888;">개인정보처리방침</a>
</footer>
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

.difficulty-toggle {
  display: inline-flex;
  gap: 4px;
  padding: 3px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(129, 88, 47, 0.12);
}

.difficulty-button {
  min-height: 22px;
  padding: 0 8px;
  border: 0;
  border-radius: 999px;
  background: transparent;
  color: #6a5038;
  font: inherit;
  font-size: 0.68rem;
  font-weight: 700;
  cursor: pointer;
}

.difficulty-button.active {
  color: #fff9f2;
  background: linear-gradient(135deg, #7b583b, #b77b35);
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
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 8px 14px;
  margin-bottom: 10px;
}

.box-toolbar-group,
.scale-toolbar {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}

.scale-toolbar {
  margin-left: auto;
  justify-content: flex-end;
}

.box-toolbar-label {
  font-size: 0.76rem;
  font-weight: 700;
  color: #6a5038;
}

.box-toolbar-buttons,
.scale-toolbar-buttons {
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
  color: #2b1d13;
  background: var(--position-soft);
  border-color: var(--position-color);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.45);
}

.scale-button {
  min-width: 32px;
  height: 30px;
  padding: 0 10px;
  border: 1px solid rgba(121, 88, 56, 0.12);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.82);
  color: #5f4735;
  font: inherit;
  font-size: 0.74rem;
  font-weight: 700;
  cursor: pointer;
}

.scale-button.active {
  color: #fff8f1;
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
.position-zone,
.position-marker,
.question-marker,
.box-note-marker {
  position: absolute;
}

.position-zone {
  top: 0;
  bottom: 0;
  border-radius: 12px;
  background: color-mix(in srgb, var(--position-soft) 86%, white);
  border: 2px solid color-mix(in srgb, var(--position-color) 72%, white);
  opacity: 0.55;
}

.position-zone.active {
  opacity: 0.82;
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
  padding: 0;
  border: 0;
  width: 12px;
  height: 12px;
  border-radius: 999px;
  transform: translate(-50%, -50%);
  background: var(--position-marker-color, #8c67f3);
  appearance: none;
  box-shadow:
    0 0 0 2px rgba(255, 255, 255, 0.86),
    0 3px 7px rgba(55, 38, 24, 0.22);
}

.box-note-marker.is-bonus-active {
  cursor: pointer;
}

.box-note-marker.is-bonus-active:hover:enabled {
  transform: translate(-50%, -50%) scale(1.1);
}

.box-note-marker.is-found {
  background: linear-gradient(135deg, #66d48b, #2f9d61);
}

.box-note-marker.is-root-note {
  width: 13px;
  height: 13px;
  background: radial-gradient(circle at 35% 35%, #6f6962 0 20%, #4a453f 45%, #2a2622 100%);
  box-shadow:
    0 0 0 2px rgba(255, 255, 255, 0.88),
    0 3px 8px rgba(31, 22, 14, 0.24);
}

.box-note-marker:disabled {
  opacity: 1;
}

.question-marker {
  width: 15px;
  height: 15px;
  display: grid;
  place-items: center;
  border-radius: 999px;
  transform: translate(-50%, -50%);
  background: radial-gradient(circle at 30% 30%, #ff9e93, #ff4035 65%, #da251e);
  box-shadow:
    0 0 0 2px rgba(255, 255, 255, 0.72),
    0 4px 8px rgba(189, 62, 46, 0.2);
}

.question-marker.with-note {
  width: 28px;
  height: 28px;
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

.question-marker-note {
  color: #fffaf4;
  font-size: 0.58rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  text-shadow: 0 1px 2px rgba(71, 19, 15, 0.3);
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

.bonus-box {
  display: flex;
  align-items: center;
  gap: 12px;
  min-height: 44px;
  padding: 8px 10px;
  border-radius: 12px;
  background: linear-gradient(135deg, rgba(255, 243, 220, 0.88), rgba(255, 250, 240, 0.94));
  border: 1px solid rgba(188, 135, 63, 0.18);
  color: #5b442f;
  font-size: 0.78rem;
}

.bonus-box strong {
  color: #2d1d11;
  white-space: nowrap;
}

.bonus-progress {
  margin-left: auto;
  font-weight: 800;
  color: #2d1d11;
}

.bonus-hint {
  margin-left: 4px;
  color: #7a5836;
}

.choices-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 6px;
  margin-top: 0;
}

.choice-button {
  min-height: 46px;
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
.difficulty-button:focus-visible,
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

.choice-note.is-pentatonic {
  display: flex;
  flex-direction: column;
  align-items: center;
  line-height: 1;
  gap: 2px;
}

.choice-degree {
  font-size: 0.82rem;
}

.choice-subnote {
  font-size: 0.72rem;
  color: #75583d;
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

.feedback-box.success {
  background: #edf8ef;
  color: #2d6b3c;
  border-color: rgba(79, 164, 103, 0.18);
}

.result-modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 20;
  display: grid;
  place-items: center;
  padding: 16px;
  background: rgba(38, 25, 16, 0.38);
  backdrop-filter: blur(6px);
}

.result-modal {
  width: min(410px, 100%);
  padding: 18px;
  border-radius: 22px;
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
  font-size: 0.66rem;
  letter-spacing: 0.15em;
  text-transform: uppercase;
}

.result-title {
  margin: 6px 0 0;
  color: #2b1d13;
  font-size: clamp(1.2rem, 2.2vw, 1.7rem);
  line-height: 1.08;
}

.result-copy {
  margin: 8px 0 0;
  color: #6a5038;
  font-size: 0.82rem;
  line-height: 1.42;
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
  margin-top: 14px;
}

.result-card {
  padding: 11px 12px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(121, 88, 56, 0.1);
}

.result-card span {
  display: block;
  color: #876646;
  font-size: 0.7rem;
}

.result-card strong {
  display: block;
  margin-top: 4px;
  color: #24170f;
  font-size: 1.16rem;
}

.result-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 14px;
}

.result-button {
  min-height: 34px;
  padding: 0 14px;
  border: 0;
  border-radius: 999px;
  font: inherit;
  font-size: 0.76rem;
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

  .difficulty-toggle {
    border-radius: 14px;
  }

  .board-topbar {
    align-items: flex-start;
  }

  .box-toolbar {
    align-items: flex-start;
  }

  .scale-toolbar {
    margin-left: 0;
    width: 100%;
    justify-content: flex-end;
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

  .difficulty-button {
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
    min-height: 38px;
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

  .bonus-box {
    flex-wrap: wrap;
    gap: 6px 10px;
    min-height: 0;
    font-size: 0.7rem;
  }

  .bonus-progress {
    margin-left: 0;
  }

  .box-note-marker {
    width: 10px;
    height: 10px;
  }

  .question-marker {
    width: 13px;
    height: 13px;
  }

  .question-marker.with-note {
    width: 24px;
    height: 24px;
  }

  .question-marker-note {
    font-size: 0.52rem;
  }

  .stats-inline {
    gap: 6px;
    font-size: 0.64rem;
    margin-left: 0;
    width: 100%;
    justify-content: flex-end;
  }

  .result-modal {
    padding: 16px;
    border-radius: 18px;
  }

  .result-grid {
    gap: 7px;
  }

  .result-card {
    padding: 10px;
    border-radius: 14px;
  }
}
</style>

