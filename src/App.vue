<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'

type Person = {
  name: string
}

type Expense = {
  payer: string
  amount: number
  note: string
}

const people = ref<Person[]>([])
const expenses = ref<Expense[]>([])
const newPersonName = ref('')
const newExpensePayer = ref('')
const newExpenseAmount = ref<number | null>(null)
const newExpenseNote = ref('')
const isHydrated = ref(false)

const currency = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'INR',
  maximumFractionDigits: 2,
})

const trimmedPeople = computed(() =>
  people.value
    .map((person) => person.name.trim())
    .filter((name) => name.length > 0)
)

const uniquePeople = computed(() => {
  const unique: string[] = []
  const seen = new Set<string>()
  for(const name of trimmedPeople.value) {
    const key = name.toLowerCase()
    if(!seen.has(key)) {
      seen.add(key)
      unique.push(name)
    }
  }
  return unique
})

const duplicateNames = computed(() => {
  const nameCounts = new Map<string, number>()
  for(const name of trimmedPeople.value) {
    const key = name.toLowerCase()
    nameCounts.set(key, (nameCounts.get(key) ?? 0) + 1)
  }

  return Array.from(nameCounts.entries())
    .filter(([, count]) => count > 1)
    .map(([name]) => name)
})

const normalizedExpenses = computed(() =>
  expenses.value.map((expense) => {
    const amount = Number.isFinite(expense.amount) ? Math.max(0, expense.amount) : 0
    return {
      payer: expense.payer.trim(),
      amount,
      note: expense.note.trim(),
    }
  })
)

const totalSpent = computed(() =>
  normalizedExpenses.value.reduce((sum, expense) => sum + expense.amount, 0)
)

const perPersonShare = computed(() => {
  if(uniquePeople.value.length === 0 || totalSpent.value <= 0) {
    return 0
  }
  return totalSpent.value / uniquePeople.value.length
})

const paidByName = computed(() => {
  const map = new Map<string, number>()
  for(const expense of normalizedExpenses.value) {
    if(!expense.payer) {
      continue
    }
    const key = expense.payer.toLowerCase()
    map.set(key, (map.get(key) ?? 0) + expense.amount)
  }
  return map
})

const balances = computed(() =>
  uniquePeople.value.map((name) => {
    const key = name.toLowerCase()
    const paid = paidByName.value.get(key) ?? 0
    const balance = paid - perPersonShare.value
    return { name, paid, balance }
  })
)

const unknownPayers = computed(() => {
  const peopleKeys = new Set(uniquePeople.value.map((name) => name.toLowerCase()))
  const payers = new Set(
    normalizedExpenses.value
      .map((expense) => expense.payer)
      .filter((payer) => payer.length > 0)
      .map((payer) => payer.toLowerCase())
  )

  return Array.from(payers).filter((payer) => !peopleKeys.has(payer))
})

const shareableUrl = computed(() => {
  const url = new URL(window.location.href)
  const params = new URLSearchParams()

  if(uniquePeople.value.length > 0) {
    params.set('people', JSON.stringify(uniquePeople.value))
  }

  const filteredExpenses = normalizedExpenses.value.filter(
    (expense) => expense.payer.length > 0 && expense.amount > 0
  )

  if(filteredExpenses.length > 0) {
    params.set('expenses', JSON.stringify(filteredExpenses))
  }

  const query = params.toString()
  url.search = query
  return url.toString()
})

const canShare = ref(false)
const canCopy = ref(false)
const shareStatus = ref('')
const shareInput = ref<HTMLInputElement | null>(null)
const storageKey = 'expense-splitter-state-v1'

function addPerson() {
  const name = newPersonName.value.trim()
  if(!name) {
    return
  }
  people.value.push({ name })
  newPersonName.value = ''
}

function removePerson(index: number) {
  people.value.splice(index, 1)
}

