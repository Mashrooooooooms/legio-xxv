<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="admin-panel">
      
      <!-- Шапка -->
      <header class="admin-header">
        <div class="header-left">
          <h2>Книга учета</h2>
          <span v-if="authenticated" class="status-badge">Онлайн</span>
        </div>
        <div class="header-right">
          <button v-if="authenticated" class="btn-refresh" @click="refreshData" :disabled="isLoading">
            <span class="icon">↻</span> Обновить
          </button>
          <button class="btn-close" @click="close">✕</button>
        </div>
      </header>

      <!-- Экран авторизации -->
      <div v-if="!authenticated" class="auth-screen">
        <div class="auth-card">
          <h3>Вход в систему</h3>
          <p>Введите данные для доступа к гильдии</p>
          <input type="text" v-model="login" placeholder="Логин" @keyup.enter="checkAuth" autofocus />
          <input type="password" v-model="password" placeholder="Пароль" @keyup.enter="checkAuth" />
          <button class="btn-primary" @click="checkAuth">Войти</button>
          <div v-if="errorMsg" class="auth-error">{{ errorMsg }}</div>
        </div>
      </div>

      <!-- Основной интерфейс -->
      <div v-else class="admin-content">
        
        <!-- ЛЕВАЯ ЧАСТЬ (Склад и Казна) -->
        <main class="main-section">
          <div class="tabs">
            <button :class="{ active: activeTab === 'stock' }" @click="activeTab = 'stock'">📦 Склад</button>
            <button :class="{ active: activeTab === 'treasury' }" @click="activeTab = 'treasury'">💰 Казна</button>
          </div>

          <!-- ===== СКЛАД ===== -->
          <div v-if="activeTab === 'stock'" class="tab-content">
            <!-- Форма добавления -->
            <div class="card form-card">
              <div class="form-row">
                <div class="input-group">
                  <label>Тип</label>
                  <select v-model="newItem.type">
                    <option value="" disabled>Выбрать</option>
                    <option v-for="t in filterTags" :key="t" :value="t">{{ t }}</option>
                  </select>
                </div>
                <div class="input-group wide">
                  <label>Название</label>
                  <input v-model="newItem.name" list="stock-names" type="text" placeholder="Например: Железный меч" />
                  <datalist id="stock-names">
                    <option v-for="name in uniqueStockNames" :key="name" :value="name"></option>
                  </datalist>
                </div>
                <div class="input-group narrow">
                  <label>Кол-во</label>
                  <input v-model.number="newItem.quantity" type="number" min="1" />
                </div>
                <button class="btn-primary" @click="addStockItem" :disabled="addingItem">Добавить</button>
                <button class="btn-secondary" @click="openBatchModal">Партией</button>
              </div>
            </div>

            <!-- Фильтры -->
            <div class="toolbar">
              <div class="filter-tags">
                <button v-for="tag in filterTags" :key="tag" class="tag"
                  :class="{ active: activeTypeFilter === tag }"
                  @click="activeTypeFilter = activeTypeFilter === tag ? '' : tag">{{ tag }}</button>
                <button class="tag" :class="{ active: showEmptyOnly }"
                  @click="showEmptyOnly = !showEmptyOnly; activeTypeFilter = ''">Нулевые остатки</button>
              </div>
              <input type="text" v-model="stockSearch" placeholder="Поиск..." class="search-input" />
            </div>

            <!-- Таблица -->
            <div class="table-wrapper">
              <table class="data-table">
                <thead>
                  <tr><th>ID</th><th>Тип</th><th>Название</th><th>Кол-во</th><th>Дата</th><th></th></tr>
                </thead>
                <tbody>
                  <tr v-for="item in filteredStock" :key="item.id">
                    <td class="text-muted">{{ item.id }}</td>
                    <td>{{ item.tape }}</td>
                    <td class="font-bold">{{ item.name }}</td>
                    <td><span class="badge" :class="item.quantity > 0 ? 'badge-success' : 'badge-danger'">{{ item.quantity }}</span></td>
                    <td class="text-muted">{{ item.date }}</td>
                    <td class="actions">
                      <button class="btn-icon" @click="openIncreaseDialog(item)" title="Добавить">+</button>
                      <button class="btn-icon" @click="openDecreaseDialog(item)" title="Убавить">-</button>
                      <button class="btn-icon sell" @click="openSellDialog(item)" title="Продать">💰</button>
                    </td>
                  </tr>
                  <tr v-if="filteredStock.length === 0"><td colspan="6" class="empty">Нет предметов</td></tr>
                </tbody>
              </table>
            </div>
          </div>

          <!-- ===== КАЗНА ===== -->
          <div v-if="activeTab === 'treasury'" class="tab-content">
            <div class="wallet-card">
              <div class="wallet-label">Текущий баланс</div>
              <div class="wallet-value">{{ treasury.total }} <span>монет</span></div>
            </div>

            <div class="grid-2col">
              <div class="card">
                <h4 class="card-title text-success">Пополнение</h4>
                <div class="input-group mb-2">
                  <input type="number" v-model.number="incomeAmount" placeholder="Сумма" />
                </div>
                <div class="input-group mb-2">
                  <input type="text" v-model="incomeNote" placeholder="Комментарий" />
                </div>
                <button class="btn-primary success" @click="addIncome" :disabled="processingFinance">Добавить</button>
              </div>
              <div class="card">
                <h4 class="card-title text-danger">Списание</h4>
                <div class="input-group mb-2">
                  <input type="number" v-model.number="expenseAmount" placeholder="Сумма" />
                </div>
                <div class="input-group mb-2">
                  <input type="text" v-model="expenseNote" placeholder="Комментарий" />
                </div>
                <button class="btn-primary danger" @click="addExpense" :disabled="processingFinance">Списать</button>
              </div>
            </div>

            <h4 class="section-title">История операций</h4>
            <div class="table-wrapper">
              <table class="data-table">
                <thead><tr><th>Дата</th><th>Тип</th><th>Сумма</th><th>Комментарий</th></tr></thead>
                <tbody>
                  <tr v-for="op in treasury.history" :key="op.id">
                    <td class="text-muted">{{ op.date }}</td>
                    <td>
                      <span class="badge" :class="op.type === 'income' ? 'badge-success' : 'badge-danger'">
                        {{ op.type === 'income' ? 'Приход' : 'Расход' }}
                      </span>
                    </td>
                    <td class="font-bold" :class="op.type === 'income' ? 'text-success' : 'text-danger'">
                      {{ op.type === 'income' ? '+' : '-' }}{{ op.amount }}
                    </td>
                    <td class="text-muted">{{ op.note || '—' }}</td>
                  </tr>
                  <tr v-if="treasury.history.length === 0"><td colspan="4" class="empty">Нет операций</td></tr>
                </tbody>
              </table>
            </div>
          </div>
        </main>

        <!-- ПРАВАЯ ЧАСТЬ (Крафты) -->
        <aside class="craft-section">
          <div class="craft-header">
            <h3>⚒️ Крафты</h3>
            <button class="btn-icon" @click="toggleRecipeForm" title="Новый рецепт">+</button>
          </div>

          <!-- Форма создания рецепта -->
          <div v-if="showRecipeForm" class="recipe-form">
            <input v-model="newRecipe.name" type="text" placeholder="Название предмета" class="mb-2" />
            <div v-for="(ing, idx) in newRecipe.ingredients" :key="idx" class="ingredient-row">
              <select v-model="ing.name" class="flex-1">
                <option value="" disabled>Выбрать предмет</option>
                <option v-for="item in uniqueStockNames" :key="item" :value="item">{{ item }}</option>
              </select>
              <input type="number" v-model.number="ing.quantity" min="1" class="w-16" />
              <button class="btn-icon danger" @click="removeRecipeIngredient(idx)">✕</button>
            </div>
            <button class="btn-secondary text-sm w-full" @click="addRecipeIngredient">+ Ингредиент</button>
            <div class="form-actions">
              <button class="btn-secondary" @click="showRecipeForm = false">Отмена</button>
              <button class="btn-primary" @click="saveRecipe">Сохранить</button>
            </div>
          </div>

          <!-- Список рецептов -->
          <div class="recipe-list">
            <div v-for="recipe in allRecipes" :key="recipe.id" class="recipe-card" :class="{ 'can-craft': craftableIds.includes(recipe.id) }">
              <div class="recipe-top">
                <div class="recipe-name">
                  {{ recipe.name }}
                  <span v-if="craftableIds.includes(recipe.id)" class="badge badge-success">Готово</span>
                </div>
                <button class="btn-icon danger" @click="deleteRecipe(recipe.id)" title="Удалить рецепт">🗑️</button>
              </div>
              <div class="recipe-ings">
                <span v-for="(ing, i) in recipe.ingredients" :key="i" class="ing-tag">
                  {{ ing.name }} x{{ ing.quantity }}
                </span>
              </div>
            </div>
            <div v-if="allRecipes.length === 0" class="empty-state">Рецептов пока нет</div>
          </div>
        </aside>

      </div>
    </div>
  </div>

  <!-- ========== МОДАЛКИ ========== -->
  <!-- Партия -->
  <div v-if="batchModalVisible" class="modal-overlay nested" @click.self="closeBatchModal">
    <div class="modal-card">
      <h3>Добавить партией</h3>
      <div class="batch-list">
        <div v-for="(item,idx) in batchItems" :key="idx" class="batch-row">
          <select v-model="item.type"><option value="" disabled>Тип</option><option v-for="t in filterTags" :value="t">{{ t }}</option></select>
          <input v-model="item.name" list="stock-names" type="text" placeholder="Название" />
          <input v-model.number="item.quantity" type="number" min="1" placeholder="Кол-во" />
          <button class="btn-icon danger" @click="removeBatchRow(idx)">✕</button>
        </div>
      </div>
      <button class="btn-secondary mb-4" @click="addBatchRow">+ Строка</button>
      <div class="form-actions">
        <button class="btn-secondary" @click="closeBatchModal">Отмена</button>
        <button class="btn-primary" @click="submitBatch" :disabled="batchSubmitting">Сохранить всё</button>
      </div>
    </div>
  </div>

  <!-- Продажа -->
  <div v-if="sellDialog.visible" class="modal-overlay nested" @click.self="closeSellDialog">
    <div class="modal-card">
      <h3>Продажа: {{ sellDialog.itemName }}</h3>
      <div class="input-group"><label>Кол-во (макс. {{ sellDialog.maxQuantity }})</label><input type="number" v-model.number="sellDialog.quantity" /></div>
      <div class="input-group"><label>Сумма</label><input type="number" v-model.number="sellDialog.amount" /></div>
      <div v-if="sellDialog.error" class="auth-error">{{ sellDialog.error }}</div>
      <div class="form-actions">
        <button class="btn-secondary" @click="closeSellDialog">Отмена</button>
        <button class="btn-primary success" @click="confirmSell">Продать</button>
      </div>
    </div>
  </div>

  <!-- Изменение кол-ва (Инкремент) -->
  <div v-if="increaseDialog.visible" class="modal-overlay nested" @click.self="closeIncreaseDialog">
    <div class="modal-card">
      <h3>Добавить: {{ increaseDialog.itemName }}</h3>
      <div class="input-group"><label>Кол-во</label><input type="number" v-model.number="increaseDialog.quantity" /></div>
      <div class="form-actions">
        <button class="btn-secondary" @click="closeIncreaseDialog">Отмена</button>
        <button class="btn-primary" @click="confirmIncrease">Применить</button>
      </div>
    </div>
  </div>

  <!-- Изменение кол-ва (Декремент) -->
  <div v-if="decreaseDialog.visible" class="modal-overlay nested" @click.self="closeDecreaseDialog">
    <div class="modal-card">
      <h3>Убавить: {{ decreaseDialog.itemName }}</h3>
      <div class="input-group"><label>Кол-во (макс. {{ decreaseDialog.maxQuantity }})</label><input type="number" v-model.number="decreaseDialog.quantity" /></div>
      <div class="form-actions">
        <button class="btn-secondary" @click="closeDecreaseDialog">Отмена</button>
        <button class="btn-primary danger" @click="confirmDecrease">Списать</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onBeforeUnmount } from 'vue'

