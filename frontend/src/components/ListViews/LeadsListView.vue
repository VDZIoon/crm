<template>
  <ListView
    :class="$attrs.class"
    :columns="columns"
    :rows="rows"
    :options="{
      getRowRoute: (row) => ({
        name: 'Lead',
        params: { leadId: row.name },
        query: { view: route.query.view, viewType: route.params.viewType },
      }),
      selectable: options.selectable,
      showTooltip: options.showTooltip,
      resizeColumn: options.resizeColumn,
    }"
    row-key="name"
    @update:selections="(selections) => emit('selectionsChanged', selections)"
  >
    <ListHeader
      class="sm:mx-5 mx-3"
      @columnWidthUpdated="emit('columnWidthUpdated')"
    >
      <ListHeaderItem
        v-for="column in columns"
        :key="column.key"
        :item="column"
        @columnWidthUpdated="(e) => onColumnWidthUpdated(e, column)"
      >
        <Button
          v-if="column.key == '_liked_by'"
          variant="ghost"
          class="!h-4"
          @click="() => emit('applyLikeFilter')"
        >
          <HeartIcon
            class="h-4 w-4"
            :class="isLikeFilterApplied ? 'fill-red-500 text-red-500' : ''"
          />
        </Button>
      </ListHeaderItem>
    </ListHeader>
    <ListRows
      v-slot="{ idx, column, item, row, isVisited }"
      :rows="rows"
      doctype="CRM Lead"
    >
      <ListRowItem :item="item" :align="column.align" class="overflow-hidden">
        <template #prefix>
          <div
            v-if="column.key === '_assign'"
            class="flex items-center truncate"
          >
            <MultipleAvatar
              :avatars="item"
              size="sm"
              :label-class="
                isVisited ? 'text-ink-gray-6' : 'font-medium text-ink-gray-9'
              "
              @click="
                (event) =>
                  emit('applyFilter', {
                    event,
                    idx,
                    column,
                    item,
                    firstColumn: columns[0],
                  })
              "
            />
          </div>
          <div v-else-if="column.key === 'status'">
            <IndicatorIcon :class="item.color" />
          </div>
          <div v-else-if="column.key === 'lead_name'">
            <Avatar
              v-if="item.label"
              class="flex items-center"
              :image="item.image"
              :label="item.image_label"
              size="sm"
            />
          </div>
          <div v-else-if="column.key === 'lead_owner'">
            <Avatar
              v-if="item.full_name"
              class="flex items-center"
              :image="item.user_image"
              :label="item.full_name"
              size="sm"
            />
          </div>
          <div v-else-if="column.key === 'mobile_no' && item">
            <PhoneIcon class="h-4 w-4" />
          </div>
        </template>
        <template #default="{ label }">
          <!-- IOON: cualquier columna editable se edita en la celda -->
          <EditableCell
            v-if="esEditable(column)"
            doctype="CRM Lead"
            :name="row.name"
            :fieldname="column.key"
            :value="valorCrudo(row[column.key])"
            :type="tipoEditable(column)"
            :options="opcionesEditables(column)"
            :display="textoVisible(label, column)"
            @updated="(cambio) => alActualizar(row, column, cambio)"
          />
          <div
            v-else-if="
              [
                'modified',
                'creation',
                'first_response_time',
                'first_responded_on',
                'response_by',
              ].includes(column.key)
            "
            class="truncate text-base"
            :class="
              isVisited ? 'text-ink-gray-6' : 'font-medium text-ink-gray-9'
            "
            @click="
              (event) =>
                emit('applyFilter', {
                  event,
                  idx,
                  column,
                  item,
                  firstColumn: columns[0],
                })
            "
          >
            <Tooltip :text="item.label">
              <div>{{ item.timeAgo }}</div>
            </Tooltip>
          </div>
          <div v-else-if="column.key === '_liked_by'">
            <Button
              variant="ghost"
              @click.stop.prevent="
                () =>
                  emit('likeDoc', {
                    name: row.name,
                    liked: isLiked(item),
                  })
              "
            >
              <HeartIcon
                class="h-4 w-4"
                :class="
                  isLiked(item)
                    ? isVisited
                      ? 'fill-red-400 text-red-400'
                      : 'fill-red-500 text-red-500'
                    : isVisited
                      ? 'text-ink-gray-6'
                      : 'text-ink-gray-9'
                "
              />
            </Button>
          </div>
          <div
            v-else-if="column.key === 'sla_status'"
            class="truncate text-base"
          >
            <Badge
              v-if="item.value"
              :variant="'subtle'"
              :theme="item.color"
              size="md"
              :label="item.value"
              @click="
                (event) =>
                  emit('applyFilter', {
                    event,
                    idx,
                    column,
                    item,
                    firstColumn: columns[0],
                  })
              "
            />
          </div>
          <div v-else-if="column.type === 'Check'">
            <FormControl
              type="checkbox"
              :modelValue="item"
              :disabled="true"
              class="text-ink-gray-9"
            />
          </div>
          <RatingInput
            v-else-if="column.type === 'Rating'"
            :value="item"
            class="!opacity-100 flex-nowrap overflow-auto"
            :disabled="true"
            :max="column.options || 5"
            @click="
              (event) =>
                emit('applyFilter', {
                  event,
                  idx,
                  column,
                  item,
                  firstColumn: columns[0],
                })
            "
          />
          <div
            v-else-if="label"
            class="truncate text-base"
            :class="
              isVisited ? 'text-ink-gray-6' : 'font-medium text-ink-gray-9'
            "
            @click="
              (event) =>
                emit('applyFilter', {
                  event,
                  idx,
                  column,
                  item,
                  firstColumn: columns[0],
                })
            "
          >
            {{ getLabel(label, column) }}
          </div>
        </template>
      </ListRowItem>
    </ListRows>
    <ListSelectBanner>
      <template #actions="{ selections, unselectAll }">
        <Dropdown
          :options="listBulkActionsRef.bulkActions(selections, unselectAll)"
        >
          <Button icon="lucide-more-horizontal" variant="ghost" />
        </Dropdown>
      </template>
    </ListSelectBanner>
  </ListView>
  <ListFooter
    v-if="pageLengthCount"
    v-model="pageLengthCount"
    class="border-t sm:px-5 px-3 py-2"
    :options="{
      rowCount: options.rowCount,
      totalCount: options.totalCount,
    }"
    @loadMore="emit('loadMore')"
  />
  <ListBulkActions ref="listBulkActionsRef" v-model="list" doctype="CRM Lead" />
