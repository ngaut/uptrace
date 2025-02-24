<template>
  <div>
    <SpanTable
      :date-range="dateRange"
      :loading="spans.loading"
      :spans="spans.items"
      :order="spans.order"
      :pager="spans.pager"
      class="mb-4"
      v-on="listeners"
    />

    <XPagination :pager="spans.pager" />
  </div>
</template>

<script lang="ts">
import { defineComponent, computed, PropType } from '@vue/composition-api'

// Composables
import { UseDateRange } from '@/use/date-range'
import { UseUql } from '@/use/uql'
import { useSpans } from '@/use/spans'

// Components
import SpanTable from '@/components/SpanTable.vue'
import { SpanChip } from '@/components/SpanChips.vue'

// Utilities
import { xkey } from '@/models/otelattr'

export default defineComponent({
  name: 'SpanListInline',
  components: { SpanTable },

  props: {
    dateRange: {
      type: Object as PropType<UseDateRange>,
      required: true,
    },
    uql: {
      type: Object as PropType<UseUql>,
      default: undefined,
    },
    axiosParams: {
      type: Object as PropType<Record<string, any>>,
      required: true,
    },
    where: {
      type: String,
      required: true,
    },
  },

  setup(props) {
    const spans = useSpans(
      () => {
        const query = props.where + ' | ' + props.axiosParams.query
        return {
          url: `/api/tracing/spans`,
          params: {
            ...props.axiosParams,
            query,
          },
        }
      },
      {
        order: {
          column: xkey.spanDuration,
          desc: true,
        },
      },
    )

    const listeners = computed(() => {
      if (!props.uql) {
        return {}
      }
      return { 'click:chip': onChipClick }
    })

    function onChipClick(chip: SpanChip) {
      const editor = props.uql.createEditor()
      editor.where(chip.key, '=', chip.value)
      props.uql.commitEdits(editor)
    }

    return { spans, listeners, onChipClick }
  },
})
</script>

<style lang="scss" scoped></style>
