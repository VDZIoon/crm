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
          <!-- IOON: celdas editables. El clic no abre el lead. -->
          <div
            v-if="esEditable(column)"
            class="w-full"
            @click.stop.prevent
            @mousedown.stop
          >
            <EditableCell
              doctype="CRM Lead"
              :name="row.name"
              :fieldname="column.key"
              :value="row[column.key] ?? ''"
              :type="tipoEditable(column)"
              :options="opcionesEditables(column)"
              :display="textoEditable(row[column.key], column)"
              @updated="({ value }) => (row[column.key] = value)"
            />
          </div>
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
import { ref, computed, watch } from 'vue'
import { useRoute } from 'vue-router'

defineProps({
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
// IOON: edición en línea
// Mismos nombres de campo que CampaignDetail.vue.
// ---------------------------------------------------------------------------
const CAMPO_CAMPANA = 'custom_campaña'

const EDITABLES = {
  custom_intentos_de_contacto: { tipo: 'Int' },
  'custom_canal_del_último_intento': {
    tipo: 'Select',
    opciones: ['Llamada', 'WhatsApp', 'Correo', 'Visita'],
  },
  'custom_resultado_del_último_intento': {
    tipo: 'Select',
    opciones: ['Contestó', 'No contestó', 'Buzón', 'Número equivocado', 'Volver a llamar'],
  },
  custom_comentarios_de_marketing: { tipo: 'Data' },
  [CAMPO_CAMPANA]: { tipo: 'Select', campanas: true },
}

const campanas = createListResource({
  doctype: 'CRM Campaign',
  fields: ['name'],
  orderBy: 'name asc',
  pageLength: 500,
  auto: true,
})

function esEditable(column) {
  return Boolean(EDITABLES[column.key])
}

function tipoEditable(column) {
  return EDITABLES[column.key]?.tipo || 'Data'
}

function opcionesEditables(column) {
  const conf = EDITABLES[column.key]
  if (!conf) return []
  if (conf.campanas) return (campanas.data || []).map((c) => c.name)
  // Si el campo trae sus opciones en la columna, se usan; si no, las de respaldo
  if (column.type === 'Select' && column.options) return column.options
  return conf.opciones || []
}

function textoEditable(valor, column) {
  if (valor === null || valor === undefined || valor === '') return ''
  if (tipoEditable(column) === 'Select' && !EDITABLES[column.key]?.campanas) return __(valor)
  return String(valor)
}

defineExpose({
  customListActions: computed(
    () => listBulkActionsRef.value?.customListActions,
  ),
})
</script>
