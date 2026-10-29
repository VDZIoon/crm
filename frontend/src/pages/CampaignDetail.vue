<!--
  IOON — Detalle de campaña para Frappe CRM
  Ubicación en el fork: frontend/src/pages/CampaignDetail.vue
  Ruta: /crm/campaigns/:campaignId

  Muestra las oportunidades (CRM Deal) asociadas a la campaña, con sus
  campos editables directamente en la tabla, y el total de leads.

  ANTES DE COMPILAR: confirma los nombres de campo de CRM Deal en
  /desk/doctype/CRM Deal y ajusta CAMPOS_DEAL si alguno difiere.
-->
<script setup>
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import { createResource, createListResource, Button } from 'frappe-ui'
import EditableCell from '@/components/ListViews/EditableCell.vue'

const props = defineProps({
  campaignId: { type: String, required: true },
})

const router = useRouter()
const CAMPO_CAMPANA = 'custom_campana'

// Columnas de la tabla de oportunidades.
// editable: tipo de EditableCell; omítelo para dejar la columna de solo lectura.
const CAMPOS_DEAL = [
  { key: 'organization', label: 'Organización', width: 'minmax(12rem, 1.4fr)' },
  { key: 'status', label: 'Estado', editable: 'Select', width: 'minmax(9rem, 1fr)' },
  { key: 'expected_deal_value', label: 'Valor esperado', editable: 'Currency', width: 'minmax(8rem, 1fr)' },
  { key: 'deal_value', label: 'Valor final', editable: 'Currency', width: 'minmax(8rem, 1fr)' },
  { key: 'probability', label: 'Probabilidad (%)', editable: 'Percent', width: 'minmax(8rem, 0.8fr)' },
  { key: 'expected_closure_date', label: 'Cierre estimado', editable: 'Date', width: 'minmax(9rem, 1fr)' },
  { key: 'deal_owner', label: 'Responsable', width: 'minmax(10rem, 1fr)' },
]

const campana = createResource({
  url: 'frappe.client.get',
  params: { doctype: 'CRM Campaign', name: props.campaignId },
  auto: true,
})

const deals = createListResource({
  doctype: 'CRM Deal',
  fields: ['name', 'currency', ...CAMPOS_DEAL.map((c) => c.key)],
  filters: { [CAMPO_CAMPANA]: props.campaignId },
  orderBy: 'modified desc',
  pageLength: 500,
  auto: true,
})

const estados = createListResource({
  doctype: 'CRM Deal Status',
  fields: ['name'],
  orderBy: 'position asc',
  pageLength: 100,
  auto: true,
})

const totalLeads = createResource({
  url: 'frappe.client.get_count',
  params: { doctype: 'CRM Lead', filters: { [CAMPO_CAMPANA]: props.campaignId } },
  auto: true,
})

const opcionesEstado = computed(() => (estados.data || []).map((e) => e.name))

const resumen = computed(() => {
  const filas = deals.data || []
  const valor = filas.reduce(
    (suma, d) => suma + (Number(d.deal_value) || Number(d.expected_deal_value) || 0),
    0,
  )
  const ganadas = filas.filter((d) => d.status === 'Won').length
  return { cantidad: filas.length, valor, ganadas }
})

const plantillaColumnas = computed(() => CAMPOS_DEAL.map((c) => c.width).join(' '))

function formatoMoneda(valor, moneda) {
  if (valor === null || valor === undefined || valor === '') return ''
  try {
    return new Intl.NumberFormat('es-NI', {
      style: 'currency',
      currency: moneda || 'NIO',
      maximumFractionDigits: 2,
    }).format(valor)
  } catch {
    return String(valor)
  }
}

function textoCelda(campo, fila) {
  const v = fila[campo.key]
  if (campo.editable === 'Currency') return formatoMoneda(v, fila.currency)
  if (campo.editable === 'Select' && v) return __(v)
  return v === null || v === undefined ? '' : String(v)
}

function actualizarFila(fila, { fieldname, value }) {
  fila[fieldname] = value
}

function abrirDeal(fila) {
  router.push({ name: 'Deal', params: { dealId: fila.name } })
}

function editarCampana() {
  window.open(`/desk/crm-campaign/${encodeURIComponent(props.campaignId)}`, '_blank')
}
</script>

