<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="modal-container">
      <div class="modal-header">
        <span>{{ markerToEdit ? 'Редактировать метку' : 'Добавить метку' }}</span>
        <button class="modal-close" @click="close">✕</button>
      </div>

      <div class="modal-body">
        <div v-if="showCategorySwitch && !markerToEdit" class="category-switch">
          <button :class="{ active: category === 'game' }" @click="category = 'game'">🏹 Дичь</button>
          <button :class="{ active: category === 'alchemy' }" @click="category = 'alchemy'">🌿 Алхимия</button>
        </div>

        <div class="field-label">Тип ресурса</div>
        <div class="type-grid">
          <button
            v-for="type in currentTypeList"
            :key="type.id"
            class="type-btn"
            :class="{ selected: selectedType === type.id }"
            @click="selectedType = type.id"
            :title="type.name"
          >
            <img :src="type.icon" :alt="type.name" />
          </button>
        </div>

        <div class="field-label">Название места</div>
        <input v-model="title" placeholder="Например: Логово волков" @keyup.enter="save" />

        <div class="field-label">Описание</div>
        <textarea v-model="description" placeholder="Что здесь водится или растёт..."></textarea>

        <div class="coords-hint" v-if="coords">
          Координаты: [{{ Math.round(coords.lat) }}, {{ Math.round(coords.lng) }}]
        </div>
      </div>

      <div class="modal-footer">
        <button class="btn-cancel" @click="close">Отмена</button>
        <button class="btn-add" @click="save">{{ markerToEdit ? 'Сохранить' : 'Добавить' }}</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';

const props = defineProps({
  visible: Boolean,
  coords: Object,
  layerMode: String,
  gameTypes: Array,
  alchemyTypes: Array,
  markerToEdit: Object
});

const emit = defineEmits(['close', 'save']);

const category = ref('game');
const selectedType = ref('');
const title = ref('');
const description = ref('');

watch(() => props.visible, (newVal) => {
  if (newVal) {
    if (props.markerToEdit) {
      const isAlchemy = props.alchemyTypes.some(t => t.id === props.markerToEdit.type);
      category.value = isAlchemy ? 'alchemy' : 'game';
      selectedType.value = props.markerToEdit.type;
      title.value = props.markerToEdit.title;
      description.value = props.markerToEdit.description;
    } else {
      category.value = 'game';
      selectedType.value = '';
      title.value = '';
      description.value = '';
    }
  }
});

const showCategorySwitch = computed(() => props.layerMode === 'both');

const currentTypeList = computed(() => {
  if (props.markerToEdit) {
    const isAlchemy = props.alchemyTypes.some(t => t.id === props.markerToEdit.type);
    return isAlchemy ? props.alchemyTypes : props.gameTypes;
  }
  
  if (props.layerMode === 'game') return props.gameTypes;
  if (props.layerMode === 'alchemy') return props.alchemyTypes;
  if (props.layerMode === 'both') {
    return category.value === 'game' ? props.gameTypes : props.alchemyTypes;
  }
  return [];
});

function save() {
  if (!selectedType.value && currentTypeList.value.length) {
    selectedType.value = currentTypeList.value[0].id;
  }
  if (!selectedType.value) return;

  const payload = {
    id: props.markerToEdit?.id,
    coords: props.markerToEdit ? props.markerToEdit.coords : `[${Math.round(props.coords.lat)}, ${Math.round(props.coords.lng)}]`,
    type: selectedType.value,
    title: title.value.trim() || 'Без названия',
    description: description.value.trim(),
  };

  emit('save', payload);
}

function close() {
  emit('close');
}
</script>

<style scoped>
.modal-overlay { 
  position: fixed; 
  top: 0; left: 0; width: 100%; height: 100%; 
  background: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(8px); 
  z-index: 2000; 
  display: flex; 
  align-items: center; 
  justify-content: center; 
}

.modal-container { 
  background: rgba(20, 20, 20, 0.85); 
  backdrop-filter: blur(16px);
  width: 90%; 
  max-width: 420px; 
  border-radius: 16px; 
  border: 1px solid rgba(255, 255, 255, 0.08); 
  box-shadow: 0 20px 40px rgba(0,0,0,0.6); 
  font-family: 'Crimson Text', serif; 
  color: #e0e0e0;
  overflow: hidden;
}

