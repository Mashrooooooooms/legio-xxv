<template>
  <div id="map" class="leaflet-map"></div>

  <!-- Индикатор загрузки -->
  <div v-if="isLoading" class="loading-indicator">
    <div class="spinner"></div>
  </div>

  <!-- Единая панель управления (слева) -->
  <header class="top-bar">
    <!-- Логотип -->
    <img :src="BASE + 'nirn.png'" alt="Nirn" class="logo" />

    <div class="divider"></div>

    <!-- Переключатель слоёв -->
    <div class="layer-switch">
      <button :class="{ active: activeLayer === 'game' }" @click="setActiveLayer('game')">Дичь</button>
      <button :class="{ active: activeLayer === 'alchemy' }" @click="setActiveLayer('alchemy')">Растения</button>
      <button :class="{ active: activeLayer === 'both' }" @click="setActiveLayer('both')">Оба</button>
    </div>

    <div class="divider"></div>

    <!-- Кнопки справа от переключателя -->
    <div class="actions">
      <button class="icon-btn" @click="openAccounting" title="Книга учета">
        <img :src="BASE + 'buh.png'" alt="Book" class="icon-img" />
      </button>
      <button v-if="isMobile" class="icon-btn" @click="burgerOpen = !burgerOpen" title="Фильтры">☰</button>
    </div>
  </header>

  <!-- Боковая панель фильтров (слева) -->
  <aside class="panel" :class="{ 'panel-open': burgerOpen || !isMobile }">
    <div class="panel-header">
      <span>Фильтры</span>
      <button v-if="isMobile" class="close-btn" @click="burgerOpen = false">✕</button>
    </div>

    <div class="panel-content">
      <!-- Фильтры дичи -->
      <div v-if="activeLayer === 'game' || activeLayer === 'both'" class="section">
        <div class="section-title">⚔️ Дичь</div>
        <div class="filter-grid">
          <label v-for="type in gameTypes" :key="type.id" class="filter-item" :class="{ active: gameFilters.includes(type.id) }">
            <input type="checkbox" :value="type.id" v-model="gameFilters" @change="applyFilters" />
            <img :src="type.icon" :alt="type.name" class="white-icon" />
            <span>{{ type.name }}</span>
          </label>
        </div>
        <div class="filter-btns">
          <button @click="selectAllGameFilters">ВСЕ</button>
          <button @click="clearAllGameFilters">НИЧЕГО</button>
        </div>
      </div>

      <!-- Фильтры алхимии -->
      <div v-if="activeLayer === 'alchemy' || activeLayer === 'both'" class="section">
        <div class="section-title">🌿 Растения</div>
        <div class="filter-grid">
          <label v-for="type in alchemyTypes" :key="type.id" class="filter-item" :class="{ active: alchemyFilters.includes(type.id) }">
            <input type="checkbox" :value="type.id" v-model="alchemyFilters" @change="applyFilters" />
            <img :src="type.icon" :alt="type.name" class="white-icon" />
            <span>{{ type.name }}</span>
          </label>
        </div>
        <div class="filter-btns">
          <button @click="selectAllAlchemyFilters">ВСЕ</button>
          <button @click="clearAllAlchemyFilters">НИЧЕГО</button>
        </div>
      </div>
    </div>

    <div class="panel-footer">
      <span>Данные синхронизируются с VPS</span>
    </div>
  </aside>

  <!-- Уведомление -->
  <transition name="fade">
    <div class="toast" v-if="toastMsg">{{ toastMsg }}</div>
  </transition>

  <!-- Модалка добавления/редактирования метки -->
  <AddMarkerModal
    :visible="showModal"
    :coords="modalCoords"
    :layer-mode="activeLayer"
    :game-types="gameTypes"
    :alchemy-types="alchemyTypes"
    :marker-to-edit="markerToEdit"
    @close="closeModal"
    @save="handleSaveMarker"
  />
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import AddMarkerModal from "./AddMarkerModal.vue";

const BASE = import.meta.env.BASE_URL;
const API_BASE = 'https://89.127.206.247';

