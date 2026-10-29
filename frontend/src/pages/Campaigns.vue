<!--
  IOON — Página de Campañas para Frappe CRM
  Ubicación sugerida en el fork: frontend/src/pages/Campaigns.vue

  Lista los registros de CRM Campaign. Al hacer clic en una campaña,
  abre su formulario en el escritorio (crear y editar campañas sigue
  haciéndose ahí). El conteo de leads usa el campo custom_campana.
-->
<script setup>
import { computed } from 'vue'
import { createListResource, createResource, ListView, Badge, Button } from 'frappe-ui'

const CAMPO_CAMPANA = 'custom_campana'

const campanas = createListResource({
  doctype: 'CRM Campaign',
  fields: ['name', 'segmento', 'estado', 'fecha_inicio', 'fecha_fin', 'responsable'],
  orderBy: 'fecha_inicio desc',
  pageLength: 200,
  auto: true,
})

// Cantidad de leads por campaña
const conteo = createResource({
  url: 'frappe.client.get_list',
  params: {
    doctype: 'CRM Lead',
    fields: [CAMPO_CAMPANA, 'count(name) as total'],
    filters: [[CAMPO_CAMPANA, 'is', 'set']],
    group_by: CAMPO_CAMPANA,
    limit_page_length: 0,
  },
  auto: true,
})

const totales = computed(() => {
  const mapa = {}
  for (const fila of conteo.data || []) mapa[fila[CAMPO_CAMPANA]] = fila.total
  return mapa
})

const columnas = [
  { label: 'Campaña', key: 'name', width: '16rem' },
  { label: 'Segmento', key: 'segmento', width: '12rem' },
  { label: 'Estado', key: 'estado', width: '8rem' },
  { label: 'Inicio', key: 'fecha_inicio', width: '8rem' },
  { label: 'Fin', key: 'fecha_fin', width: '8rem' },
  { label: 'Leads', key: 'leads', width: '6rem' },
]

const filas = computed(() =>
  (campanas.data || []).map((c) => ({ ...c, leads: totales.value[c.name] || 0 })),
)

const temaEstado = { Activa: 'green', Planificada: 'gray', Cerrada: 'red' }

function abrirCampana(fila) {
  window.open(`/desk/crm-campaign/${encodeURIComponent(fila.name)}`, '_blank')
}

function nuevaCampana() {
  window.open('/desk/crm-campaign/new', '_blank')
}
</script>

<template>
  <div class="flex h-full flex-col overflow-hidden">
    <header class="flex items-center justify-between border-b px-5 py-2.5">
      <h1 class="text-lg font-medium text-ink-gray-9">Campañas</h1>
      <Button variant="solid" label="Crear campaña" @click="nuevaCampana" />
    </header>

    <div class="flex-1 overflow-auto px-5 py-3">
      <ListView
        v-if="filas.length"
        :columns="columnas"
        :rows="filas"
        row-key="name"
        :options="{ selectable: false, showTooltip: false, onRowClick: abrirCampana }"
      >
        <template #cell="{ column, row, item }">
          <Badge
            v-if="column.key === 'estado' && item"
            :label="item"
            :theme="temaEstado[item] || 'gray'"
            variant="subtle"
          />
          <span v-else class="truncate text-base text-ink-gray-8">{{ item ?? '' }}</span>
        </template>
      </ListView>

      <div
        v-else-if="!campanas.loading"
        class="flex h-full flex-col items-center justify-center gap-3 text-ink-gray-5"
      >
        <p class="text-base">Todavía no hay campañas. Crea la primera para empezar a segmentar tus leads.</p>
        <Button label="Crear campaña" @click="nuevaCampana" />
      </div>
    </div>
  </div>
</template>
