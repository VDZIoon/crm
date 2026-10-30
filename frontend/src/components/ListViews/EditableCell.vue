<!--
  IOON — Celda editable para tablas de Frappe CRM
  Ubicación en el fork: frontend/src/components/ListViews/EditableCell.vue

  Guarda con frappe.client.set_value, que respeta los permisos del usuario.
  Tipos: Data, Int, Float, Currency, Percent, Select, Date, Datetime.

  Las filas de las listas del CRM son enlaces (<a>). Por eso cada clic dentro
  de la celda detiene la propagación Y cancela la acción por defecto; si solo
  se detuviera la propagación, el navegador seguiría el enlace al lead.
-->
<script setup>
import { ref, computed, watch, nextTick } from 'vue'
import { call } from 'frappe-ui'

const props = defineProps({
  doctype: { type: String, required: true },
  name: { type: String, required: true },
  fieldname: { type: String, required: true },
  value: { default: '' },
  type: { type: String, default: 'Data' },
  // Arreglo de opciones o texto con una opción por línea
  options: { type: [Array, String], default: () => [] },
  // Texto a mostrar cuando no se está editando
  display: { type: String, default: '' },
})

const emit = defineEmits(['updated'])

const editando = ref(false)
const guardando = ref(false)
const error = ref('')
const valor = ref('')
const campo = ref(null)

const esNumero = computed(() => ['Int', 'Float', 'Currency', 'Percent'].includes(props.type))

const opciones = computed(() =>
  Array.isArray(props.options)
    ? props.options
    : String(props.options || '').split('\n').filter(Boolean),
)

const textoVisible = computed(() => {
  if (props.display) return props.display
  const v = props.value
  return v === '' || v === null || v === undefined ? '' : String(v)
})

// '2026-10-28 14:30:00' <-> '2026-10-28T14:30'
function aInput(v) {
  if (v === null || v === undefined) return ''
  if (props.type === 'Datetime') return String(v).replace(' ', 'T').slice(0, 16)
  if (props.type === 'Date') return String(v).slice(0, 10)
  return v
}

function desdeInput(v) {
  if (props.type === 'Datetime') return v ? v.replace('T', ' ') + ':00' : ''
  if (props.type === 'Int') return v === '' ? 0 : parseInt(v, 10)
  if (['Float', 'Currency', 'Percent'].includes(props.type)) return v === '' ? 0 : parseFloat(v)
  return v
}

watch(
  () => props.value,
  (nuevo) => {
    if (!editando.value) valor.value = aInput(nuevo)
  },
  { immediate: true },
)

// Bloquea la navegación del enlace de la fila
function bloquear(e) {
  e.stopPropagation()
  e.preventDefault()
}

async function abrir() {
  valor.value = aInput(props.value)
  error.value = ''
  editando.value = true
  await nextTick()
  campo.value?.focus()
  if (['Date', 'Datetime'].includes(props.type)) {
    try {
      campo.value?.showPicker?.()
    } catch {
      /* algunos navegadores no lo permiten; el campo queda enfocado */
    }
  }
}

function cancelar() {
  valor.value = aInput(props.value)
  editando.value = false
}

async function guardar() {
  if (guardando.value || !editando.value) return
  const nuevo = desdeInput(valor.value)
  const anterior = props.value ?? ''

  if (String(nuevo ?? '') === String(anterior)) {
    editando.value = false
    return
  }

  guardando.value = true
  error.value = ''
  try {
    await call('frappe.client.set_value', {
      doctype: props.doctype,
      name: props.name,
      fieldname: props.fieldname,
      value: nuevo,
    })
    emit('updated', { fieldname: props.fieldname, value: nuevo })
    editando.value = false
  } catch (e) {
    error.value = 'No se pudo guardar. Revisa el valor o tus permisos.'
  } finally {
    guardando.value = false
  }
}
</script>

<template>
  <div
    class="w-full"
    @click="bloquear"
    @dblclick="bloquear"
    @mousedown.stop
    @dragstart.prevent
  >
    <button
      v-if="!editando"
      type="button"
      class="w-full truncate rounded px-1.5 py-1 text-left text-base text-ink-gray-8 hover:bg-surface-gray-2 focus:outline-none focus-visible:ring-2 focus-visible:ring-outline-gray-3"
      :title="error || 'Clic para editar'"
      @click="abrir"
    >
      <span v-if="textoVisible">{{ textoVisible }}</span>
      <span v-else class="text-ink-gray-4">—</span>
    </button>

    <template v-else>
      <select
        v-if="type === 'Select'"
        ref="campo"
        v-model="valor"
        class="form-select w-full text-base"
        :disabled="guardando"
        @change="guardar"
        @blur="guardar"
        @keydown.esc="cancelar"
      >
        <option value=""></option>
        <option v-for="o in opciones" :key="o" :value="o">{{ __(o) }}</option>
      </select>

      <input
        v-else-if="esNumero"
        ref="campo"
        v-model="valor"
        type="number"
        :step="type === 'Int' ? 1 : 'any'"
        class="form-input w-full text-base"
        :disabled="guardando"
        @keydown.enter="guardar"
        @keydown.esc="cancelar"
        @blur="guardar"
      />

      <input
        v-else-if="type === 'Date' || type === 'Datetime'"
        ref="campo"
        v-model="valor"
        :type="type === 'Date' ? 'date' : 'datetime-local'"
        class="form-input w-full text-base"
        :disabled="guardando"
        @change="type === 'Date' && guardar()"
        @keydown.enter="guardar"
        @keydown.esc="cancelar"
        @blur="guardar"
      />

      <input
        v-else
        ref="campo"
        v-model="valor"
        type="text"
        class="form-input w-full text-base"
        :disabled="guardando"
        @keydown.enter="guardar"
        @keydown.esc="cancelar"
        @blur="guardar"
      />
    </template>

    <p v-if="error" class="mt-1 text-xs text-ink-red-4">{{ error }}</p>
  </div>
</template>