<template>
  <div class="flex h-full flex-col overflow-hidden">
    <header class="flex items-center justify-between gap-3 border-b px-5 py-2.5">
      <div class="flex min-w-0 items-center gap-1.5 text-lg font-medium">
        <router-link :to="{ name: 'Campaigns' }" class="text-ink-gray-5 hover:text-ink-gray-8">
          Campañas
        </router-link>
        <span class="text-ink-gray-4">/</span>
        <span class="truncate text-ink-gray-9">{{ campaignId }}</span>
      </div>
      <Button label="Editar campaña" @click="editarCampana" />
    </header>

    <div class="flex-1 overflow-auto px-5 py-4">
      <section
        v-if="campana.data"
        class="mb-5 flex flex-wrap gap-x-8 gap-y-2 text-base text-ink-gray-6"
      >
        <span>Segmento: <strong class="font-medium text-ink-gray-9">{{ campana.data.segmento || '—' }}</strong></span>
        <span>Estado: <strong class="font-medium text-ink-gray-9">{{ campana.data.estado || '—' }}</strong></span>
        <span>
          Plazo:
          <strong class="font-medium text-ink-gray-9">
            {{ campana.data.fecha_inicio || '—' }} → {{ campana.data.fecha_fin || '—' }}
          </strong>
        </span>
        <span>Leads: <strong class="font-medium text-ink-gray-9">{{ totalLeads.data ?? 0 }}</strong></span>
      </section>

      <section class="mb-4 flex flex-wrap items-baseline gap-x-8 gap-y-1">
        <h2 class="text-base font-semibold text-ink-gray-9">Oportunidades</h2>
        <span class="text-base text-ink-gray-6">{{ resumen.cantidad }} en total</span>
        <span class="text-base text-ink-gray-6">{{ resumen.ganadas }} ganadas</span>
        <span class="text-base text-ink-gray-6">Valor: {{ formatoMoneda(resumen.valor) || '0' }}</span>
      </section>

      <div v-if="deals.data?.length" class="overflow-x-auto rounded border border-outline-gray-2">
        <div class="min-w-[56rem]">
          <div
            class="grid gap-3 border-b border-outline-gray-2 bg-surface-gray-2 px-3 py-2 text-sm text-ink-gray-5"
            :style="{ gridTemplateColumns: plantillaColumnas }"
          >
            <span v-for="campo in CAMPOS_DEAL" :key="campo.key">{{ campo.label }}</span>
          </div>

          <div
            v-for="fila in deals.data"
            :key="fila.name"
            class="grid items-center gap-3 border-b border-outline-gray-1 px-3 py-1.5 last:border-b-0 hover:bg-surface-gray-1"
            :style="{ gridTemplateColumns: plantillaColumnas }"
          >
            <template v-for="campo in CAMPOS_DEAL" :key="campo.key">
              <EditableCell
                v-if="campo.editable"
                doctype="CRM Deal"
                :name="fila.name"
                :fieldname="campo.key"
                :value="fila[campo.key]"
                :type="campo.editable"
                :options="campo.key === 'status' ? opcionesEstado : []"
                :display="textoCelda(campo, fila)"
                @updated="(cambio) => actualizarFila(fila, cambio)"
              />
              <button
                v-else-if="campo.key === 'organization'"
                type="button"
                class="truncate text-left text-base font-medium text-ink-gray-9 hover:underline"
                :title="'Abrir ' + (fila.organization || fila.name)"
                @click="abrirDeal(fila)"
              >
                {{ fila.organization || fila.name }}
              </button>
              <span v-else class="truncate px-1.5 text-base text-ink-gray-7">
                {{ textoCelda(campo, fila) || '—' }}
              </span>
            </template>
          </div>
        </div>
      </div>

      <p v-else-if="deals.error" class="text-base text-ink-red-4">
        No se pudieron cargar las oportunidades. Revisa que CRM Deal tenga el campo
        {{ CAMPO_CAMPANA }} y que los nombres de CAMPOS_DEAL existan.
      </p>

      <p v-else-if="!deals.loading" class="text-base text-ink-gray-5">
        Esta campaña aún no tiene oportunidades. Asigna la campaña a una oportunidad,
        o conviértele un lead que ya la tenga, para verla aquí.
      </p>
    </div>
  </div>
</template>
