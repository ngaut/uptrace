<template>
  <v-sheet color="grey lighten-5" class="my-0 x-code">
    <prism :code="code" :inline="inline" :language="language" />
  </v-sheet>
</template>

<script lang="ts">
import { defineComponent } from '@vue/composition-api'

import Prism from 'vue-prism-component'

export default defineComponent({
  name: 'XCode',
  components: { Prism },

  props: {
    code: {
      type: String,
      required: true,
    },
    language: {
      type: String,
      default: 'markup',
    },
    inline: {
      type: Boolean,
      default: false,
    },
  },

  setup() {
    return {}
  },
})
</script>

<style lang="scss">
.v-sheet.x-code {
  margin: 16px 0;
  position: relative;
  padding: 12px;

  pre,
  code {
    background: transparent;
    font-size: 1rem;
    font-weight: 300;
    margin: 0 !important;
  }

  > pre {
    border-radius: inherit;
  }

  code[class*='language'],
  pre[class*='language'] {
    background: none;
    font-family: Consolas, Monaco, 'Andale Mono', 'Ubuntu Mono', monospace;
    font-size: 0.875rem;
    hyphens: none;
    line-height: 1.5;
    margin: 0;
    padding: 0;
    tab-size: 4;
    text-align: left;
    text-shadow: none;
    white-space: pre;
    word-break: normal;
    word-spacing: normal;
    word-wrap: normal;
  }
}
</style>