const gameTypes = [
  { id: "elk", name: "Олень", icon: BASE + "elk.png" },
  { id: "wolf", name: "Волк", icon: BASE + "wolf.png" },
  { id: "rat", name: "Крыса", icon: BASE + "rat.png" },
  { id: "bear", name: "Медведь", icon: BASE + "bear.png" },
  { id: "troll", name: "Тролль", icon: BASE + "troll.png" },
  { id: "sabertooth", name: "Саблезуб", icon: BASE + "cat.png" },
  { id: "spider", name: "Паук", icon: BASE + "spider.png" },
  { id: "scorpio", name: "Корус", icon: BASE + "scorpio.png" },
  { id: "snake", name: "Змея", icon: BASE + "snake.png" },
  { id: "eleph", name: "Мамонт", icon: BASE + "eleph.png" },
  { id: "gig", name: "Великан", icon: BASE + "gig.png" },
  { id: "bird", name: "Фазан", icon: BASE + "bird.png" },
];

const alchemyTypes = [
  { id: "herb", name: "Трава", icon: BASE + "herb.png" },
  { id: "mushroom", name: "Гриб", icon: BASE + "mash.png" },
  { id: "flower", name: "Цветок", icon: BASE + "flower.png" },
  { id: "wildberry", name: "Ягоды", icon: BASE + "wildberris.png" },
];

const toastMsg = ref("");
const gameFilters = ref(gameTypes.map(t => t.id));
const alchemyFilters = ref(alchemyTypes.map(t => t.id));
const isLoading = ref(false);
const isMobile = ref(false);
const burgerOpen = ref(false);
const activeLayer = ref("game");

const allMarkers = ref([]);
let mapInstance = null;
let updateInterval = null;
let leafletMarkers = [];

const showModal = ref(false);
const modalCoords = ref(null);
const markerToEdit = ref(null);

// Открытие Книги Учета
function openAccounting() {
  window.open('https://legion-ledger.imperial-legion-nirn.workers.dev/', '_blank');
}

function toast(msg) {
  toastMsg.value = msg;
  setTimeout(() => (toastMsg.value = ""), 2000);
}

function handleResize() {
  const isTouchDevice = window.matchMedia("(pointer: coarse)").matches;
  isMobile.value = isTouchDevice || window.innerWidth <= 1024;
  if (!isMobile.value) burgerOpen.value = false;
}

async function loadMarkers() {
  isLoading.value = true;
  try {
    const res = await fetch(`${API_BASE}/api/markers`);
    const data = await res.json();
    allMarkers.value = data;
    refreshMapMarkers();
  } catch (err) {
    toast(`❌ Ошибка загрузки: ${err.message}`);
  } finally {
    isLoading.value = false;
  }
}

function createIcon(typeId) {
  const isAlchemy = alchemyTypes.some(t => t.id === typeId);
  const iconUrl = isAlchemy 
    ? alchemyTypes.find(t => t.id === typeId)?.icon 
    : (gameTypes.find(t => t.id === typeId)?.icon || BASE + "elk.png");
  
  const className = isAlchemy ? 'map-marker-alchemy' : 'map-marker';
  
  return L.divIcon({
    className: "",
    html: `<div class="${className}"><img src="${iconUrl}" alt="" /></div>`,
    iconSize: [36, 36],
    iconAnchor: [18, 18],
  });
}

function refreshMapMarkers() {
  leafletMarkers.forEach((item) => {
    if (mapInstance?.hasLayer(item.marker)) mapInstance.removeLayer(item.marker);
  });
  leafletMarkers.length = 0;

  allMarkers.value.forEach((m) => {
    let coordsArray = Array.isArray(m.coords) ? m.coords : JSON.parse(m.coords);
    
    const popupHtml = `
      <div class="marker-popup-content">
        <div class="marker-info">
          <b>${m.title}</b><br><span>${m.description}</span>
        </div>
        <div class="marker-actions">
          <button class="tooltip-btn btn-edit" data-id="${m.id}">✏️</button>
          <button class="tooltip-btn btn-del" data-id="${m.id}">🗑️</button>
        </div>
      </div>
    `;

    let closeTimer = null;

    const marker = L.marker(coordsArray, { icon: createIcon(m.type) })
      .bindPopup(popupHtml, {
        offset: [20, 0],
        className: 'custom-popup',
        autoPan: false,
        closeButton: false,
        minWidth: 220,
        autoClose: true,
        closeOnClick: false
      });

    marker.on('mouseover', () => {
      if (closeTimer) clearTimeout(closeTimer);
      marker.openPopup();
    });

    marker.on('mouseout', () => {
      if (closeTimer) clearTimeout(closeTimer);
      closeTimer = setTimeout(() => {
        marker.closePopup();
      }, 350);
    });

    marker.on('click', (e) => e.originalEvent.stopPropagation());

    marker.on('popupopen', (e) => {
      const el = e.popup.getElement();
      if (!el) return;

      el.onmouseenter = () => {
        if (closeTimer) clearTimeout(closeTimer);
      };
      
      el.onmouseleave = () => {
        if (closeTimer) clearTimeout(closeTimer);
        closeTimer = setTimeout(() => {
          marker.closePopup();
        }, 350);
      };

      const btnEdit = el.querySelector('.btn-edit');
      const btnDel = el.querySelector('.btn-del');

      if (btnEdit) {
        btnEdit.onclick = (ev) => {
          ev.stopPropagation();
          marker.closePopup();
          openEditModal(m);
        };
      }
      
      if (btnDel) {
        btnDel.onclick = (ev) => {
          ev.stopPropagation();
          marker.closePopup();
          deleteMarker(m.id);
        };
      }
    });

    marker.on('popupclose', () => {
      if (closeTimer) clearTimeout(closeTimer);
    });

    leafletMarkers.push({ marker, type: m.type, id: m.id });
  });

  applyFilters();
}