const props = defineProps({ visible: Boolean })
const emit = defineEmits(['close'])

const API_BASE = 'http://89.127.206.247:3000';
const LOGIN = "ЯЩЕР"
const PASSWORD = "1234"

// Состояние
const authenticated = ref(false)
const login = ref("")
const password = ref("")
const errorMsg = ref("")
const isLoading = ref(false)
const activeTab = ref("stock")
let autoUpdateInterval = null

const stockItems = ref([])
const treasury = ref({ total: 0, history: [] })
const allRecipes = ref([])
const craftableIds = ref([])

// Склад
const filterTags = ["Оружие", "Расходник", "Ресурс"]
const activeTypeFilter = ref("")
const showEmptyOnly = ref(false)
const stockSearch = ref("")
const newItem = ref({ type: "Ресурс", name: "", quantity: 1 })
const addingItem = ref(false)

// Казна
const incomeAmount = ref(null)
const expenseAmount = ref(null)
const incomeNote = ref("")
const expenseNote = ref("")
const processingFinance = ref(false)

// Рецепты
const showRecipeForm = ref(false)
const newRecipe = ref({ name: "", ingredients: [{ name: "", quantity: 1 }] })
const uniqueStockNames = computed(() => [...new Set(stockItems.value.map(i => i.name))])

