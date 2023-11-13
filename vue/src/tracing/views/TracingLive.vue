<template>
  <v-row>
    <v-col>
      <v-card rounded="lg" outlined class="mb-4">
        <v-toolbar flat color="light-blue lighten-5">
          <v-toolbar-title>
            <span>Spans</span>
          </v-toolbar-title>

          <v-spacer />

          <div class="text-body-2 blue-grey--text text--darken-3">
            <strong><XNum :value="spans.pager.numItem" verbose /></strong> spans
          </div>
        </v-toolbar>

        <v-row class="px-4 pb-4">
          <v-col>
            <LoadPctileChart :axios-params="axiosParams" class="pa-4" />

            <SpansLiveTable
              :loading="spans.loading"
              :spans="spans.items"
              :order="spans.order"
              :events-mode="systems.isEvent"
              :show-system="showSystem"
              @click:chip="onChipClick"
            />
          </v-col>
        </v-row>
      </v-card>

    </v-col>
  </v-row>
</template>

<script lang="ts">
import { defineComponent, computed, watch, PropType, onBeforeUnmount, getCurrentInstance } from 'vue'

// Composables
import { useRouter } from '@/use/router'
import { UseDateRange } from '@/use/date-range'
import { UseSystems } from '@/tracing/system/use-systems'
import { UseUql } from '@/use/uql'
import { useSpans } from '@/tracing/use-spans'

// Components
import SpansLiveTable from '@/tracing/SpansLiveTable.vue'
import { SpanChip } from '@/tracing/SpanChips.vue'
import LoadPctileChart from '@/components/LoadPctileChart.vue'

const LIVE_UPDATE_TIMER_DELAY = 5 * SECOND

// Utilities
import { isGroupSystem, AttrKey } from '@/models/otel'
import { SECOND } from '@/util/fmt/date'

export default defineComponent({
  name: 'TracingLive',
  components: { SpansLiveTable, LoadPctileChart },

  props: {
    dateRange: {
      type: Object as PropType<UseDateRange>,
      required: true,
    },
    systems: {
      type: Object as PropType<UseSystems>,
      required: true,
    },
    uql: {
      type: Object as PropType<UseUql>,
      required: true,
    },
    axiosParams: {
      type: Object as PropType<Record<string, any>>,
      required: true,
    },
  },

  setup(props) {
    props.dateRange.roundUp()
    const { route } = useRouter()
    let liveUpdateTimer: ReturnType<typeof setTimeout> | null

    const spans = useSpans(() => {
      const { projectId } = route.value.params
      return {
        url: `/api/v1/tracing/${projectId}/spanstail`,
        params: props.axiosParams,
      }
    })
    spans.order.syncQueryParams()

    const showSystem = computed(() => {
      const systems = props.systems.activeSystems
      if (systems.length > 1) {
        return true
      }
      if (systems.length === 1) {
        return isGroupSystem(systems[0])
      }
      return false
    })

    watch(
      () => props.systems.activeSystems,
      (system, oldSystem) => {
        if (!spans.order.column || oldSystem) {
          resetOrder()
        }
      },
    )
    props.dateRange.reloadNow()
    liveUpdate()
    watch(
    () => route.value.name,
    () => {
      liveUpdate()
    },
    { flush: 'post' },
  )

  if (getCurrentInstance()) {
    onBeforeUnmount(() => {
      if (liveUpdateTimer) {
        clearTimeout(liveUpdateTimer)
        liveUpdateTimer = null
      }
    })
  }

    function liveUpdate(): boolean {
      props.dateRange.isNow = true
      props.dateRange.reloadNow()

      if (liveUpdateTimer) {
        clearTimeout(liveUpdateTimer)
      }
      liveUpdateTimer = setTimeout(liveUpdate, LIVE_UPDATE_TIMER_DELAY)

      return true
    }

    function resetOrder() {
      spans.order.column = undefined
      spans.order.desc = true

      if (!props.systems.activeSystems) {
        return
      }
      spans.order.column = props.systems.isEvent ? AttrKey.spanTime : AttrKey.spanDuration
    }

    function onChipClick(chip: SpanChip) {
      const editor = props.uql.createEditor()
      editor.where(chip.key, '=', chip.value)
      props.uql.commitEdits(editor)
    }

    return {
      spans,
      showSystem,

      onChipClick,
    }
  },

})
</script>

<style lang="scss" scoped></style>