function addExpense() {
  const payer = newExpensePayer.value.trim()
  const amount = newExpenseAmount.value ?? 0
  if(!payer || amount <= 0) {
    return
  }
  expenses.value.push({
    payer,
    amount,
    note: newExpenseNote.value.trim(),
  })
  newExpensePayer.value = ''
  newExpenseAmount.value = null
  newExpenseNote.value = ''
}

function removeExpense(index: number) {
  expenses.value.splice(index, 1)
}

function loadFromUrl() {
  const params = new URLSearchParams(window.location.search)
  let hasUrlData = false

  const peopleParam = params.get('people')
  if(peopleParam) {
    try {
      const parsed = JSON.parse(peopleParam)
      if(Array.isArray(parsed)) {
        people.value = parsed
          .filter((entry) => typeof entry === 'string')
          .map((name) => ({ name }))
        hasUrlData = true
      }
    } catch {
      people.value = []
    }
  }

  const expensesParam = params.get('expenses')
  if(expensesParam) {
    try {
      const parsed = JSON.parse(expensesParam)
      if(Array.isArray(parsed)) {
        expenses.value = parsed
          .filter((entry) => typeof entry === 'object' && entry)
          .map((entry) => ({
            payer: typeof entry.payer === 'string' ? entry.payer : '',
            amount: typeof entry.amount === 'number' ? entry.amount : 0,
            note: typeof entry.note === 'string' ? entry.note : '',
          }))
        hasUrlData = true
      }
    } catch {
      expenses.value = []
    }
  }

  if(!hasUrlData) {
    loadFromStorage()
  }
}

function loadFromStorage() {
  if(typeof localStorage === 'undefined') {
    return
  }

  const raw = localStorage.getItem(storageKey)
  if(!raw) {
    return
  }

  try {
    const parsed = JSON.parse(raw)
    if(Array.isArray(parsed?.people)) {
      people.value = parsed.people
        .filter((entry: unknown) => typeof entry === 'string')
        .map((name: string) => ({ name }))
    }

    if(Array.isArray(parsed?.expenses)) {
      expenses.value = parsed.expenses
        .filter((entry: unknown) => typeof entry === 'object' && entry)
        .map((entry: any) => ({
          payer: typeof entry.payer === 'string' ? entry.payer : '',
          amount: typeof entry.amount === 'number' ? entry.amount : 0,
          note: typeof entry.note === 'string' ? entry.note : '',
        }))
    }
  } catch {
    // Ignore invalid storage data.
  }
}

function saveToStorage() {
  if(typeof localStorage === 'undefined') {
    return
  }

  const payload = {
    people: uniquePeople.value,
    expenses: normalizedExpenses.value,
  }

  try {
    localStorage.setItem(storageKey, JSON.stringify(payload))
  } catch {
    // Ignore storage errors.
  }
}

function syncPeopleFromExpenses() {
  const known = new Set(people.value.map((person) => person.name.trim().toLowerCase()))
  let changed = false

  for(const expense of expenses.value) {
    const payer = expense.payer.trim()
    if(!payer) {
      continue
    }
    const key = payer.toLowerCase()
    if(!known.has(key)) {
      known.add(key)
      people.value.push({ name: payer })
      changed = true
    }
  }

  if(changed) {
    saveToStorage()
  }

  return changed
}

function syncUrl() {
  if(!isHydrated.value) {
    return
  }

  const params = new URLSearchParams()
  if(uniquePeople.value.length > 0) {
    params.set('people', JSON.stringify(uniquePeople.value))
  }

  const filteredExpenses = normalizedExpenses.value.filter(
    (expense) => expense.payer.length > 0 && expense.amount > 0
  )

  if(filteredExpenses.length > 0) {
    params.set('expenses', JSON.stringify(filteredExpenses))
  }

  const query = params.toString()
  const nextUrl = query.length ? `?${query}` : window.location.pathname
  window.history.replaceState({}, '', nextUrl)
}