// Модалки
const sellDialog = ref({ visible: false, itemId: null, itemName: "", maxQuantity: 0, quantity: 1, amount: 0, error: "" })
const increaseDialog = ref({ visible: false, itemId: null, itemName: "", itemType: "", quantity: 1 })
const decreaseDialog = ref({ visible: false, itemId: null, itemName: "", maxQuantity: 0, quantity: 1 })
const batchModalVisible = ref(false)
const batchItems = ref([])
const batchSubmitting = ref(false)

// Нормализация названия (ЖЕЛЕзный Меч -> Железный меч)
function getCanonicalName(inputName) {
  if (!inputName || !inputName.trim()) return "";
  const lower = inputName.trim().toLowerCase();
  const found = stockItems.value.find(i => i.name.toLowerCase() === lower);
  return found ? found.name : inputName.trim();
}

// API
async function fetchData() {
  isLoading.value = true
  try {
    const [accRes, recsRes, craftRes] = await Promise.all([
      fetch(`${API_BASE}/api/accounting`),
      fetch(`${API_BASE}/api/recipes`),
      fetch(`${API_BASE}/api/recipes/craftable`)
    ]);
    
    const acc = await accRes.json()
    stockItems.value = acc.stock || []
    treasury.value = { total: acc.total || 0, history: acc.history || [] }
    
    allRecipes.value = await recsRes.json()
    const craftable = await craftRes.json()
    craftableIds.value = craftable.map(c => c.id)
    
  } catch (e) { console.error(e); errorMsg.value = "Ошибка загрузки данных" } finally { isLoading.value = false }
}