function applyFilters() {
  leafletMarkers.forEach(({ marker, type }) => {
    let shouldShow = false;
    const isAlchemy = alchemyTypes.some(t => t.id === type);

    if (isAlchemy && (activeLayer.value === 'alchemy' || activeLayer.value === 'both')) {
      shouldShow = alchemyFilters.value.includes(type);
    } else if (!isAlchemy && (activeLayer.value === 'game' || activeLayer.value === 'both')) {
      shouldShow = gameFilters.value.includes(type);
    }

    if (shouldShow && !mapInstance.hasLayer(marker)) marker.addTo(mapInstance);
    if (!shouldShow && mapInstance.hasLayer(marker)) mapInstance.removeLayer(marker);
  });
}

async function handleSaveMarker(markerData) {
  try {
    const isEdit = !!markerData.id;
    const method = isEdit ? 'PUT' : 'POST';
    const url = isEdit ? `${API_BASE}/api/markers/${markerData.id}` : `${API_BASE}/api/markers`;

    let coordsArray = typeof markerData.coords === 'string' ? JSON.parse(markerData.coords) : markerData.coords;

    const res = await fetch(url, {
      method,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ ...markerData, coords: coordsArray })
    });
    
    const json = await res.json();
    if (json.error) throw new Error(json.error);

    await loadMarkers();
    toast(isEdit ? `✅ Обновлено: ${markerData.title}` : `✅ Добавлено: ${markerData.title}`);
    closeModal();
  } catch (err) {
    toast(`❌ Ошибка: ${err.message}`);
  }
}

async function deleteMarker(id) {
  if (!confirm("Вы уверены, что хотите удалить эту метку?")) return;
  try {
    await fetch(`${API_BASE}/api/markers/${id}`, { method: 'DELETE' });
    await loadMarkers();
    toast("✅ Метка удалена");
  } catch (err) {
    toast(`❌ Ошибка удаления: ${err.message}`);
  }
}

function openEditModal(marker) {
  markerToEdit.value = marker;
  modalCoords.value = { lat: marker.coords[0], lng: marker.coords[1] };
  showModal.value = true;
}

function closeModal() {
  showModal.value = false;
  modalCoords.value = null;
  markerToEdit.value = null;
}

function setActiveLayer(layer) {
  activeLayer.value = layer;
  applyFilters();
}

function onMapDblClick(e) {
  if (isMobile.value) return;
  const lat = Math.round(e.latlng.lat);
  const lng = Math.round(e.latlng.lng);
  modalCoords.value = { lat, lng };
  markerToEdit.value = null;
  showModal.value = true;
}

function selectAllGameFilters() { gameFilters.value = gameTypes.map(t => t.id); applyFilters(); }
function clearAllGameFilters() { gameFilters.value = []; applyFilters(); }
function selectAllAlchemyFilters() { alchemyFilters.value = alchemyTypes.map(t => t.id); applyFilters(); }
function clearAllAlchemyFilters() { alchemyFilters.value = []; applyFilters(); }

onMounted(() => {
  window.addEventListener("resize", handleResize);
  handleResize();

  const img = new Image();
  img.src = BASE + "5171-0-1480272622.webp";
  img.onload = () => {
    const boundsH = 1000;
    const boundsW = (img.naturalWidth / img.naturalHeight) * boundsH;
    const imageBounds = [[0, 0], [boundsH, boundsW]];

    mapInstance = L.map("map", { 
      crs: L.CRS.Simple, 
      minZoom: -10, 
      maxZoom: 10, 
      zoomSnap: 0.25, 
      attributionControl: false, 
      doubleClickZoom: false,
      zoomControl: false 
    });
    L.imageOverlay(BASE + "5171-0-1480272622.webp", imageBounds).addTo(mapInstance);
    mapInstance.fitBounds(imageBounds);
    
    mapInstance.on("dblclick", onMapDblClick);

    loadMarkers();
    updateInterval = setInterval(loadMarkers, 30000);
  };
});

