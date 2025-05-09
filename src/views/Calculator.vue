<script setup>
defineOptions({
  name: 'MyCalculator'
});
import { ref, computed, onMounted, onBeforeUnmount } from 'vue';

const current = ref('0');
const previous = ref(null);
const operator = ref(null);
const operatorClicked = ref(false);
const displayFull = ref(false);
const maxLength = ref(20);
const calculated = ref(false);
const expression = ref('');
const history = ref([]);
const showHistory = ref(false);

const displayedPrefix = computed(() => {
  const numberString = current.value;
  const parts = numberString.split('.');
  const integerPart = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  const decimalPart = numberString.includes('.') ? '.' + (parts[1] ?? '') : '';
  return integerPart + decimalPart;
});

const displayFontSize = computed(() => {
  if (current.value.length > 10) {
    const baseSize = 24;
    const reductionFactor = 0.5;
    return Math.max(16, baseSize - (current.value.length - 10) * reductionFactor) + 'px';
  }
  return '24px';
});

function clear() {
  current.value = '0';
  previous.value = null;
  operator.value = null;
  operatorClicked.value = false;
  displayFull.value = false;
  calculated.value = false;
  expression.value = '';
}

function append(number) {
  if (calculated.value) {
    current.value = '';
    calculated.value = false;
    displayFull.value = false;
    expression.value = '';
  }
  if (operatorClicked.value) {
    current.value = '';
    operatorClicked.value = false;
    displayFull.value = false;
  }

  if (current.value === '0' && number !== '.') {
    current.value = number;
  } else if (current.value.length < maxLength.value) {
    current.value = `${current.value}${number}`;
  }

  expression.value += number;
}