// Авторизация
function checkAuth() {
  if (login.value === LOGIN && password.value === PASSWORD) {
    authenticated.value = true
    fetchData()
    errorMsg.value = ""
    if (autoUpdateInterval) clearInterval(autoUpdateInterval)
    autoUpdateInterval = setInterval(() => { if (authenticated.value) fetchData() }, 30000)
  } else {
    errorMsg.value = "Неверный логин или пароль"
    password.value = ""
  }
}

// Склад
async function addStockItem() {
  if (!newItem.value.name || newItem.value.quantity < 1) return alert("Заполните название и количество")
  addingItem.value = true
  // Нормализуем имя перед отправкой
  const canonicalName = getCanonicalName(newItem.value.name)
  
  try {
    await fetch(`${API_BASE}/api/accounting/stock`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ action: 'addStock', type: newItem.value.type, name: canonicalName, quantity: newItem.value.quantity })
    })
    await fetchData()
    newItem.value.name = ""
    newItem.value.quantity = 1
  } catch (e) { alert("Ошибка при добавлении") } finally { addingItem.value = false }
}

// Казна
async function addIncome() {
  if (!incomeAmount.value) return
  processingFinance.value = true
  await fetch(`${API_BASE}/api/accounting/finance`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'addIncome', amount: incomeAmount.value, note: incomeNote.value })
  })
  await fetchData(); incomeAmount.value = null; incomeNote.value = ""; processingFinance.value = false
}