</template>

<script setup>
import HeartIcon from '@/components/Icons/HeartIcon.vue'
import IndicatorIcon from '@/components/Icons/IndicatorIcon.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import RatingInput from '@/components/Controls/RatingInput.vue'
import MultipleAvatar from '@/components/MultipleAvatar.vue'
import ListBulkActions from '@/components/ListBulkActions.vue'
import ListRows from '@/components/ListViews/ListRows.vue'
import EditableCell from '@/components/ListViews/EditableCell.vue'
import { isTranslatable, formatDuration } from '@/utils'
import {
  Avatar,
  ListView,
  ListHeader,
  ListHeaderItem,
  ListSelectBanner,
  ListRowItem,
  ListFooter,
  Dropdown,
  Tooltip,
  createListResource,
} from 'frappe-ui'
import { sessionStore } from '@/stores/session'
import { ref, computed, watch, shallowReactive } from 'vue'
import { useRoute } from 'vue-router'

const props = defineProps({
  rows: { type: Array, required: true },
  columns: { type: Array, required: true },
  options: {
    type: Object,
    default: () => ({
      selectable: true,
      showTooltip: true,
      resizeColumn: false,
      totalCount: 0,
      rowCount: 0,
    }),
  },
})
const emit = defineEmits([
  'loadMore',
  'updatePageCount',
  'columnWidthUpdated',
  'applyFilter',
  'applyLikeFilter',
  'likeDoc',
  'selectionsChanged',
])

const route = useRoute()

const pageLengthCount = defineModel({ type: Number })
const list = defineModel('list', { type: Object })

function onColumnWidthUpdated({ width, save }, column) {
  column.width = width
  if (save) emit('columnWidthUpdated', column)
}