async function copyShareUrl() {
  shareStatus.value = ''

  try {
    if(typeof navigator !== 'undefined' && navigator.clipboard?.writeText) {
      await navigator.clipboard.writeText(shareableUrl.value)
      shareStatus.value = 'Copied to clipboard.'
      return
    }
  } catch {
    // Fall through to manual copy.
  }

  if(shareInput.value) {
    shareInput.value.focus()
    shareInput.value.select()
    const success = document.execCommand('copy')
    shareStatus.value = success ? 'Copied to clipboard.' : 'Copy failed.'
  } else {
    shareStatus.value = 'Copy failed.'
  }
}

async function shareSplit() {
  try {
    if(canShare.value) {
      await navigator.share({
        title: 'Group expense split',
        text: 'Here is the shared expense split.',
        url: shareableUrl.value,
      })
    } else if(canCopy.value) {
      await copyShareUrl()
    } else {
      shareStatus.value = 'Sharing is not supported.'
    }
  } catch {
    // Ignore cancellation or share errors.
    if(canCopy.value) {
      await copyShareUrl()
    }
  }
}

onMounted(() => {
  loadFromUrl()
  syncPeopleFromExpenses()
  isHydrated.value = true
  canShare.value = typeof navigator !== 'undefined' && typeof navigator.share === 'function'
  canCopy.value =
    typeof navigator !== 'undefined' &&
    (typeof navigator.clipboard?.writeText === 'function' ||
      (typeof document !== 'undefined' && typeof document.execCommand === 'function'))
  saveToStorage()
})

watch(expenses, () => {
  syncPeopleFromExpenses()
}, { deep: true })

watch([people, expenses], () => {
  syncUrl()
  saveToStorage()
}, { deep: true, flush: 'post' })
</script>