async function addExpense() {
  if (!expenseAmount.value) return
  processingFinance.value = true
  await fetch(`${API_BASE}/api/accounting/finance`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'addExpense', amount: expenseAmount.value, note: expenseNote.value })
  })
  await fetchData(); expenseAmount.value = null; expenseNote.value = ""; processingFinance.value = false
}

// Рецепты
async function saveRecipe() {
  if (!newRecipe.value.name.trim()) return alert("Введите название")
  const validIngs = newRecipe.value.ingredients.filter(i => i.name && i.quantity > 0)
  if (!validIngs.length) return alert("Добавьте ингредиенты")
  
  await fetch(`${API_BASE}/api/recipes`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name: newRecipe.value.name, ingredients: validIngs })
  })
  await fetchData()
  showRecipeForm.value = false
  newRecipe.value = { name: "", ingredients: [{ name: "", quantity: 1 }] }
}

async function deleteRecipe(id) {
  if (!confirm("Вы уверены, что хотите удалить этот рецепт?")) return;
  await fetch(`${API_BASE}/api/recipes/${id}`, { method: 'DELETE' });
  await fetchData();
}

function toggleRecipeForm() {
  showRecipeForm.value = !showRecipeForm.value
  if (showRecipeForm.value) newRecipe.value = { name: "", ingredients: [{ name: "", quantity: 1 }] }
}
function addRecipeIngredient() { newRecipe.value.ingredients.push({ name: "", quantity: 1 }) }
function removeRecipeIngredient(i) { newRecipe.value.ingredients.splice(i, 1) }

// Модалки
function openSellDialog(item) { sellDialog.value = { visible: true, itemId: item.id, itemName: item.name, maxQuantity: item.quantity, quantity: 1, amount: 0, error: "" } }
function closeSellDialog() { sellDialog.value.visible = false }
async function confirmSell() {
  if (sellDialog.value.quantity > sellDialog.value.maxQuantity) return sellDialog.value.error = "Недостаточно товара"
  await fetch(`${API_BASE}/api/accounting/stock`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'sellItem', id: sellDialog.value.itemId, quantity: sellDialog.value.quantity, amount: sellDialog.value.amount })
  })
  await fetchData(); closeSellDialog()
}

function openIncreaseDialog(item) { increaseDialog.value = { visible: true, itemId: item.id, itemName: item.name, itemType: item.tape, quantity: 1 } }
function closeIncreaseDialog() { increaseDialog.value.visible = false }
async function confirmIncrease() {
  await fetch(`${API_BASE}/api/accounting/stock`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'addStock', type: increaseDialog.value.itemType, name: increaseDialog.value.itemName, quantity: increaseDialog.value.quantity })
  })
  await fetchData(); closeIncreaseDialog()
}

function openDecreaseDialog(item) { decreaseDialog.value = { visible: true, itemId: item.id, itemName: item.name, maxQuantity: item.quantity, quantity: 1 } }
function closeDecreaseDialog() { decreaseDialog.value.visible = false }
async function confirmDecrease() {
  await fetch(`${API_BASE}/api/accounting/stock`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'removeItem', id: decreaseDialog.value.itemId, quantity: decreaseDialog.value.quantity })
  })
  await fetchData(); closeDecreaseDialog()
}

