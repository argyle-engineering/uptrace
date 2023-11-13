<template>
  <div>
    <v-simple-table>
      <thead v-if="loading">
        <tr class="v-data-table__progress">
          <th colspan="99" class="column">
            <v-progress-linear height="2" absolute indeterminate />
          </th>
        </tr>
      </thead>

      <tbody v-if="!spans.length">
        <tr class="v-data-table__empty-wrapper">
          <td colspan="99" class="py-16">
            <div class="mb-4">There are no matching spans. Try to change filters.</div>
            <v-btn :to="{ name: 'TracingHelp' }">
              <v-icon left>mdi-help-circle-outline</v-icon>
              <span>Help</span>
            </v-btn>
          </td>
        </tr>
      </tbody>
    </v-simple-table>
    <v-container
      fluid="true"
      >
      <v-row v-if="spans.length">
        <v-col>Span Name</v-col>
        <v-col cols="1" v-if="showSystem">System</v-col>
        <v-col cols="2"></v-col>
        <v-col cols="2" :value="AttrKey.spanTime" :order="order">Time</v-col>
        <v-col cols="1" v-if="!eventsMode" :value="AttrKey.spanDuration" :order="order" align="end">
          <span>Dur.</span>
        </v-col>
    </v-row>
    <v-divider></v-divider>
    <v-virtual-scroll
          :bench="5"
          :items="spans"
          height="500"
          item-height="34"
        >
    <template v-slot:default="{ item }">
            <v-row :key="item">

            <v-col @click="dialog.showSpan(item)" class="cursor-pointer text-no-wrap text-truncate">
              {{ item.displayName }}
            </v-col>
            <v-col cols="1" align="left" class="text-no-wrap text-truncate" v-if="showSystem">
              <router-link :to="systemRoute(item)" @click.native.stop>{{
                item.system
              }}</router-link>
            </v-col>
            <v-col cols="2">
              <SpanChips
                :span="item"
                :clickable="'click:chip' in $listeners"
                @click:chip="$emit('click:chip', $event)"
              />
            </v-col>
            <v-col cols="2" class="text-no-wrap"><XDate :date="item.time" format="time" /></v-col>
            <v-col cols="1" v-if="!eventsMode" class="text-right">
              <XDuration :duration="item.duration" fixed />
            </v-col>

            </v-row>

          </template>
        </v-virtual-scroll>
      </v-container>
    <v-dialog v-model="dialog.isActive" max-width="1280">
      <v-sheet>
        <SpanCardDateRange v-if="dialog.activeSpan" :span="dialog.activeSpan" />
      </v-sheet>
    </v-dialog>
  </div>
</template>

<script lang="ts">
import { defineComponent, shallowRef, nextTick, proxyRefs, PropType } from 'vue'

// Composables
import { useRouteQuery } from '@/use/router'
import { UseOrder } from '@/use/order'

// Components
import SpanCardDateRange from '@/tracing/SpanCardDateRange.vue'
import SpanChips from '@/tracing/SpanChips.vue'

// Utilities
import { AttrKey } from '@/models/otel'
import { Span } from '@/models/span'

export default defineComponent({
  name: 'SpansLiveTable',
  components: {
    SpanChips,
    SpanCardDateRange,
  },

  props: {
    loading: {
      type: Boolean,
      required: true,
    },
    spans: {
      type: Array as PropType<Span[]>,
      required: true,
    },
    order: {
      type: Object as PropType<UseOrder>,
      required: true,
    },
    columns: {
      type: Array as PropType<string[]>,
      default: () => [],
    },
    eventsMode: {
      type: Boolean,
      required: true,
    },
    showSystem: {
      type: Boolean,
      default: false,
    },
  },

  setup(props) {
    const dialog = useDialog()

    useRouteQuery().onRouteChanged(() => {
      dialog.close()
    })

    function systemRoute(span: Span) {
      return {
        query: {
          //...route.value.query,
          system: span.system,
        },
      }
    }

    function onSortBy() {
      nextTick(() => {
        props.order.desc = true
      })
    }

    return {
      AttrKey,
      dialog,

      systemRoute,
      onSortBy,
    }
  },
})

function useDialog() {
  const dialog = shallowRef(false)
  const activeSpan = shallowRef<Span>()

  function showSpan(span: Span) {
    activeSpan.value = span
    dialog.value = true
  }

  function close() {
    dialog.value = false
  }

  return proxyRefs({
    isActive: dialog,
    activeSpan,

    showSpan,
    close,
  })
}
</script>

<style lang="scss" scoped>
.v-simple-table__wrapper {
  max-height: 500px;
  overflow-y: auto;
}
</style>