onBeforeUnmount(() => {
  if (updateInterval) clearInterval(updateInterval);
  window.removeEventListener("resize", handleResize);
  if (mapInstance) mapInstance.remove();
});
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Cinzel:wght@600;700&family=Crimson+Text:ital@0;1&display=swap");

.leaflet-map { width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; z-index: 1; }

/* ================= ЕДИНАЯ ПАНЕЛЬ УПРАВЛЕНИЯ ================= */
.top-bar {
  position: fixed;
  top: 20px;
  left: 20px;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  background: #1e1e1e;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  pointer-events: auto;
  width: 300px;
  box-sizing: border-box;
}

.logo {
  height: 24px;
  width: auto;
  object-fit: contain;
}

.divider {
  width: 1px;
  height: 20px;
  background: rgba(255, 255, 255, 0.15);
}

.layer-switch {
  display: flex;
  gap: 4px;
  flex: 1;
  justify-content: center;
}

.layer-switch button {
  background: transparent;
  border: none;
  color: #a0a0a0;
  padding: 6px 8px;
  border-radius: 6px;
  font: bold 12px "Cinzel", serif;
  cursor: pointer;
  transition: color 0.2s, background 0.2s;
  white-space: nowrap;
}

.layer-switch button:hover {
  color: #e8c8a0;
}

.layer-switch button.active {
  background: rgba(232, 200, 160, 0.15);
  color: #e8c8a0;
}

.actions {
  display: flex;
  gap: 4px;
}

.icon-btn {
  background: transparent;
  border: none;
  color: #e8c8a0;
  width: 32px;
  height: 32px;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
  padding: 0;
}

.icon-btn:hover {
  background: rgba(255, 255, 255, 0.1);
}

.icon-img {
  width: 22px;
  height: 22px;
  object-fit: contain;
  filter: brightness(0) invert(1);
}

/* ================= ПАНЕЛЬ ФИЛЬТРОВ ================= */
.panel {
  position: fixed;
  top: 80px;
  left: 20px;
  bottom: 20px;
  width: 300px;
  z-index: 999;
  background: #1e1e1e;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  display: flex;
  flex-direction: column;
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  font-family: "Cinzel", serif;
  font-weight: bold;
  color: #e8c8a0;
  font-size: 16px;
}

.close-btn {
  background: none;
  border: none;
  color: #a0a0a0;
  font-size: 18px;
  cursor: pointer;
}

.panel-content {
  flex: 1;
  overflow-y: auto;
  padding: 16px 20px;
}

.panel-content::-webkit-scrollbar { width: 6px; }
.panel-content::-webkit-scrollbar-track { background: transparent; }
.panel-content::-webkit-scrollbar-thumb { background: rgba(255, 255, 255, 0.1); border-radius: 3px; }
.panel-content::-webkit-scrollbar-thumb:hover { background: rgba(255, 255, 255, 0.2); }

.section {
  margin-bottom: 24px;
}

.section-title {
  font: bold 12px "Cinzel", serif;
  letter-spacing: 1px;
  color: #a0a0a0;
  margin-bottom: 12px;
  text-transform: uppercase;
}

.filter-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  margin-bottom: 12px;
}

/* ================= КНОПКИ ЗВЕРЕЙ ================= */
.filter-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 4px;
  background: transparent;
  border: none;
  cursor: pointer;
  opacity: 0.5;
  transition: opacity 0.2s;
  position: relative;
}

.filter-item:hover {
  opacity: 1;
}

.filter-item:hover span {
  color: #ffffff;
}

.filter-item.active {
  opacity: 1;
}

.filter-item.active::before {
  content: "•";
  position: absolute;
  left: -12px;
  top: 50%;
  transform: translateY(-50%);
  color: #e8c8a0;
  font-size: 24px;
  line-height: 1;
}

.filter-item.active span {
  color: #e8c8a0;
}

.filter-item input { display: none; }

.filter-item img {
  width: 24px;
  height: 24px;
  object-fit: contain;
  filter: brightness(0) invert(1);
  opacity: 0.6;
  transition: opacity 0.3s;
}