// Партия
function openBatchModal() { batchItems.value = [{ type: "Ресурс", name: "", quantity: 1 }]; batchModalVisible.value = true }
function closeBatchModal() { batchModalVisible.value = false }
function addBatchRow() { batchItems.value.push({ type: "Ресурс", name: "", quantity: 1 }) }
function removeBatchRow(i) { batchItems.value.splice(i, 1) }
async function submitBatch() {
  const valid = batchItems.value.filter(i => i.name && i.quantity > 0)
  if (!valid.length) return
  
  // Нормализуем имена в партии
  const normalized = valid.map(i => ({
    ...i,
    name: getCanonicalName(i.name)
  }))

  batchSubmitting.value = true
  await fetch(`${API_BASE}/api/accounting/stock`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'addStockBatch', items: JSON.stringify(normalized) })
  })
  await fetchData(); closeBatchModal(); batchSubmitting.value = false
}

function refreshData() { fetchData() }

const filteredStock = computed(() => {
  let f = stockItems.value
  if (showEmptyOnly.value) f = f.filter(i => i.quantity === 0)
  else {
    f = f.filter(i => i.quantity > 0)
    if (activeTypeFilter.value) f = f.filter(i => i.tape === activeTypeFilter.value)
  }
  if (stockSearch.value) f = f.filter(i => i.name.toLowerCase().includes(stockSearch.value.toLowerCase()))
  return f.sort((a,b) => b.id - a.id)
})

watch(() => props.visible, (val) => {
  if (!val) {
    authenticated.value = false; login.value = ""; password.value = ""
    if (autoUpdateInterval) { clearInterval(autoUpdateInterval); autoUpdateInterval = null }
  }
})
onBeforeUnmount(() => { if (autoUpdateInterval) clearInterval(autoUpdateInterval) })
function close() { emit('close') }
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Cinzel:wght@600;700&family=Crimson+Text:ital@0;1&family=Inter:wght@400;500;600&display=swap");

/* ===== БАЗОВЫЕ ===== */
.modal-overlay {
  position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
  background: #121212; z-index: 10000; display: flex; align-items: center; justify-content: center;
  font-family: 'Inter', sans-serif;
}
.admin-panel {
  width: 100vw; height: 100vh; background: #121212;
  display: flex; flex-direction: column; overflow: hidden; color: #e0e0e0;
}

