<!--
  IOON — Tabla con celdas editables para Frappe CRM
  Ubicación en el fork: frontend/src/components/ListViews/TablaEditable.vue

  Cada columna: { key, label, width, editable?, options?, format?, link? }
    editable: tipo de EditableCell (Data, Int, Currency, Percent, Select, Date...)
    options:  opciones para Select
    format:   (valor, fila) => texto a mostrar
    link:     true para mostrarla como enlace que emite "abrir"
-->
<script setup>
import { computed } from 'vue'
import EditableCell from '@/components/ListViews/EditableCell.vue'

const props = defineProps({
  doctype: { type: String, required: true },
  filas: { type: Array, default: () => [] },
  columnas: { type: Array, required: true },
  anchoMinimo: { type: String, default: '56rem' },
})

const emit = defineEmits(['abrir'])

const plantilla = computed(() =>
  props.columnas.map((c) => c.width || 'minmax(8rem, 1fr)').join(' '),
)

function texto(columna, fila) {
  const v = fila[columna.key]
  if (columna.format) return columna.format(v, fila) || ''
  if (columna.editable === 'Select' && v) return __(v)
  return v === null || v === undefined ? '' : String(v)
}

function actualizar(fila, { fieldname, value }) {
  fila[fieldname] = value
}
</script>

<template>
  <div class="overflow-x-auto rounded border border-outline-gray-2">
    <div :style="{ minWidth: anchoMinimo }">
      <div
        class="grid gap-3 border-b border-outline-gray-2 bg-surface-gray-2 px-3 py-2 text-sm text-ink-gray-5"
        :style="{ gridTemplateColumns: plantilla }"
      >
        <span v-for="c in columnas" :key="c.key" class="truncate">{{ c.label }}</span>
      </div>

      <div
        v-for="fila in filas"
        :key="fila.name"
        class="grid items-center gap-3 border-b border-outline-gray-1 px-3 py-1.5 last:border-b-0 hover:bg-surface-gray-1"
        :style="{ gridTemplateColumns: plantilla }"
      >
        <template v-for="c in columnas" :key="c.key">
          <button
            v-if="c.link"
            type="button"
            class="truncate text-left text-base font-medium text-ink-gray-9 hover:underline"
            :title="'Abrir ' + (texto(c, fila) || fila.name)"
            @click="emit('abrir', fila)"
          >
            {{ texto(c, fila) || fila.name }}
          </button>
          <EditableCell
            v-else-if="c.editable"
            :doctype="doctype"
            :name="fila.name"
            :fieldname="c.key"
            :value="fila[c.key]"
            :type="c.editable"
            :options="c.options || []"
            :display="texto(c, fila)"
            @updated="(cambio) => actualizar(fila, cambio)"
          />
          <span v-else class="truncate px-1.5 text-base text-ink-gray-7">
            {{ texto(c, fila) || '—' }}
          </span>
        </template>
      </div>
    </div>
  </div>
</template>
