<script lang="ts" setup>
import { vOnLongPress } from '@vueuse/components'
import { storeToRefs } from 'pinia'
import { computed } from 'vue'
import Header from './Header.vue'
import TableTorrent from './TableTorrent.vue'
import { useI18nUtils } from '@/composables'
import { TorrentState } from '@/constants/vuetorrent'
import { comparators, getTorrentStateColor } from '@/helpers'
import { useAppStore, useDashboardStore, useTorrentStore, useVueTorrentStore } from '@/stores'
import { Torrent, Torrent as TorrentType } from '@/types/vuetorrent'

defineProps<{
  height: number
  paginatedTorrents: TorrentType[]
}>()

defineEmits<{
  onCheckboxClick: [e: { shiftKey: boolean; metaKey: boolean; ctrlKey: boolean }, torrent: TorrentType]
  onTorrentClick: [e: { shiftKey: boolean; metaKey: boolean; ctrlKey: boolean }, torrent: TorrentType]
  onTorrentDblClick: [torrent: TorrentType]
  onTorrentRightClick: [e: MouseEvent, torrent: TorrentType]
}>()

const { t, getTorrentStateString } = useI18nUtils()
const appStore = useAppStore()
const dashboardStore = useDashboardStore()
const { sortCriterias } = storeToRefs(useTorrentStore())
const vuetorrentStore = useVueTorrentStore()

const torrentProperties = computed(() => vuetorrentStore.tableProperties.filter(ppt => ppt.active).sort((a, b) => comparators.numeric.asc(a.order, b.order)))
const sortCriteria = computed({
  get: () => sortCriterias.value[0],
  set: v => (sortCriterias.value = [{ value: v.value, reverse: v.reverse }]),
})

const headers = computed(() => [
  { key: 'statusIndicator', sortable: false },
  ...(dashboardStore.isSelectionMultiple ? [{ key: 'multipleSelectionCheckbox', sortable: false }] : []),
  { title: t('torrent.properties.name'), key: 'name' },
  ...torrentProperties.value.filter(ppt => appStore.isFeatureAvailable(ppt.qbitVersion)).map(ppt => ({ title: t(ppt.props.titleKey), key: ppt.sortKey })),
])

function onHeaderClick(sortKey: keyof Torrent) {
  const crit = sortCriteria.value
  if (crit.value === sortKey) {
    sortCriteria.value = { value: crit.value, reverse: !crit.reverse }
  } else {
    sortCriteria.value = { value: sortKey, reverse: crit.reverse }
  }
}

function isTorrentSelected(torrent: TorrentType) {
  return dashboardStore.isTorrentInSelection(torrent.hash)
}

function getTorrentRowColorClass(torrent: TorrentType) {
  return [isTorrentSelected(torrent) ? `bg-${getTorrentStateColor(torrent.state)}-darken-3` : '']
}
</script>

<template>
  <v-data-table
    id="torrentList"
    density="compact"
    :mobile="false"
    :headers="headers"
    :items="paginatedTorrents"
    :height="height"
    :item-height="36"
    items-per-page="-1"
    hide-default-footer
    fixed-header>
    <template #headers="{ columns }">
      <tr>
        <template v-for="header in columns" :key="header.key">
          <th v-if="header.key === 'statusIndicator'" class="px-1" />
          <th v-else-if="header.key === 'multipleSelectionCheckbox'" />
          <Header v-else :title="header.title!" :sort-key="header.key!" @on-header-click="onHeaderClick(header.key as keyof Torrent)" />
        </template>
      </tr>
    </template>

    <template #item="{ item: torrent }">
      <tr
        v-ripple
        v-on-long-press.prevent="e => $emit('onTorrentRightClick', e, torrent)"
        data-custom-context-menu
        :class="['cursor-pointer', 'selected', 'ripple-fix', getTorrentRowColorClass(torrent)]"
        @contextmenu="$emit('onTorrentRightClick', $event, torrent)"
        @click="$emit('onTorrentClick', $event, torrent)"
        @dblclick="$emit('onTorrentDblClick', torrent)">
        <v-tooltip top>
          <template #activator="{ props }">
            <td v-bind="props" :class="`pa-0 bg-torrent-${TorrentState[torrent.state].toLowerCase()}`" />
          </template>
          {{ getTorrentStateString(torrent.state) }}
        </v-tooltip>

        <td v-if="dashboardStore.isSelectionMultiple">
          <v-checkbox-btn
            :model-value="isTorrentSelected(torrent)"
            :color="`torrent-${TorrentState[torrent.state].toLowerCase()}`"
            variant="text"
            @click.stop="$emit('onCheckboxClick', $event, torrent)" />
        </td>

        <td class="torrent-name text-no-wrap">
          {{ torrent.name }}
        </td>
        <TableTorrent :torrent="torrent" />
      </tr>
    </template>
  </v-data-table>
</template>

<style lang="scss">
@use 'vuetify/settings';

#torrentList {
  background-color: unset;

  tbody tr:nth-child(even) {
    background-color: settings.$card-background;
  }

  tbody tr.selected {
    position: relative;

    &:nth-child(odd)::after {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
    }
  }

  .torrent-name {
    max-width: 40vw;
    overflow: hidden;
    text-overflow: ellipsis;
  }
}
</style>
