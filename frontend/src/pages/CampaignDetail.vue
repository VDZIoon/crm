<!--
  IOON — Detalle de campaña para Frappe CRM
  Ubicación en el fork: frontend/src/pages/CampaignDetail.vue
  Ruta: /crm/campaigns/:campaignId

  Lista los clientes potenciales de la campaña con sus campos de seguimiento
  editables en la tabla. Si la campaña ya tiene oportunidades, las muestra debajo.

  ANTES DE COMPILAR: confirma los nombres reales de tus campos personalizados
  de CRM Lead y ajusta CAMPOS_SEGUIMIENTO.
-->
<script setup>
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import { createResource, createListResource, Button } from 'frappe-ui'
import TablaEditable from '@/components/ListViews/TablaEditable.vue'

const props = defineProps({
  campaignId: { type: String, required: true },
})

const router = useRouter()
const CAMPO_CAMPANA = 'custom_campaña'

// ---------------------------------------------------------------------------
// Campos personalizados de seguimiento en CRM Lead (ajusta a tus fieldnames)
// ---------------------------------------------------------------------------
const CAMPOS_SEGUIMIENTO = {
  intentos: 'custom_intentos_de_contacto',
  ultimo: 'custom_último_intento',
  canal: 'custom_canal_del_último_intento',
  resultado: 'custom_resultado_del_último_intento',
  comentarios: 'custom_comentarios_de_marketing',
}

const OPCIONES_CANAL = ['Llamada', 'WhatsApp', 'Correo', 'Visita']
const OPCIONES_RESULTADO = [
  'Contestó',
  'No contestó',
  'Buzón',
  'Número equivocado',
  'Volver a llamar',
]

// ---------------------------------------------------------------------------
// Datos
// ---------------------------------------------------------------------------
const campana = createResource({
  url: 'frappe.client.get',
  params: { doctype: 'CRM Campaign', name: props.campaignId },
  auto: true,
})

const leads = createListResource({
  doctype: 'CRM Lead',
  fields: [
    'name',
    'lead_name',
    'mobile_no',
    'status',
    'lead_owner',
    ...Object.values(CAMPOS_SEGUIMIENTO),
  ],
  filters: { [CAMPO_CAMPANA]: props.campaignId },
  orderBy: 'modified desc',
  pageLength: 500,
  auto: true,
})

const deals = createListResource({
  doctype: 'CRM Deal',
  fields: [
    'name',
    'organization',
    'status',
    'expected_deal_value',
    'deal_value',
    'probability',
    'expected_closure_date',
    'deal_owner',
    'currency',
  ],
  filters: { [CAMPO_CAMPANA]: props.campaignId },
  orderBy: 'modified desc',
  pageLength: 500,
  auto: true,
})

const estadosLead = createListResource({
  doctype: 'CRM Lead Status',
  fields: ['name'],
  orderBy: 'position asc',
  pageLength: 100,
  auto: true,
})

const estadosDeal = createListResource({
  doctype: 'CRM Deal Status',
  fields: ['name'],
  orderBy: 'position asc',
  pageLength: 100,
  auto: true,
})

// ---------------------------------------------------------------------------
// Formato
// ---------------------------------------------------------------------------
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

function formatoFechaHora(valor) {
  if (!valor) return ''
  const fecha = new Date(String(valor).replace(' ', 'T'))
  if (isNaN(fecha)) return String(valor)
  return fecha.toLocaleString('es-NI', { dateStyle: 'short', timeStyle: 'short' })
}

// ---------------------------------------------------------------------------
// Columnas
// ---------------------------------------------------------------------------
const columnasLeads = computed(() => [
  { key: 'lead_name', label: 'Nombre', link: true, width: 'minmax(11rem, 1.4fr)' },
  { key: 'mobile_no', label: 'Móvil', width: 'minmax(9rem, 1fr)' },
  {
    key: 'status',
    label: 'Estado',
    editable: 'Select',
    options: (estadosLead.data || []).map((e) => e.name),
    width: 'minmax(9rem, 1fr)',
  },
  { key: CAMPOS_SEGUIMIENTO.intentos, label: 'Intentos', editable: 'Int', width: 'minmax(5rem, 0.6fr)' },
  {
    key: CAMPOS_SEGUIMIENTO.ultimo,
    label: 'Último intento',
    format: formatoFechaHora,
    width: 'minmax(9rem, 1fr)',
  },
  {
    key: CAMPOS_SEGUIMIENTO.canal,
    label: 'Canal',
    editable: 'Select',
    options: OPCIONES_CANAL,
    width: 'minmax(8rem, 0.9fr)',
  },
  {
    key: CAMPOS_SEGUIMIENTO.resultado,
    label: 'Resultado',
    editable: 'Select',
    options: OPCIONES_RESULTADO,
    width: 'minmax(10rem, 1fr)',
  },
  {
    key: CAMPOS_SEGUIMIENTO.comentarios,
    label: 'Comentarios de marketing',
    editable: 'Data',
    width: 'minmax(14rem, 2fr)',
  },
])