<template>
  <div class="page">
    <header class="hero">
      <div>
        <p class="eyebrow">Group expense splitter</p>
        <h1>Multiple people paid. Everyone settles up.</h1>
        <p class="subtitle">
          Add each payment and share the URL. The group list and expenses reload from the link.
        </p>
      </div>
      <div class="stat-card">
        <p class="label">Total spent</p>
        <p class="value">{{ currency.format(totalSpent) }}</p>
        <p class="caption">
          {{ uniquePeople.length }} people · {{ currency.format(perPersonShare) }} each
        </p>
      </div>
    </header>

    <main class="grid">
      <section class="panel">
        <h2>Expenses paid</h2>
        <div class="field-group">
          <label class="field">
            <span>New expense</span>
            <div class="expense-row">
              <input
                v-model.trim="newExpensePayer"
                type="text"
                placeholder="Paid by"
                list="people-list"
              />
              <input
                v-model.number="newExpenseAmount"
                type="number"
                min="0"
                step="0.01"
                placeholder="Amount"
              />
              <input v-model.trim="newExpenseNote" type="text" placeholder="Note" />
              <button type="button" class="primary" @click="addExpense">Add</button>
            </div>
          </label>
        </div>

        <datalist id="people-list">
          <option v-for="name in uniquePeople" :key="name" :value="name" />
        </datalist>

        <div class="list">
          <div v-for="(expense, index) in expenses" :key="index" class="list-item">
            <input v-model.trim="expense.payer" type="text" list="people-list" />
            <input v-model.number="expense.amount" type="number" min="0" step="0.01" />
            <input v-model.trim="expense.note" type="text" placeholder="Note" />
            <button type="button" class="ghost" @click="removeExpense(index)">Remove</button>
          </div>
        </div>

        <div v-if="unknownPayers.length" class="warning">
          Payers not in the group list: {{ unknownPayers.join(', ') }}. Add them to include in the split.
        </div>
      </section>

      <section class="panel">
        <h2>People in the group</h2>
        <div class="field-group">
          <label class="field">
            <span>Add person</span>
            <div class="add-row">
              <input
                v-model.trim="newPersonName"
                type="text"
                placeholder="Alex"
                @keyup.enter="addPerson"
              />
              <button type="button" class="primary" @click="addPerson">Add</button>
            </div>
          </label>
        </div>

        <div class="list">
          <div v-for="(person, index) in people" :key="index" class="list-item">
            <input v-model.trim="person.name" type="text" />
            <button type="button" class="ghost" @click="removePerson(index)">Remove</button>
          </div>
        </div>

        <div v-if="duplicateNames.length" class="warning">
          Duplicate names detected: {{ duplicateNames.join(', ') }}. Duplicates are merged in the split.
        </div>

        <div class="split-list">
          <p v-if="uniquePeople.length > 0" class="label">Active group</p>
          <div class="pill-row">
            <span v-for="name in uniquePeople" :key="name" class="pill">{{ name }}</span>
          </div>
        </div>
      </section>

      <section class="panel">
        <h2>Settle up</h2>
        <div class="summary">
          <div>
            <p class="label">Total spent</p>
            <p class="value">{{ currency.format(totalSpent) }}</p>
          </div>
          <div>
            <p class="label">Per-person share</p>
            <p class="value">{{ currency.format(perPersonShare) }}</p>
          </div>
        </div>

        <div class="balance-list">
          <div v-for="person in balances" :key="person.name" class="balance-item">
            <div>
              <p class="label">{{ person.name }}</p>
              <p class="caption">Paid {{ currency.format(person.paid) }}</p>
            </div>
            <p class="balance" :class="{ positive: person.balance >= 0, negative: person.balance < 0 }">
              {{ person.balance >= 0 ? 'Receives' : 'Owes' }}
              {{ currency.format(Math.abs(person.balance)) }}
            </p>
          </div>
        </div>
      </section>

      <section class="panel share">
        <h2>Share this split</h2>
        <p class="muted">The data lives in the URL. Copy and send it to the group.</p>
        <input ref="shareInput" class="share-url" type="text" :value="shareableUrl" readonly />
        <div class="share-actions">
          <button type="button" class="primary" @click="shareSplit">
            Share link
          </button>
          <span v-if="shareStatus" class="hint">{{ shareStatus }}</span>
          <span v-else-if="!canShare && !canCopy" class="hint">
            Sharing is not supported in this browser.
          </span>
          <span v-else-if="!canShare && canCopy" class="hint">
            Sharing not available here. Copy will be used instead.
          </span>
        </div>
      </section>
    </main>
  </div>
</template>

<style lang="scss">
:root {
  color-scheme: only light;
  --bg-1: #f7f1e8;
  --bg-2: #f4ddc8;
  --ink-1: #1d1b19;
  --ink-2: #4c4037;
  --ink-3: #7c6a5d;
  --accent: #ff7a59;
  --accent-dark: #e2613e;
  --card: #ffffff;
  --outline: rgba(77, 64, 55, 0.12);
  font-family: 'DM Sans', 'Segoe UI', sans-serif;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background: radial-gradient(circle at top left, var(--bg-2), var(--bg-1) 55%);
  color: var(--ink-1);
}

h1,
h2,
.value {
  font-family: 'Fraunces', 'Times New Roman', serif;
  font-weight: 600;
  margin: 0;
}

h1 {
  font-size: clamp(2.2rem, 4vw, 3.4rem);
  letter-spacing: -0.02em;
}

h2 {
  font-size: 1.3rem;
}

p {
  margin: 0;
}

input,
button {
  font: inherit;
}

.page {
  min-height: 100vh;
  padding: 3.5rem clamp(1.5rem, 3vw, 4rem) 4rem;
}

.hero {
  display: flex;
  flex-wrap: wrap;
  gap: 2rem;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 2.5rem;
}

.eyebrow {
  text-transform: uppercase;
  letter-spacing: 0.3em;
  font-size: 0.75rem;
  color: var(--ink-3);
  margin-bottom: 0.6rem;
}

.subtitle {
  color: var(--ink-2);
  margin-top: 0.9rem;
  line-height: 1.5;
}

