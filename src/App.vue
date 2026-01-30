<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'

type Person = {
  name: string
}

const payerName = ref('')
const totalAmount = ref(0)
const includePayer = ref(true)
const people = ref<Person[]>([])
const newPersonName = ref('')
const isHydrated = ref(false)

const currency = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'INR',
  maximumFractionDigits: 2,
})

const splitNames = computed(() => {
  const names = people.value
    .map((person) => person.name.trim())
    .filter((name) => name.length > 0)

  const payer = payerName.value.trim()
  if(includePayer.value && payer.length > 0) {
    names.push(payer)
  }

  const unique: string[] = []
  const seen = new Set<string>()
  for(const name of names) {
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
  const allNames = [payerName.value, ...people.value.map((person) => person.name)]
    .map((name) => name.trim())
    .filter((name) => name.length > 0)

  for(const name of allNames) {
    const key = name.toLowerCase()
    nameCounts.set(key, (nameCounts.get(key) ?? 0) + 1)
  }

  return Array.from(nameCounts.entries())
    .filter(([, count]) => count > 1)
    .map(([name]) => name)
})

const normalizedTotal = computed(() => {
  if(!Number.isFinite(totalAmount.value)) {
    return 0
  }
  return Math.max(0, totalAmount.value)
})

const perPersonShare = computed(() => {
  const count = splitNames.value.length
  if(count === 0 || normalizedTotal.value <= 0) {
    return 0
  }
  return normalizedTotal.value / count
})

const payerNet = computed(() => {
  if(normalizedTotal.value <= 0) {
    return 0
  }
  if(!includePayer.value) {
    return normalizedTotal.value
  }
  if(splitNames.value.length === 0) {
    return 0
  }
  return normalizedTotal.value - perPersonShare.value
})

const shareableUrl = computed(() => {
  const url = new URL(window.location.href)
  const params = new URLSearchParams()

  const payer = payerName.value.trim()
  if(payer.length > 0) {
    params.set('payer', payer)
  }

  if(normalizedTotal.value > 0) {
    params.set('total', String(normalizedTotal.value))
  }

  const filteredPeople = people.value
    .map((person) => person.name.trim())
    .filter((name) => name.length > 0)

  if(filteredPeople.length > 0) {
    params.set('people', JSON.stringify(filteredPeople))
  }

  params.set('includePayer', includePayer.value ? '1' : '0')

  const query = params.toString()
  url.search = query
  return url.toString()
})

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

function loadFromUrl() {
  const params = new URLSearchParams(window.location.search)

  payerName.value = params.get('payer') ?? 'Alex'

  const totalParam = params.get('total')
  const parsedTotal = totalParam ? Number.parseFloat(totalParam) : Number.NaN
  totalAmount.value = Number.isFinite(parsedTotal) ? parsedTotal : 120

  const includeParam = params.get('includePayer')
  if(includeParam === '0' || includeParam === 'false') {
    includePayer.value = false
  } else if(includeParam === '1' || includeParam === 'true') {
    includePayer.value = true
  } else {
    includePayer.value = true
  }

  const peopleParam = params.get('people')
  if(peopleParam) {
    try {
      const parsed = JSON.parse(peopleParam)
      if(Array.isArray(parsed)) {
        people.value = parsed
          .filter((entry) => typeof entry === 'string')
          .map((name) => ({ name }))
      }
    } catch {
      people.value = []
    }
  }

  if(people.value.length === 0) {
    people.value = [{ name: 'Sam' }, { name: 'Riya' }]
  }
}

function syncUrl() {
  if(!isHydrated.value) {
    return
  }
  const url = new URL(window.location.href)
  url.search = ''

  const params = new URLSearchParams()
  const payer = payerName.value.trim()
  if(payer.length > 0) {
    params.set('payer', payer)
  }
  if(normalizedTotal.value > 0) {
    params.set('total', String(normalizedTotal.value))
  }

  const filteredPeople = people.value
    .map((person) => person.name.trim())
    .filter((name) => name.length > 0)

  if(filteredPeople.length > 0) {
    params.set('people', JSON.stringify(filteredPeople))
  }

  params.set('includePayer', includePayer.value ? '1' : '0')

  const query = params.toString()
  const nextUrl = query.length ? `${url.pathname}?${query}` : url.pathname
  window.history.replaceState({}, '', nextUrl)
}

onMounted(() => {
  loadFromUrl()
  isHydrated.value = true
})

watch([payerName, totalAmount, includePayer, people], syncUrl, { deep: true })
</script>

<template>
  <div class="page">
    <header class="hero">
      <div>
        <p class="eyebrow">Group expense splitter</p>
        <h1>One person paid. Everyone settles up.</h1>
        <p class="subtitle">
          Shareable URL keeps the payer, amount, and people list. Open the link to restore the split.
        </p>
      </div>
      <div class="stat-card">
        <p class="label">Total to split</p>
        <p class="value">{{ currency.format(normalizedTotal) }}</p>
        <p class="caption">
          {{ splitNames.length }} people · {{ currency.format(perPersonShare) }} each
        </p>
      </div>
    </header>

    <main class="grid">
      <section class="panel">
        <h2>Payment</h2>
        <div class="field-group">
          <label class="field">
            <span>Payer name</span>
            <input v-model.trim="payerName" type="text" placeholder="Alex" />
          </label>
          <label class="field">
            <span>Total amount</span>
            <input v-model.number="totalAmount" type="number" min="0" step="0.01" />
          </label>
        </div>

        <label class="toggle">
          <input v-model="includePayer" type="checkbox" />
          <span>Include the payer in the split</span>
        </label>

        <div v-if="duplicateNames.length" class="warning">
          Duplicate names detected: {{ duplicateNames.join(', ') }}. Duplicates are merged in the split.
        </div>

        <div class="summary">
          <div>
            <p class="label">Per-person share</p>
            <p class="value">{{ currency.format(perPersonShare) }}</p>
          </div>
          <div>
            <p class="label">Payer receives</p>
            <p class="value">{{ currency.format(payerNet) }}</p>
          </div>
        </div>
      </section>

      <section class="panel">
        <h2>People splitting the expense</h2>
        <div class="field-group">
          <label class="field">
            <span>Add person</span>
            <div class="add-row">
              <input
                v-model.trim="newPersonName"
                type="text"
                placeholder="Sam"
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

        <div class="split-list">
          <p class="label">Active split</p>
          <div class="pill-row">
            <span v-for="name in splitNames" :key="name" class="pill">{{ name }}</span>
          </div>
        </div>
      </section>

      <section class="panel share">
        <h2>Share this split</h2>
        <p class="muted">The data lives in the URL. Copy and send it to the group.</p>
        <input class="share-url" type="text" :value="shareableUrl" readonly />
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
  max-width: 520px;
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

.add-row {
  display: flex;
  gap: 0.75rem;
}

.add-row input {
  flex: 1;
}

.toggle {
  display: flex;
  gap: 0.6rem;
  align-items: center;
  margin-top: 1rem;
  color: var(--ink-2);
}

.toggle input {
  accent-color: var(--accent);
  width: 18px;
  height: 18px;
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
  flex: 1;
  padding: 0.65rem 0.8rem;
  border-radius: 12px;
  border: 1px solid var(--outline);
}

.split-list {
  margin-top: 1rem;
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

  .add-row {
    flex-direction: column;
  }

  button {
    width: 100%;
  }
}
</style>
