<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'

type Person = {
  name: string
  paid: boolean
  paidAmount: number
  paidTo: string
  auto: boolean
}

type Expense = {
  payer: string
  amount: number
  note: string
}

type Payment = {
  from: string
  to: string
  amount: number
  note: string
}


const people = ref<Person[]>([])
const expenses = ref<Expense[]>([])
const payments = ref<Payment[]>([])
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

const normalizedPayments = computed(() =>
  payments.value
    .map((payment) => {
      const amount = Number.isFinite(payment.amount) ? Math.max(0, payment.amount) : 0
      return {
        from: payment.from.trim(),
        to: payment.to.trim(),
        amount,
        note: payment.note.trim(),
      }
    })
    .filter((payment) => payment.from && payment.to && payment.from !== payment.to)
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

const sentByName = computed(() => {
  const map = new Map<string, number>()
  for(const payment of normalizedPayments.value) {
    if(!payment.from || !payment.to || payment.amount <= 0) {
      continue
    }
    const key = payment.from.toLowerCase()
    map.set(key, (map.get(key) ?? 0) + payment.amount)
  }
  return map
})

const receivedByName = computed(() => {
  const map = new Map<string, number>()
  for(const payment of normalizedPayments.value) {
    if(!payment.from || !payment.to || payment.amount <= 0) {
      continue
    }
    const key = payment.to.toLowerCase()
    map.set(key, (map.get(key) ?? 0) + payment.amount)
  }
  return map
})

const balances = computed(() =>
  uniquePeople.value.map((name) => {
    const key = name.toLowerCase()
    const paid = paidByName.value.get(key) ?? 0
    const sent = sentByName.value.get(key) ?? 0
    const received = receivedByName.value.get(key) ?? 0
    const balance = paid + sent - received - perPersonShare.value
    return { name, paid, sent, received, balance }
  })
)

const settlementEpsilon = 0.01
const allSettled = computed(() =>
  totalSpent.value > 0 &&
  uniquePeople.value.length > 1 &&
  balances.value.length > 0 &&
  normalizedPayments.value.length > 0 &&
  balances.value.every((person) => Math.abs(person.balance) <= settlementEpsilon)
)
const showConfetti = ref(false)
let confettiTimer: number | undefined

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

function base64UrlEncodeBytes(bytes: Uint8Array) {
  let binary = ''
  for(const byte of bytes) {
    binary += String.fromCharCode(byte)
  }
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/g, '')
}

function base64UrlDecodeToBytes(input: string) {
  const base = input.replace(/-/g, '+').replace(/_/g, '/')
  const pad = base.length % 4 ? '='.repeat(4 - (base.length % 4)) : ''
  const binary = atob(base + pad)
  const bytes = new Uint8Array(binary.length)
  for(let i = 0; i < binary.length; i += 1) {
    bytes[i] = binary.charCodeAt(i)
  }
  return bytes
}

function base64UrlEncodeString(input: string) {
  const bytes = new TextEncoder().encode(input)
  return base64UrlEncodeBytes(bytes)
}

function base64UrlDecodeString(input: string) {
  const bytes = base64UrlDecodeToBytes(input)
  return new TextDecoder().decode(bytes)
}

async function compressString(input: string) {
  if(typeof CompressionStream === 'undefined' || typeof DecompressionStream === 'undefined') {
    return `u1.${base64UrlEncodeString(input)}`
  }
  try {
    const bytes = new TextEncoder().encode(input)
    const stream = new Blob([bytes]).stream().pipeThrough(new CompressionStream('gzip'))
    const buffer = await new Response(stream).arrayBuffer()
    return `c1.${base64UrlEncodeBytes(new Uint8Array(buffer))}`
  } catch {
    return `u1.${base64UrlEncodeString(input)}`
  }
}

