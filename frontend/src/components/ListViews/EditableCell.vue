<!--
  IOON — Celda editable para tablas de Frappe CRM
  Ubicación en el fork: frontend/src/components/ListViews/EditableCell.vue

  Guarda con frappe.client.set_value, que respeta los permisos del usuario.
  Tipos: Data, Int, Float, Currency, Percent, Select, Date, Small Text.
  Para campos Link (por ejemplo, el estado de un deal) usa type="Select"
  y pásale las opciones cargadas desde su doctype.
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
  // Texto a mostrar cuando no se está editando (por ejemplo, un valor con formato)
  display: { type: String, default: '' },
})

const emit = defineEmits(['updated'])

const editando = ref(false)
const guardando = ref(false)
const error = ref('')
const valor = ref(props.value)
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

watch(
  () => props.value,
  (nuevo) => {
    if (!editando.value) valor.value = nuevo
  },
)

async function abrir() {
  valor.value = props.value
  error.value = ''
  editando.value = true
  await nextTick()
  campo.value?.focus()
}

function cancelar() {
  valor.value = props.value
  editando.value = false
}

async function guardar() {
  if (guardando.value) return
  let nuevo = valor.value
  if (props.type === 'Int') nuevo = nuevo === '' ? 0 : parseInt(nuevo, 10)
  if (['Float', 'Currency', 'Percent'].includes(props.type)) {
    nuevo = nuevo === '' ? 0 : parseFloat(nuevo)
  }

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
  <!-- @click.stop evita que el clic abra el registro de la fila -->
  <div class="w-full" @click.stop @dblclick.stop>
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
        v-else-if="type === 'Date'"
        ref="campo"
        v-model="valor"
        type="date"
        class="form-input w-full text-base"
        :disabled="guardando"
        @change="guardar"
        @keydown.esc="cancelar"
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