/* ===== ШАПКА ===== */
.admin-header {
  display: flex; justify-content: space-between; align-items: center;
  padding: 16px 24px; background: #1a1a1a; border-bottom: 1px solid #2a2a2a;
}
.header-left { display: flex; align-items: center; gap: 12px; }
.header-left h2 { margin: 0; font-family: 'Cinzel', serif; color: #e8c8a0; font-size: 20px; }
.status-badge { background: rgba(44, 110, 44, 0.2); color: #80c080; padding: 4px 10px; border-radius: 12px; font-size: 12px; font-weight: 500; }
.header-right { display: flex; gap: 12px; align-items: center; }
.btn-refresh { background: #2a2a2a; border: 1px solid #333; color: #ccc; padding: 8px 16px; border-radius: 8px; cursor: pointer; display: flex; align-items: center; gap: 8px; font-size: 14px; transition: 0.2s; }
.btn-refresh:hover { background: #333; color: #fff; }
.btn-close { background: transparent; border: none; color: #888; font-size: 24px; cursor: pointer; padding: 0 8px; transition: 0.2s; }
.btn-close:hover { color: #fff; }

/* ===== АВТОРИЗАЦИЯ ===== */
.auth-screen { flex: 1; display: flex; align-items: center; justify-content: center; }
.auth-card { background: #1a1a1a; padding: 40px; border-radius: 16px; border: 1px solid #2a2a2a; width: 340px; text-align: center; }
.auth-card h3 { margin: 0 0 8px; font-family: 'Cinzel', serif; color: #e8c8a0; }
.auth-card p { color: #888; margin-bottom: 24px; font-size: 14px; }
.auth-card input { width: 100%; padding: 12px; background: #0d0d0d; border: 1px solid #333; border-radius: 8px; color: #fff; margin-bottom: 12px; font-size: 14px; outline: none; transition: 0.2s; }
.auth-card input:focus { border-color: #e8c8a0; }
.auth-error { color: #ff6b6b; font-size: 13px; margin-top: 12px; }

/* ===== ОСНОВНОЙ КОНТЕНТ ===== */
.admin-content { flex: 1; display: grid; grid-template-columns: 1fr 380px; overflow: hidden; }
.main-section { padding: 24px; overflow-y: auto; border-right: 1px solid #2a2a2a; }
.craft-section { background: #161616; display: flex; flex-direction: column; overflow: hidden; }

/* ===== ТАБЫ ===== */
.tabs { display: flex; gap: 8px; margin-bottom: 24px; }
.tabs button { background: #1a1a1a; border: 1px solid #2a2a2a; padding: 10px 24px; border-radius: 8px; color: #aaa; cursor: pointer; font-family: 'Cinzel', serif; font-size: 14px; transition: 0.2s; }
.tabs button:hover { color: #e8c8a0; border-color: #444; }
.tabs button.active { background: #e8c8a0; color: #111; border-color: #e8c8a0; font-weight: bold; }

/* ===== КАРТОЧКИ И ФОРМЫ ===== */
.card { background: #1a1a1a; border: 1px solid #2a2a2a; border-radius: 12px; padding: 20px; margin-bottom: 24px; }
.form-row { display: flex; gap: 16px; align-items: flex-end; flex-wrap: wrap; }
.input-group { display: flex; flex-direction: column; gap: 6px; flex: 1; min-width: 120px; }
.input-group.wide { flex: 2; }
.input-group.narrow { flex: 0.5; min-width: 80px; }
.input-group label { font-size: 12px; color: #888; text-transform: uppercase; letter-spacing: 0.5px; }
input, select { background: #0d0d0d; border: 1px solid #333; border-radius: 8px; padding: 12px; color: #fff; font-size: 14px; outline: none; transition: 0.2s; width: 100%; }
input:focus, select:focus { border-color: #e8c8a0; }
.btn-primary { background: #e8c8a0; border: none; padding: 12px 24px; border-radius: 8px; color: #111; font-weight: 600; cursor: pointer; transition: 0.2s; height: 42px; }
.btn-primary:hover { background: #f0d5b5; }
.btn-primary.success { background: #2c6e2c; color: #fff; }
.btn-primary.danger { background: #8b3a2a; color: #fff; }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; }
.btn-secondary { background: #2a2a2a; border: 1px solid #333; padding: 12px 24px; border-radius: 8px; color: #ccc; cursor: pointer; transition: 0.2s; height: 42px; }
.btn-secondary:hover { background: #333; color: #fff; }

/* ===== ТУЛБАР И ТАБЛИЦА ===== */
.toolbar { display: flex; justify-content: space-between; align-items: center; gap: 16px; margin-bottom: 16px; flex-wrap: wrap; }
.filter-tags { display: flex; gap: 8px; }
.tag { background: #1a1a1a; border: 1px solid #333; padding: 6px 16px; border-radius: 20px; color: #888; cursor: pointer; font-size: 13px; transition: 0.2s; }
.tag:hover { border-color: #555; color: #ccc; }
.tag.active { background: #e8c8a0; color: #111; border-color: #e8c8a0; font-weight: 600; }
.search-input { width: 250px; }
.table-wrapper { background: #1a1a1a; border: 1px solid #2a2a2a; border-radius: 12px; overflow: hidden; }
.data-table { width: 100%; border-collapse: collapse; text-align: left; }
.data-table th { background: #222; padding: 14px 16px; font-size: 12px; font-weight: 600; color: #888; text-transform: uppercase; border-bottom: 1px solid #333; }
.data-table td { padding: 14px 16px; border-bottom: 1px solid #2a2a2a; font-size: 14px; }
.data-table tr:last-child td { border-bottom: none; }
.data-table tr:hover { background: rgba(255,255,255,0.02); }
.text-muted { color: #666; font-size: 13px; }
.font-bold { font-weight: 600; color: #fff; }
.badge { padding: 4px 10px; border-radius: 12px; font-size: 12px; font-weight: 600; }
.badge-success { background: rgba(44, 110, 44, 0.2); color: #80c080; }
.badge-danger { background: rgba(139, 58, 42, 0.2); color: #ff8a65; }
.text-success { color: #80c080; }
.text-danger { color: #ff8a65; }
.actions { display: flex; gap: 6px; }
.btn-icon { width: 32px; height: 32px; border-radius: 6px; background: #2a2a2a; border: 1px solid #333; color: #ccc; cursor: pointer; display: flex; align-items: center; justify-content: center; font-size: 16px; transition: 0.2s; }
.btn-icon:hover { background: #333; color: #fff; }
.btn-icon.danger { color: #ff8a65; }
.btn-icon.sell { color: #e8c8a0; }
.empty { text-align: center; padding: 40px; color: #666; font-style: italic; }

/* ===== КАЗНА ===== */
.wallet-card { background: linear-gradient(135deg, #1a1a1a, #222); border: 1px solid #333; border-radius: 16px; padding: 30px; text-align: center; margin-bottom: 24px; }
.wallet-label { color: #888; font-size: 14px; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 8px; }
.wallet-value { font-family: 'Cinzel', serif; font-size: 48px; color: #e8c8a0; font-weight: bold; }
.wallet-value span { font-size: 18px; color: #888; font-family: 'Inter', sans-serif; }
.grid-2col { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; margin-bottom: 24px; }
.card-title { font-family: 'Cinzel', serif; font-size: 16px; margin: 0 0 16px; }
.section-title { font-family: 'Cinzel', serif; font-size: 18px; color: #e8c8a0; margin-bottom: 16px; }

/* ===== КРАФТЫ (ПРАВАЯ КОЛОНКА) ===== */
.craft-header { padding: 20px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #2a2a2a; }
.craft-header h3 { margin: 0; font-family: 'Cinzel', serif; color: #e8c8a0; font-size: 18px; }
.recipe-form { padding: 20px; background: #1a1a1a; border-bottom: 1px solid #2a2a2a; }
.mb-2 { margin-bottom: 8px; }
.mb-4 { margin-bottom: 16px; }
.text-sm { font-size: 13px; padding: 8px 12px; height: auto; }
.w-full { width: 100%; }
.ingredient-row { display: flex; gap: 8px; margin-bottom: 8px; align-items: center; }
.flex-1 { flex: 1; }
.w-16 { width: 64px; }
.form-actions { display: flex; gap: 8px; margin-top: 16px; }
.form-actions button { flex: 1; }

.recipe-list { flex: 1; overflow-y: auto; padding: 20px; display: flex; flex-direction: column; gap: 12px; }
.recipe-card { background: #222; border: 1px solid #333; border-radius: 12px; padding: 16px; transition: 0.2s; }
.recipe-card.can-craft { border-color: rgba(44, 110, 44, 0.5); background: rgba(44, 110, 44, 0.05); }
.recipe-top { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
.recipe-name { display: flex; align-items: center; gap: 8px; font-family: 'Cinzel', serif; font-weight: 600; color: #e8c8a0; }
.recipe-ings { display: flex; flex-wrap: wrap; gap: 6px; }
.ing-tag { background: #2a2a2a; padding: 4px 10px; border-radius: 6px; font-size: 12px; color: #aaa; border: 1px solid #333; }
.can-craft .ing-tag { background: rgba(44, 110, 44, 0.2); border-color: transparent; color: #80c080; }
.empty-state { text-align: center; color: #666; font-style: italic; margin-top: 20px; }

/* ===== МОДАЛКИ ===== */
.modal-overlay.nested { background: rgba(0,0,0,0.8); z-index: 10001; }
.modal-card { background: #1a1a1a; border: 1px solid #333; border-radius: 16px; padding: 24px; width: 400px; max-width: 90%; }
.modal-card h3 { margin: 0 0 20px; font-family: 'Cinzel', serif; color: #e8c8a0; text-align: center; }
.batch-list { max-height: 300px; overflow-y: auto; margin-bottom: 16px; }
.batch-row { display: flex; gap: 8px; margin-bottom: 8px; }
.batch-row select { flex: 1; }
.batch-row input { flex: 1; }
</style>