async function decompressString(input: string) {
  if(input.startsWith('c1.')) {
    if(typeof DecompressionStream === 'undefined') {
      throw new Error('Compression not supported')
    }
    const bytes = base64UrlDecodeToBytes(input.slice(3))
    const stream = new Blob([bytes]).stream().pipeThrough(new DecompressionStream('gzip'))
    const buffer = await new Response(stream).arrayBuffer()
    return new TextDecoder().decode(new Uint8Array(buffer))
  }
  if(input.startsWith('u1.')) {
    return base64UrlDecodeString(input.slice(3))
  }
  return base64UrlDecodeString(input)
}

function buildSharePayload() {
  return {
    v: 1,
    sid: shareId.value || undefined,
    people: people.value.map((person) => ({
      name: person.name.trim(),
      paid: person.paid,
      paidAmount: person.paidAmount,
      paidTo: person.paidTo,
      auto: person.auto,
    })),
    expenses: normalizedExpenses.value,
    payments: normalizedPayments.value,
  }
}

function generateShareId() {
  if(typeof crypto !== 'undefined' && typeof crypto.randomUUID === 'function') {
    return crypto.randomUUID()
  }
  return `sid-${Math.random().toString(36).slice(2, 10)}${Date.now().toString(36)}`
}

const hasGroupData = computed(() =>
  people.value.some((person) => person.name.trim().length > 0) ||
  expenses.value.length > 0 ||
  payments.value.length > 0
)

const encodedShare = ref('')

const shareableUrl = computed(() => {
  const url = new URL(window.location.href)
  if(!hasGroupData.value) {
    url.search = ''
    url.hash = ''
    return url.toString()
  }
  url.search = ''
  url.hash = encodedShare.value ? `d=${encodedShare.value}` : ''
  return url.toString()
})

const canShare = ref(false)
const canCopy = ref(false)
const shareStatus = ref('')
const shareInput = ref<HTMLInputElement | null>(null)
const storageKey = 'expense-splitter-state-v1'
const shareId = ref('')

function addPerson() {
  const name = newPersonName.value.trim()
  if(!name) {
    return
  }
  people.value.push({ name, paid: false, paidAmount: 0, paidTo: '', auto: false })
  newPersonName.value = ''
}