function setOperator(newOperator) {
  if (current.value === 'Error') return;

  if (calculated.value) {
    calculated.value = false;
    expression.value = current.value;
  }

  if (/[+\-*/]$/.test(expression.value.trim())) {
    expression.value = expression.value.trim().slice(0, -1);
  }

  try {
    const sanitized = expression.value.replace(/[^-()\d/*+.]/g, '');
    const result = roundResult(eval(sanitized));
    current.value = result.toString();
  } catch (e) {
    console.log(e);
  }

  operator.value = newOperator;
  operatorClicked.value = true;
  expression.value += ` ${newOperator} `;
}

function del() {
  current.value = current.value.slice(0, -1);
  expression.value = expression.value.slice(0, -1);
  if (current.value.length === 0) {
    current.value = '0';
    displayFull.value = false;
  } else if (current.value.length <= maxLength.value) {
    displayFull.value = false;
  }
  calculated.value = false;
}

function formatNumberWithCommas(number) {
  const numberString = String(number);
  const parts = numberString.split('.');
  const integerPart = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  const decimalPart = parts[1] ? `.${parts[1]}` : '';
  return integerPart + decimalPart;
}

function roundResult(num, decimals = 10) {
  return parseFloat(num.toFixed(decimals));
}

function calculate() {
  try {
    if (expression.value.trim() !== '') {
      const sanitized = expression.value.replace(/[^-()\d/*+.]/g, '');
      const result = roundResult(eval(sanitized));
      const formattedResult = formatNumberWithCommas(result);
      history.value.push(`${expression.value} = ${formattedResult}`);
      current.value = result.toString();
      previous.value = null;
      operator.value = null;
      calculated.value = true;
      expression.value = '';
      displayFull.value = false;
    }
  } catch (e) {
    history.value.push(`${expression.value} = Error`);
    current.value = 'Error';
    expression.value = '';
    calculated.value = false;
    displayFull.value = false;
    console.log(e);
  }
}

function handleKeyPress(event) {
  const key = event.key;

  if (/[0-9]/.test(key)) {
    append(key);
  } else if (['+', '-', '*', '/'].includes(key)) {
    setOperator(key);
  } else if (key === 'Enter' || key === '=') {
    calculate();
  } else if (key === 'Backspace') {
    del();
  } else if (key === 'Escape') {
    clear();
  } else if (key === '(' || key === ')') {
    append(key);
    expression.value = `${expression.value}${key}`;
  } else if (key === '.') {
    appendDecimal();
  }
}
function appendDecimal() {
  if (calculated.value || operatorClicked.value) {
    current.value = '0.';
    calculated.value = false;
    operatorClicked.value = false;
    displayFull.value = false;
    expression.value = '0.';
  } else if (!current.value.includes('.')) {
    if (current.value === '0') {
      current.value = '0.';
      expression.value = '0.';
    } else {
      current.value = `${current.value}.`;
      expression.value = `${expression.value}.`;
    }
  }
}

function appendParenthesis(p) {
  append(p);

  if (p === ')') {
    try {
      const sanitized = expression.value.replace(/[^-()\d/*+.]/g, '');
      const result = roundResult(eval(sanitized));
      current.value = result.toString();
    } catch (e) {
      console.log(e)
    }
  }
}

function handleHistoryClick(record) {
  expression.value = record.split(' =')[0];
  current.value = '0';
  previous.value = null;
  operator.value = null;
  operatorClicked.value = false;
  calculated.value = false;
}

function toggleHistory() {
  showHistory.value = !showHistory.value;
}

function clearHistory() {
  history.value = [];
}

onMounted(() => {
  window.addEventListener('keydown', handleKeyPress);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyPress);
});
</script>

<template>
  <div class="calculator">
    <div class="display">
      <div class="expression-display">{{ expression }}</div>
      <div :style="{ fontSize: displayFontSize }">{{ displayedPrefix }}</div>
    </div>
    <div class="buttons">
      <button @click="clear">C</button>
      <button @click="appendParenthesis('(')">(</button>
      <button @click="appendParenthesis(')')">)</button>
      <button @click="del">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="currentColor" class="bi bi-backspace" viewBox="0 0 16 16">
        <path d="M5.83 5.146a.5.5 0 0 0 0 .708L7.975 8l-2.147 2.146a.5.5 0 0 0 .707.708l2.147-2.147 2.146 2.147a.5.5 0 0 0 .707-.708L9.39 8l2.146-2.146a.5.5 0 0 0-.707-.708L8.683 7.293 6.536 5.146a.5.5 0 0 0-.707 0z"/>
        <path d="M13.683 1a2 2 0 0 1 2 2v10a2 2 0 0 1-2 2h-7.08a2 2 0 0 1-1.519-.698L.241 8.65a1 1 0 0 1 0-1.302L5.084 1.7A2 2 0 0 1 6.603 1zm-7.08 1a1 1 0 0 0-.76.35L1 8l4.844 5.65a1 1 0 0 0 .759.35h7.08a1 1 0 0 0 1-1V3a1 1 0 0 0-1-1z"/>
        </svg>
      </button>

      <button @click="append('7')">7</button>
      <button @click="append('8')">8</button>
      <button @click="append('9')">9</button>
      <button @click="setOperator('/')">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="#F57878" class="bi bi-percent" viewBox="0 0 16 16">
        <path d="M13.442 2.558a.625.625 0 0 1 0 .884l-10 10a.625.625 0 1 1-.884-.884l10-10a.625.625 0 0 1 .884 0M4.5 6a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3m0 1a2.5 2.5 0 1 0 0-5 2.5 2.5 0 0 0 0 5m7 6a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3m0 1a2.5 2.5 0 1 0 0-5 2.5 2.5 0 0 0 0 5"/>
        </svg>
      </button>

      <button @click="append('4')">4</button>
      <button @click="append('5')">5</button>
      <button @click="append('6')">6</button>
      <button @click="setOperator('*')">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="#F57878" class="bi bi-x-lg" viewBox="0 0 16 16">
        <path d="M2.146 2.854a.5.5 0 1 1 .708-.708L8 7.293l5.146-5.147a.5.5 0 0 1 .708.708L8.707 8l5.147 5.146a.5.5 0 0 1-.708.708L8 8.707l-5.146 5.147a.5.5 0 0 1-.708-.708L7.293 8z"/>
        </svg>
      </button>

      <button @click="append('1')">1</button>
      <button @click="append('2')">2</button>
      <button @click="append('3')">3</button>
      <button @click="setOperator('-')">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="#F57878" class="bi bi-dash-lg" viewBox="0 0 16 16">
        <path fill-rule="evenodd" d="M2 8a.5.5 0 0 1 .5-.5h11a.5.5 0 0 1 0 1h-11A.5.5 0 0 1 2 8"/>
        </svg>
      </button>

      <button @click="appendDecimal()">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="currentColor" class="bi bi-dot" viewBox="0 0 16 16">
        <path d="M8 9.5a1.5 1.5 0 1 0 0-3 1.5 1.5 0 0 0 0 3"/>
        </svg>
      </button>
      <button @click="append('0')">0</button>
      <button @click="calculate" class="equals">=</button>
      <button @click="setOperator('+')">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="#F57878" class="bi bi-plus-lg" viewBox="0 0 16 16">
        <path fill-rule="evenodd" d="M8 2a.5.5 0 0 1 .5.5v5h5a.5.5 0 0 1 0 1h-5v5a.5.5 0 0 1-1 0v-5h-5a.5.5 0 0 1 0-1h5v-5A.5.5 0 0 1 8 2"/>
        </svg>
      </button>

      <button @click="toggleHistory" class="history-button">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="lightblue" class="bi bi-chat-right-dots" viewBox="0 0 16 16">
        <path d="M2 1a1 1 0 0 0-1 1v8a1 1 0 0 0 1 1h9.586a2 2 0 0 1 1.414.586l2 2V2a1 1 0 0 0-1-1zm12-1a2 2 0 0 1 2 2v12.793a.5.5 0 0 1-.854.353l-2.853-2.853a1 1 0 0 0-.707-.293H2a2 2 0 0 1-2-2V2a2 2 0 0 1 2-2z"/>
        <path d="M5 6a1 1 0 1 1-2 0 1 1 0 0 1 2 0m4 0a1 1 0 1 1-2 0 1 1 0 0 1 2 0m4 0a1 1 0 1 1-2 0 1 1 0 0 1 2 0"/>
        </svg>
        History</button>
    </div>

    <div v-if="showHistory" class="history-panel">
      <h2>계산 기록</h2>
      <ul v-if="history.length > 0">
        <li v-for="(item, index) in history" :key="index" @click="handleHistoryClick(item)">{{ item }}</li>
      </ul>
      <p v-else>계산 기록이 없습니다.</p>
      <div class="button-row">
        <button @click="clearHistory">지우기</button>
        <button @click="showHistory = false">닫기</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.calculator {
  width: 300px;
  margin: 50px auto;
  background: #f4f4f4;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
  position: relative;
}

.display {
  background: #fff;
  color: black;
  padding: 15px;
  font-size: 24px;
  text-align: right;
  margin-bottom: 20px;
  border-radius: 5px;
  box-shadow: inset 0 0 5px rgba(0,0,0,0.1);
  min-height: 90px;
  word-wrap: break-word;
  word-break: break-all;
  overflow-wrap: break-word;
  max-width: 100%;
  cursor: pointer;
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.expression-display {
  color: #777;
  font-size: 16px;
  text-align: right;
  overflow: hidden;
  white-space: nowrap;
  max-width: 100%;
}

.buttons {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
}

button {
  padding: 15px;
  font-size: 18px;
  border: none;
  background: #fff;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.2s;
}

button:hover {
  background: #DCEBFF;
}

button:active{
  background: #C8D7FF;
}

.equals {
  background: #68D168;
  color: white;
}

.equals:hover {
  background: #45a049;
}

.equals:active {
  background: #288C28;
}

.history-button {
  grid-column: 1 / 5;
}

.ellipsis {
  color: #777;
}

.display.full .ellipsis {
  display: none;
}

.history-panel {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  color: black;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.history-panel h2 {
  color: white;
  margin-bottom: 20px;
}

.history-panel ul {
  background: white;
  border-radius: 5px;
  padding: 20px;
  max-height: 300px;
  overflow-y: auto;
  margin-bottom: 20px;
  width: 80%;
  box-shadow: 0 0 10px rgba(0,0,0,0.2);
}

.history-panel li {
  padding: 8px 0;
  border-bottom: 1px solid #eee;
}

.history-panel li:last-child {
  border-bottom: none;
}

.history-panel p {
  color: white;
}

.history-panel button {
  padding: 10px 20px;
  font-size: 12px;
  border: none;
  background: #007bff;
  color: white;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.2s;
}

.button-row {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  /* width: 80%; */
}

.history-panel button:hover {
  background: #0056b3;
}
</style>