function getLabel(label, column) {
  if (column.type === 'Duration') return formatDuration(label)
  if (column.options && isTranslatable(column.options)) return __(label)
  return label
}

const isLikeFilterApplied = computed(() => {
  return list.value.params?.filters?._liked_by ? true : false
})

const { user } = sessionStore()

function isLiked(item) {
  if (item) {
    let likedByMe = JSON.parse(item)
    return likedByMe.includes(user)
  }
}

watch(pageLengthCount, (val, old_value) => {
  if (val === old_value) return
  emit('updatePageCount', val)
})

const listBulkActionsRef = ref(null)

// ---------------------------------------------------------------------------
// IOON: edición en línea de cualquier columna editable
// ---------------------------------------------------------------------------

// Columnas que no se editan en la celda: siguen abriendo el lead al hacer clic.
// lead_name es un nombre calculado (nombre + apellido); para cambiarlo,
// muestra las columnas "Nombre" y "Apellido", que sí son editables.
const NO_EDITABLES = new Set([
  'name',
  'lead_name',
  'lead_owner',
  '_assign',
  '_liked_by',
  'modified',
  'creation',
  'owner',
  'modified_by',
  'first_response_time',
  'first_responded_on',
  'response_by',
  'sla',
  'sla_status',
  'sla_creation',
  'communication_status',
  'image',
  'naming_series',
  'converted',
])

// Tipo de campo de Frappe -> tipo de EditableCell
const TIPOS = {
  Data: 'Data',
  'Small Text': 'Data',
  Text: 'Data',
  'Long Text': 'Data',
  Phone: 'Data',
  Int: 'Int',
  Float: 'Float',
  Currency: 'Currency',
  Percent: 'Percent',
  Date: 'Date',
  Datetime: 'Datetime',
  Select: 'Select',
  Link: 'Select',
}

// Opciones de respaldo si la columna no trae las suyas
const OPCIONES_RESPALDO = {
  'custom_canal_del_último_intento': ['Llamada', 'WhatsApp', 'Correo', 'Visita'],
  'custom_resultado_del_último_intento': [
    'Contestó',
    'No contestó',
    'Buzón',
    'Número equivocado',
    'Volver a llamar',
  ],
}

function esEditable(column) {
  if (!column?.key || column.key.startsWith('_')) return false
  if (NO_EDITABLES.has(column.key)) return false
  return Boolean(TIPOS[column.type])
}

function tipoEditable(column) {
  return TIPOS[column.type] || 'Data'
}

// Opciones de los campos Link (estado, fuente, industria, campaña...),
// cargadas una vez por doctype
const recursosLink = shallowReactive({})

function asegurarOpciones(doctype) {
  if (!doctype || recursosLink[doctype]) return
  recursosLink[doctype] = createListResource({
    doctype,
    fields: ['name'],
    orderBy: doctype.endsWith('Status') ? 'position asc' : 'name asc',
    pageLength: 500,
    auto: true,
  })
}

watch(
  () => props.columns,
  (columnas) => {
    for (const c of columnas || []) {
      if (c.type === 'Link' && esEditable(c)) asegurarOpciones(c.options)
    }
  },
  { immediate: true },
)

function opcionesEditables(column) {
  if (column.type === 'Link') {
    return (recursosLink[column.options]?.data || []).map((d) => d.name)
  }
  if (column.type === 'Select') {
    return column.options || OPCIONES_RESPALDO[column.key] || []
  }
  return []
}

// Algunas columnas llegan como objeto ({ label, color }); se edita el valor real
function valorCrudo(v) {
  if (v && typeof v === 'object') return v.value ?? v.name ?? v.label ?? ''
  return v ?? ''
}

function textoVisible(label, column) {
  if (label === null || label === undefined || label === '') return ''
  return String(getLabel(label, column))
}

function alActualizar(row, column, { value }) {
  // Si la columna se dibuja con datos calculados (por ejemplo, el color del
  // estado), se recarga la lista; si no, basta con actualizar la fila.
  if (row[column.key] && typeof row[column.key] === 'object') {
    list.value?.reload?.()
  } else {
    row[column.key] = value
  }
}

defineExpose({
  customListActions: computed(
    () => listBulkActionsRef.value?.customListActions,
  ),
})
</script>