function removePerson(index: number) {
  const person = people.value[index]
  if(!person) {
    return
  }
  const key = person.name.trim().toLowerCase()
  if(!key) {
    return
  }
  const hasPaidExpense = normalizedExpenses.value.some(
    (expense) => expense.payer.toLowerCase() === key && expense.amount > 0
  )
  if(hasPaidExpense) {
    return
  }
  const nameKey = person.name.trim().toLowerCase()
  payments.value = payments.value.filter((payment) => {
    const fromKey = payment.from.trim().toLowerCase()
    const toKey = payment.to.trim().toLowerCase()
    return fromKey !== nameKey && toKey !== nameKey
  })
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

function logPaymentForPerson(person: Person) {
  const from = person.name.trim()
  const to = person.paidTo.trim()
  const amount = person.paidAmount
  if(!person.paid || !from || !to || from === to || amount <= 0) {
    return
  }

  payments.value.push({
    from,
    to,
    amount,
    note: 'Settlement',
  })
}

function removeExpense(index: number) {
  expenses.value.splice(index, 1)
}

function resetGroup() {
  people.value = []
  expenses.value = []
  payments.value = []
  newPersonName.value = ''
  newExpensePayer.value = ''
  newExpenseAmount.value = null
  newExpenseNote.value = ''
  shareStatus.value = ''
  shareId.value = ''
  if(typeof localStorage !== 'undefined') {
    localStorage.removeItem(storageKey)
  }
  syncUrl()
}


async function loadFromUrl() {
  const hashParams = new URLSearchParams(window.location.hash.replace(/^#/, ''))
  let hasUrlData = false

  const dataParam = hashParams.get('d')
  if(dataParam) {
    try {
      const decoded = await decompressString(dataParam)
      const parsed = JSON.parse(decoded)
      if(parsed && typeof parsed === 'object') {
        if(Array.isArray(parsed.people)) {
          people.value = parsed.people
            .filter((entry: unknown) => typeof entry === 'object' && entry)
            .map((entry: any) => ({
              name: typeof entry.name === 'string' ? entry.name : '',
              paid: typeof entry.paid === 'boolean' ? entry.paid : false,
              paidAmount: typeof entry.paidAmount === 'number' ? entry.paidAmount : 0,
              paidTo: typeof entry.paidTo === 'string' ? entry.paidTo : '',
              auto: typeof entry.auto === 'boolean' ? entry.auto : false,
            }))
        }
        if(Array.isArray(parsed.expenses)) {
          expenses.value = parsed.expenses
            .filter((entry: unknown) => typeof entry === 'object' && entry)
            .map((entry: any) => ({
              payer: typeof entry.payer === 'string' ? entry.payer : '',
              amount: typeof entry.amount === 'number' ? entry.amount : 0,
              note: typeof entry.note === 'string' ? entry.note : '',
            }))
        }
        if(Array.isArray(parsed.payments)) {
          payments.value = parsed.payments
            .filter((entry: unknown) => typeof entry === 'object' && entry)
            .map((entry: any) => ({
              from: typeof entry.from === 'string' ? entry.from : '',
              to: typeof entry.to === 'string' ? entry.to : '',
              amount: typeof entry.amount === 'number' ? entry.amount : 0,
              note: typeof entry.note === 'string' ? entry.note : '',
            }))
        }
        if(typeof parsed.sid === 'string') {
          shareId.value = parsed.sid
        }
        hasUrlData = true
      }
    } catch {
      // Ignore invalid hash data.
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
    if(typeof parsed?.shareId === 'string') {
      shareId.value = parsed.shareId
    }
    if(Array.isArray(parsed?.people)) {
      if(parsed.people.every((entry: unknown) => typeof entry === 'string')) {
        people.value = parsed.people.map((name: string) => ({
          name,
          paid: false,
          paidAmount: 0,
          paidTo: '',
          auto: false,
        }))
      } else {
        people.value = parsed.people
          .filter((entry: unknown) => typeof entry === 'object' && entry)
          .map((entry: any) => ({
            name: typeof entry.name === 'string' ? entry.name : '',
            paid: typeof entry.paid === 'boolean' ? entry.paid : false,
            paidAmount: typeof entry.paidAmount === 'number' ? entry.paidAmount : 0,
            paidTo: typeof entry.paidTo === 'string' ? entry.paidTo : '',
            auto: typeof entry.auto === 'boolean' ? entry.auto : false,
          }))
      }
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

    if(Array.isArray(parsed?.payments)) {
      payments.value = parsed.payments
        .filter((entry: unknown) => typeof entry === 'object' && entry)
        .map((entry: any) => ({
          from: typeof entry.from === 'string' ? entry.from : '',
          to: typeof entry.to === 'string' ? entry.to : '',
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
    shareId: shareId.value,
    people: people.value.map((person) => ({
      name: person.name.trim(),
      paid: person.paid,
      paidAmount: person.paidAmount,
      paidTo: person.paidTo,
      auto: person.auto,
    })),
    expenses: normalizedExpenses.value,
    payments: normalizedPayments.value,
  }

  try {
    localStorage.setItem(storageKey, JSON.stringify(payload))
  } catch {
    // Ignore storage errors.
  }
}

function syncPeopleFromExpenses() {
  const referenced = new Set<string>()

  for(const expense of expenses.value) {
    const payer = expense.payer.trim()
    if(payer) {
      referenced.add(payer.toLowerCase())
    }
  }

  for(const payment of payments.value) {
    const from = payment.from.trim()
    const to = payment.to.trim()
    if(from) {
      referenced.add(from.toLowerCase())
    }
    if(to) {
      referenced.add(to.toLowerCase())
    }
  }

  const known = new Set(people.value.map((person) => person.name.trim().toLowerCase()))
  let changed = false

  for(const name of referenced) {
    if(!known.has(name)) {
      known.add(name)
      const original = expenses.value.find(
        (expense) => expense.payer.trim().toLowerCase() === name
      )?.payer
        ?? payments.value.find((payment) => payment.from.trim().toLowerCase() === name)?.from
        ?? payments.value.find((payment) => payment.to.trim().toLowerCase() === name)?.to
        ?? name
      people.value.push({
        name: original,
        paid: false,
        paidAmount: 0,
        paidTo: '',
        auto: true,
      })
      changed = true
    }
  }

  const beforeLength = people.value.length
  people.value = people.value.filter((person) => {
    const key = person.name.trim().toLowerCase()
    if(!key) {
      return false
    }
    if(person.auto && !referenced.has(key)) {
      return false
    }
    return true
  })

  if(people.value.length !== beforeLength) {
    changed = true
  }

  if(changed) {
    saveToStorage()
  }

  return changed
}

function syncPeoplePaymentStatus() {
  const sentTotals = new Map<string, number>()
  const recipientTotals = new Map<string, Map<string, number>>()

  for(const payment of normalizedPayments.value) {
    if(!payment.from || !payment.to || payment.amount <= 0) {
      continue
    }
    const fromKey = payment.from.toLowerCase()
    const toKey = payment.to.toLowerCase()
    sentTotals.set(fromKey, (sentTotals.get(fromKey) ?? 0) + payment.amount)
    if(!recipientTotals.has(fromKey)) {
      recipientTotals.set(fromKey, new Map())
    }
    const perRecipient = recipientTotals.get(fromKey)!
    perRecipient.set(toKey, (perRecipient.get(toKey) ?? 0) + payment.amount)
  }

  for(const person of people.value) {
    const key = person.name.trim().toLowerCase()
    if(!key) {
      continue
    }
    const paid = paidByName.value.get(key) ?? 0
    const owed = Math.max(0, perPersonShare.value - paid)
    const sent = sentTotals.get(key) ?? 0
    const fullyPaid = owed > 0 && sent >= owed - settlementEpsilon

    if(fullyPaid) {
      if(!person.paid) {
        person.paid = true
      }
      if(person.paidAmount === 0) {
        person.paidAmount = sent
      }
      const recipients = recipientTotals.get(key)
      if(recipients && recipients.size > 0) {
        let topRecipient = ''
        let topAmount = -1
        for(const [recipient, amount] of recipients.entries()) {
          if(amount > topAmount) {
            topAmount = amount
            topRecipient = recipient
          }
        }
        const match = uniquePeople.value.find(
          (name) => name.toLowerCase() === topRecipient
        )
        if(!person.paidTo) {
          person.paidTo = match ?? person.paidTo
        }
      }
    }
  }
}

async function syncUrl() {
  if(!isHydrated.value) {
    return
  }

  if(!hasGroupData.value) {
    window.history.replaceState({}, '', window.location.pathname)
    encodedShare.value = ''
    return
  }

  const payload = buildSharePayload()
  const encoded = await compressString(JSON.stringify(payload))
  encodedShare.value = encoded
  const nextUrl = encoded ? `${window.location.pathname}#d=${encoded}` : window.location.pathname
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
  shareId.value = generateShareId()
  await syncUrl()
  saveToStorage()
  try {
    if(canShare.value) {
      await navigator.share({
        title: 'Group expense split',
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

onMounted(async () => {
  await loadFromUrl()
  syncPeopleFromExpenses()
  syncPeoplePaymentStatus()
  isHydrated.value = true
  canShare.value = typeof navigator !== 'undefined' && typeof navigator.share === 'function'
  canCopy.value =
    typeof navigator !== 'undefined' &&
    (typeof navigator.clipboard?.writeText === 'function' ||
      (typeof document !== 'undefined' && typeof document.execCommand === 'function'))
  saveToStorage()
  syncUrl()
})

watch(allSettled, (settled, prevSettled) => {
  if(settled && !prevSettled) {
    showConfetti.value = true
    if(confettiTimer) {
      window.clearTimeout(confettiTimer)
    }
    confettiTimer = window.setTimeout(() => {
      showConfetti.value = false
    }, 2200)
  }
})

watch(expenses, () => {
  syncPeopleFromExpenses()
}, { deep: true })

watch(payments, () => {
  syncPeopleFromExpenses()
  syncPeoplePaymentStatus()
}, { deep: true })

watch([people, expenses], () => {
  syncPeoplePaymentStatus()
}, { deep: true })

watch([people, expenses, payments], () => {
  syncUrl()
  saveToStorage()
}, { deep: true, flush: 'post' })

onUnmounted(() => {
  if(confettiTimer) {
    window.clearTimeout(confettiTimer)
  }
})
</script>

<template>
  <div class="page">
    <div v-if="showConfetti" class="confetti" aria-hidden="true">
      <span v-for="n in 30" :key="n" class="confetti-piece" />
    </div>
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
                @keyup.enter="addExpense"
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
                placeholder="Rakshith"
                @keyup.enter="addPerson"
              />
              <button type="button" class="primary" @click="addPerson">Add</button>
            </div>
          </label>
        </div>

        <div class="list">
          <div v-for="(person, index) in people" :key="index" class="list-item">
            <input v-model.trim="person.name" type="text" />
            <label class="paid-toggle">
              <input v-model="person.paid" type="checkbox" />
              <span>Paid</span>
            </label>
            <select v-model="person.paidTo" class="paid-select" :disabled="!person.paid">
              <option value="">Paid to</option>
              <option
                v-for="name in uniquePeople"
                :key="`${person.name}-${name}`"
                :value="name"
                :disabled="name.toLowerCase() === person.name.trim().toLowerCase()"
              >
                {{ name }}
              </option>
            </select>
            <input
              v-model.number="person.paidAmount"
              class="paid-amount"
              type="number"
              min="0"
              step="0.01"
              placeholder="Paid amount"
              :disabled="!person.paid"
            />
            <button type="button" class="ghost" @click="logPaymentForPerson(person)">
              Log payment
            </button>
            <button
              type="button"
              class="ghost"
              :disabled="normalizedExpenses.some((expense) => expense.payer.toLowerCase() === person.name.trim().toLowerCase())"
              @click="removePerson(index)"
            >
              Remove
            </button>
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
              <p class="caption">
                Paid {{ currency.format(person.paid) }} · Sent {{ currency.format(person.sent) }} · Received
                {{ currency.format(person.received) }}
              </p>
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
          <button v-if="hasGroupData" type="button" class="ghost" @click="resetGroup">
            New group expense
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

.field input,
.field select {
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

.paid-toggle {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.9rem;
  color: var(--ink-2);
}

.paid-toggle input {
  accent-color: var(--accent);
  width: 18px;
  height: 18px;
}

.paid-amount {
  max-width: 160px;
}

.paid-select {
  min-width: 140px;
  padding: 0.65rem 0.8rem;
  border-radius: 12px;
  border: 1px solid var(--outline);
  background: #fdfcfb;
  color: var(--ink-2);
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

.confetti {
  position: fixed;
  inset: 0;
  pointer-events: none;
  overflow: hidden;
  z-index: 10;
}

.confetti-piece {
  position: absolute;
  top: -10%;
  width: 10px;
  height: 16px;
  opacity: 0.9;
  animation: confetti-fall 1.8s ease-out forwards;
}

.confetti-piece:nth-child(3n) {
  width: 8px;
  height: 12px;
  animation-duration: 1.6s;
}

.confetti-piece:nth-child(4n) {
  width: 6px;
  height: 10px;
  animation-duration: 1.4s;
}

.confetti-piece:nth-child(6n) {
  width: 12px;
  height: 18px;
  animation-duration: 2s;
}

.confetti-piece:nth-child(1) { left: 5%; background: #ff7a59; animation-delay: 0s; }
.confetti-piece:nth-child(2) { left: 12%; background: #f3c969; animation-delay: 0.2s; }
.confetti-piece:nth-child(3) { left: 20%; background: #6cc3b8; animation-delay: 0.4s; }
.confetti-piece:nth-child(4) { left: 28%; background: #f28fb4; animation-delay: 0.1s; }
.confetti-piece:nth-child(5) { left: 36%; background: #8d9bff; animation-delay: 0.3s; }
.confetti-piece:nth-child(6) { left: 44%; background: #ffb347; animation-delay: 0.5s; }
.confetti-piece:nth-child(7) { left: 52%; background: #9dd56e; animation-delay: 0.15s; }
.confetti-piece:nth-child(8) { left: 60%; background: #ff7a59; animation-delay: 0.35s; }
.confetti-piece:nth-child(9) { left: 68%; background: #6cc3b8; animation-delay: 0.55s; }
.confetti-piece:nth-child(10) { left: 76%; background: #f3c969; animation-delay: 0.25s; }
.confetti-piece:nth-child(11) { left: 84%; background: #f28fb4; animation-delay: 0.45s; }
.confetti-piece:nth-child(12) { left: 92%; background: #8d9bff; animation-delay: 0.6s; }
.confetti-piece:nth-child(13) { left: 3%; background: #ffb347; animation-delay: 0.12s; }
.confetti-piece:nth-child(14) { left: 15%; background: #9dd56e; animation-delay: 0.22s; }
.confetti-piece:nth-child(15) { left: 27%; background: #ff7a59; animation-delay: 0.32s; }
.confetti-piece:nth-child(16) { left: 39%; background: #6cc3b8; animation-delay: 0.42s; }
.confetti-piece:nth-child(17) { left: 51%; background: #f3c969; animation-delay: 0.52s; }
.confetti-piece:nth-child(18) { left: 63%; background: #f28fb4; animation-delay: 0.18s; }
.confetti-piece:nth-child(19) { left: 75%; background: #8d9bff; animation-delay: 0.28s; }
.confetti-piece:nth-child(20) { left: 87%; background: #ffb347; animation-delay: 0.38s; }
.confetti-piece:nth-child(21) { left: 9%; background: #9dd56e; animation-delay: 0.48s; }
.confetti-piece:nth-child(22) { left: 21%; background: #ff7a59; animation-delay: 0.58s; }
.confetti-piece:nth-child(23) { left: 33%; background: #6cc3b8; animation-delay: 0.08s; }
.confetti-piece:nth-child(24) { left: 45%; background: #f3c969; animation-delay: 0.18s; }
.confetti-piece:nth-child(25) { left: 57%; background: #f28fb4; animation-delay: 0.28s; }
.confetti-piece:nth-child(26) { left: 69%; background: #8d9bff; animation-delay: 0.38s; }
.confetti-piece:nth-child(27) { left: 81%; background: #ffb347; animation-delay: 0.48s; }
.confetti-piece:nth-child(28) { left: 93%; background: #9dd56e; animation-delay: 0.58s; }
.confetti-piece:nth-child(29) { left: 17%; background: #ff7a59; animation-delay: 0.26s; }
.confetti-piece:nth-child(30) { left: 71%; background: #6cc3b8; animation-delay: 0.36s; }

@keyframes confetti-fall {
  0% {
    transform: translate3d(0, -20vh, 0) rotate(0deg);
  }
  100% {
    transform: translate3d(0, 110vh, 0) rotate(280deg);
  }
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