.stat-card {
  background: var(--card);
  border-radius: 24px;
  padding: 1.5rem 2rem;
  min-width: 240px;
  box-shadow: 0 24px 45px rgba(29, 27, 25, 0.08);
}

.label {
  text-transform: uppercase;
  letter-spacing: 0.18em;
  font-size: 0.7rem;
  color: var(--ink-3);
}

.caption {
  color: var(--ink-2);
  margin-top: 0.5rem;
  font-size: 0.95rem;
}

.grid {
  display: grid;
  gap: 1.8rem;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
}

.panel {
  background: var(--card);
  border-radius: 24px;
  padding: 1.75rem;
  box-shadow: 0 20px 40px rgba(29, 27, 25, 0.08);
  border: 1px solid var(--outline);
}

.field-group {
  display: grid;
  gap: 1rem;
  margin-top: 1.2rem;
}

.field {
  display: grid;
  gap: 0.5rem;
  color: var(--ink-2);
}

.field input {
  padding: 0.75rem 0.9rem;
  border-radius: 12px;
  border: 1px solid var(--outline);
  background: #fdfcfb;
}

.add-row,
.expense-row {
  display: grid;
  gap: 0.75rem;
}

.expense-row {
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
}

.list {
  display: grid;
  gap: 0.8rem;
  margin-top: 1.4rem;
}

.list-item {
  display: flex;
  gap: 0.6rem;
  align-items: center;
  flex-wrap: wrap;
}

.list-item input {
  flex: 1 1 140px;
  padding: 0.65rem 0.8rem;
  border-radius: 12px;
  border: 1px solid var(--outline);
}

.warning {
  background: rgba(255, 122, 89, 0.12);
  border: 1px solid rgba(255, 122, 89, 0.3);
  color: #9d3d25;
  padding: 0.75rem 1rem;
  border-radius: 14px;
  margin-top: 1rem;
  font-size: 0.9rem;
}

.summary {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 1rem;
  margin-top: 1.4rem;
}

.value {
  font-size: 1.8rem;
  margin-top: 0.25rem;
}

.split-list {
  margin-top: 1rem;
}

.balance-list {
  display: grid;
  gap: 0.8rem;
  margin-top: 1.2rem;
}

.balance-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  padding: 0.85rem 1rem;
  border-radius: 16px;
  background: #fdf7f2;
  border: 1px solid rgba(77, 64, 55, 0.08);
}

.balance {
  font-weight: 600;
}

.balance.positive {
  color: #2b7a3d;
}

.balance.negative {
  color: #a5422d;
}

button {
  border: none;
  padding: 0.7rem 1.1rem;
  border-radius: 999px;
  background: var(--accent);
  color: white;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

button:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px rgba(226, 97, 62, 0.3);
}

button.ghost {
  background: var(--bg-2);
  color: var(--ink-2);
  border: 1px solid var(--outline);
}

button.ghost:hover {
  box-shadow: none;
  transform: translateY(-1px);
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
  box-shadow: none;
  transform: none;
}

.pill-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.6rem;
}

.pill {
  padding: 0.35rem 0.8rem;
  background: #f6ede4;
  border-radius: 999px;
  color: var(--ink-2);
  font-size: 0.9rem;
}

.share {
  grid-column: 1 / -1;
}

.share-url {
  width: 100%;
  padding: 0.85rem 1rem;
  border-radius: 14px;
  border: 1px dashed var(--outline);
  background: #fff9f4;
  margin-top: 0.8rem;
  color: var(--ink-2);
}

.share-actions {
  margin-top: 1rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.hint {
  font-size: 0.9rem;
  color: var(--ink-3);
}

.muted {
  color: var(--ink-2);
  margin-top: 0.5rem;
}

@media (max-width: 720px) {
  .hero {
    align-items: flex-start;
  }

  .stat-card {
    width: 100%;
  }

  button {
    width: 100%;
  }
}
</style>