.modal-header { 
  display: flex; 
  justify-content: space-between; 
  align-items: center; 
  padding: 16px 20px; 
  border-bottom: 1px solid rgba(255, 255, 255, 0.05); 
  font-weight: bold; 
  font-family: 'Cinzel', serif; 
  color: #e8c8a0;
  font-size: 16px;
}

.modal-close { 
  background: none; 
  border: none; 
  font-size: 18px; 
  cursor: pointer; 
  color: #a0a0a0; 
  transition: color 0.2s;
}

.modal-close:hover { color: #e8c8a0; }

.modal-body { padding: 20px; }

.category-switch { 
  display: flex; 
  gap: 8px; 
  margin-bottom: 20px; 
  background: rgba(255, 255, 255, 0.03);
  padding: 4px;
  border-radius: 10px;
}

.category-switch button { 
  flex: 1; 
  padding: 10px; 
  background: transparent; 
  border: none; 
  border-radius: 8px; 
  font-family: 'Cinzel', serif; 
  font-weight: bold; 
  color: #a0a0a0;
  cursor: pointer; 
  transition: all 0.2s;
}

.category-switch button:hover {
  color: #e8c8a0;
}

.category-switch button.active { 
  background: rgba(232, 200, 160, 0.15); 
  color: #e8c8a0; 
}

.field-label { 
  font-size: 11px; 
  font-family: 'Cinzel', serif; 
  letter-spacing: 1px;
  text-transform: uppercase;
  margin: 16px 0 8px 0; 
  color: #808080; 
}

.type-grid { 
  display: flex; 
  gap: 8px; 
  flex-wrap: wrap; 
}

.type-btn { 
  width: 44px; 
  height: 44px; 
  background: rgba(255, 255, 255, 0.03); 
  border: none; 
  border-radius: 10px; 
  cursor: pointer; 
  opacity: 0.5; 
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.type-btn:hover {
  background: rgba(255, 255, 255, 0.08);
  opacity: 1;
}

.type-btn img { 
  width: 24px; 
  height: 24px; 
  filter: brightness(0) invert(1);
}

.type-btn.selected { 
  opacity: 1; 
  background: rgba(232, 200, 160, 0.15);
  box-shadow: inset 0 0 0 1px rgba(232, 200, 160, 0.3);
}

input, textarea { 
  width: 100%; 
  padding: 12px; 
  background: rgba(255, 255, 255, 0.05); 
  border: 1px solid rgba(255, 255, 255, 0.08); 
  border-radius: 10px; 
  font-family: inherit; 
  color: #fff;
  font-size: 14px;
  transition: border-color 0.2s;
  outline: none;
}

input:focus, textarea:focus {
  border-color: rgba(232, 200, 160, 0.4);
}

textarea { 
  min-height: 80px; 
  resize: vertical; 
}

.coords-hint { 
  margin-top: 16px; 
  font-size: 12px; 
  color: #808080; 
  text-align: center; 
  background: rgba(255, 255, 255, 0.03); 
  padding: 8px; 
  border-radius: 8px; 
}

.modal-footer { 
  display: flex; 
  gap: 12px; 
  padding: 16px 20px; 
  border-top: 1px solid rgba(255, 255, 255, 0.05); 
  background: rgba(0, 0, 0, 0.2);
}

.modal-footer button { 
  flex: 1; 
  padding: 12px; 
  border-radius: 10px; 
  font-weight: bold; 
  cursor: pointer; 
  font-family: 'Cinzel', serif;
  transition: all 0.2s;
}

.btn-add { 
  background: #e8c8a0; 
  border: none; 
  color: #1a1a1a; 
  box-shadow: 0 4px 15px rgba(232, 200, 160, 0.2);
}

.btn-add:hover {
  background: #f0d5b5;
  transform: translateY(-1px);
}

.btn-cancel { 
  background: transparent; 
  border: 1px solid rgba(255, 255, 255, 0.1); 
  color: #a0a0a0;
}

.btn-cancel:hover {
  background: rgba(255, 255, 255, 0.05);
  color: #e0e0e0;
}
</style>