const columnasDeals = computed(() => [
  { key: 'organization', label: 'Organización', link: true, width: 'minmax(12rem, 1.4fr)' },
  {
    key: 'status',
    label: 'Estado',
    editable: 'Select',
    options: (estadosDeal.data || []).map((e) => e.name),
    width: 'minmax(9rem, 1fr)',
  },
  {
    key: 'expected_deal_value',
    label: 'Valor esperado',
    editable: 'Currency',
    format: (v, f) => formatoMoneda(v, f.currency),
    width: 'minmax(8rem, 1fr)',
  },
  {
    key: 'deal_value',
    label: 'Valor final',
    editable: 'Currency',
    format: (v, f) => formatoMoneda(v, f.currency),
    width: 'minmax(8rem, 1fr)',
  },
  { key: 'probability', label: 'Probabilidad (%)', editable: 'Percent', width: 'minmax(8rem, 0.8fr)' },
  { key: 'expected_closure_date', label: 'Cierre estimado', editable: 'Date', width: 'minmax(9rem, 1fr)' },
  { key: 'deal_owner', label: 'Responsable', width: 'minmax(10rem, 1fr)' },
])

// ---------------------------------------------------------------------------
// Resúmenes
// ---------------------------------------------------------------------------
const resumenLeads = computed(() => {
  const filas = leads.data || []
  const contactados = filas.filter((l) => Number(l[CAMPOS_SEGUIMIENTO.intentos]) > 0).length
  return { cantidad: filas.length, contactados, sinContactar: filas.length - contactados }
})

const resumenDeals = computed(() => {
  const filas = deals.data || []
  const valor = filas.reduce(
    (suma, d) => suma + (Number(d.deal_value) || Number(d.expected_deal_value) || 0),
    0,
  )
  return { cantidad: filas.length, ganadas: filas.filter((d) => d.status === 'Won').length, valor }
})

// ---------------------------------------------------------------------------
// Navegación
// ---------------------------------------------------------------------------
function abrirLead(fila) {
  router.push({ name: 'Lead', params: { leadId: fila.name } })
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
        class="mb-6 flex flex-wrap gap-x-8 gap-y-2 text-base text-ink-gray-6"
      >
        <span>Segmento: <strong class="font-medium text-ink-gray-9">{{ campana.data.segmento || '—' }}</strong></span>
        <span>Estado: <strong class="font-medium text-ink-gray-9">{{ campana.data.estado || '—' }}</strong></span>
        <span v-if="campana.data.fecha_inicio || campana.data.fecha_fin">
          Plazo:
          <strong class="font-medium text-ink-gray-9">
            {{ campana.data.fecha_inicio || '—' }} → {{ campana.data.fecha_fin || '—' }}
          </strong>
        </span>
      </section>

      <!-- Clientes potenciales -->
      <section class="mb-8">
        <div class="mb-3 flex flex-wrap items-baseline gap-x-8 gap-y-1">
          <h2 class="text-base font-semibold text-ink-gray-9">Clientes potenciales</h2>
          <span class="text-base text-ink-gray-6">{{ resumenLeads.cantidad }} en total</span>
          <span class="text-base text-ink-gray-6">{{ resumenLeads.contactados }} con intentos</span>
          <span class="text-base text-ink-gray-6">{{ resumenLeads.sinContactar }} sin contactar</span>
        </div>

        <TablaEditable
          v-if="leads.data?.length"
          doctype="CRM Lead"
          :filas="leads.data"
          :columnas="columnasLeads"
          ancho-minimo="72rem"
          @abrir="abrirLead"
        />
        <p v-else-if="leads.error" class="text-base text-ink-red-4">
          No se pudieron cargar los clientes potenciales. Revisa que los nombres de
          CAMPOS_SEGUIMIENTO coincidan con tus campos de CRM Lead.
        </p>
        <p v-else-if="!leads.loading" class="text-base text-ink-gray-5">
          Esta campaña aún no tiene clientes potenciales asignados.
        </p>
      </section>

      <!-- Oportunidades: solo si existen -->
      <section v-if="deals.data?.length">
        <div class="mb-3 flex flex-wrap items-baseline gap-x-8 gap-y-1">
          <h2 class="text-base font-semibold text-ink-gray-9">Oportunidades</h2>
          <span class="text-base text-ink-gray-6">{{ resumenDeals.cantidad }} en total</span>
          <span class="text-base text-ink-gray-6">{{ resumenDeals.ganadas }} ganadas</span>
          <span class="text-base text-ink-gray-6">Valor: {{ formatoMoneda(resumenDeals.valor) || '0' }}</span>
        </div>

        <TablaEditable
          doctype="CRM Deal"
          :filas="deals.data"
          :columnas="columnasDeals"
          @abrir="abrirDeal"
        />
      </section>
    </div>
  </div>
</template>