.filter-item:hover img,
.filter-item.active img {
  opacity: 1;
}

.filter-item span {
  font-family: "Crimson Text", serif;
  font-size: 14px;
  color: #a0a0a0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  transition: color 0.3s;
}

/* ================= КНОПКИ ВСЕ / НИЧЕГО ================= */
.filter-btns {
  display: flex;
  gap: 8px;
}

.filter-btns button {
  flex: 1;
  padding: 8px;
  background: transparent;
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 6px;
  font: bold 11px "Cinzel", serif;
  color: #a0a0a0;
  cursor: pointer;
  transition: background 0.3s, color 0.3s, border-color 0.3s;
}

.filter-btns button:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #e0e0e0;
  border-color: rgba(255, 255, 255, 0.3);
}

.panel-footer {
  padding: 12px;
  text-align: center;
  font-size: 12px;
  color: #666;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  font-family: "Crimson Text", serif;
}

/* ================= МОБИЛЬНАЯ ВЕРСИЯ ================= */
@media (max-width: 1024px) {
  .top-bar {
    right: 20px;
    width: auto;
  }
  .panel {
    top: 80px;
    left: 20px;
    right: 20px;
    width: auto;
    transform: translateY(120%);
  }
  .panel.panel-open {
    transform: translateY(0);
  }
  .logo, .divider { display: none; }
  .layer-switch { flex: 1; justify-content: center; }
}

/* ================= ОСТАЛЬНОЕ ================= */
.loading-indicator {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1001;
  background: #1e1e1e;
  padding: 8px 16px;
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.15);
}
.spinner {
  width: 20px; height: 20px;
  border: 2px solid rgba(255, 255, 255, 0.1);
  border-top: 2px solid #e8c8a0;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}
@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

.toast {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 2000;
  background: #1e1e1e;
  color: #e8c8a0;
  padding: 12px 24px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  font: bold 14px "Cinzel", serif;
}
.fade-enter-active, .fade-leave-active { transition: opacity 0.2s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>

<style>
/* ================= МАРКЕРЫ НА КАРТЕ (БЕЗ ЭФФЕКТОВ И ТЕНЕЙ) ================= */
/* Дичь (Красный) */
.map-marker {
  width: 36px; height: 36px;
  background: #b91c1c;
  border: 3px solid #ffffff;
  border-radius: 50%;
  display: flex; align-items: center; justify-content: center;
  cursor: pointer;
}
.map-marker img { 
  width: 20px; height: 20px; 
  filter: brightness(0) invert(1); 
}

/* Растения (Синий) */
.map-marker-alchemy {
  width: 36px; height: 36px;
  background: #2563eb;
  border: 3px solid #ffffff;
  border-radius: 50%;
  display: flex; align-items: center; justify-content: center;
  cursor: pointer;
}
.map-marker-alchemy img { 
  width: 20px; height: 20px; 
  filter: brightness(0) invert(1); 
}

/* Стили для Popup (БЕЗ ТЕНЕЙ И ИСКАЖЕНИЙ) */
.custom-popup .leaflet-popup-content-wrapper {
  background: #1e1e1e;
  color: #e0e0e0;
  border: 2px solid rgba(255, 255, 255, 0.15);
  border-radius: 8px;
  padding: 0;
  box-shadow: none;
}
.custom-popup .leaflet-popup-tip {
  background: #1e1e1e;
  border: 2px solid rgba(255, 255, 255, 0.15);
  box-shadow: none;
}
.custom-popup .leaflet-popup-tip-container {
  overflow: visible;
}
.custom-popup .leaflet-popup-content {
  margin: 12px 16px;
  font-family: "Crimson Text", serif;
  font-size: 14px;
}
.marker-popup-content { display: flex; align-items: flex-start; gap: 12px; }
.marker-info { flex: 1; max-width: 200px; line-height: 1.4; }
.marker-info b { font-family: "Cinzel", serif; font-size: 14px; color: #e8c8a0; display: block; margin-bottom: 4px; }
.marker-info span { color: #a0a0a0; font-style: italic; }
.marker-actions { display: flex; flex-direction: column; gap: 6px; }
.tooltip-btn {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 6px;
  color: #e0e0e0;
  cursor: pointer;
  padding: 6px;
  font-size: 14px;
  transition: background 0.2s, color 0.2s;
}
.tooltip-btn:hover { background: rgba(255, 255, 255, 0.15); color: #e8c8a0; }
</style>