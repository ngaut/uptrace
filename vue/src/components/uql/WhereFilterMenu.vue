<template>
  <v-menu v-model="menu" offset-y :close-on-content-click="false">
    <template #activator="{ on, attrs }">
      <v-btn text class="v-btn--filter" v-bind="attrs" v-on="on">
        <span>Where</span>
        <v-icon right class="ml-0">mdi-menu-down</v-icon>
      </v-btn>
    </template>
    <v-card class="pa-3">
      <v-row>
        <v-col cols="auto">
          <v-form ref="form" v-model="isValid" class="pa-3" style="width: 400px">
            <v-row>
              <v-col class="grey--text text--darken-3"
                >For example, <strong>where span.duration > 100ms</strong>.</v-col
              >
            </v-row>

            <v-row class="mb-n3">
              <v-col cols="auto" class="pt-4 pr-2">
                <v-avatar color="grey" size="30">
                  <span class="white--text text-h6">1</span>
                </v-avatar>
              </v-col>
              <v-col>
                <SimpleSuggestions
                  v-model="column"
                  :loading="columnSuggestions.loading"
                  :suggestions="columnSuggestions"
                  :rules="rules.column"
                  label="Column"
                  dense
                  class="fit"
                />
              </v-col>
            </v-row>

            <v-row class="mb-n3">
              <v-col cols="auto" class="pt-4 pr-2">
                <v-avatar color="grey" size="30">
                  <span class="white--text text-h6">2</span>
                </v-avatar>
              </v-col>
              <v-col>
                <v-autocomplete
                  v-model="op"
                  :rules="rules.op"
                  label="Operator"
                  :items="opItems"
                  outlined
                  dense
                  clearable
                  hide-details="auto"
                  class="fit"
                  style="width: 200px"
                ></v-autocomplete>
              </v-col>
            </v-row>

            <v-row>
              <v-col cols="auto" class="pt-4 pr-2">
                <v-avatar color="grey" size="30">
                  <span class="white--text text-h6">3</span>
                </v-avatar>
              </v-col>
              <v-col>
                <SimpleSuggestions
                  v-model="colValue"
                  :loading="valueSuggestions.loading"
                  :suggestions="valueSuggestions"
                  :rules="rules.colValue"
                  label="Value"
                  :placeholder="valuePlaceholder"
                  :hint="valueHint"
                  :disabled="valueDisabled"
                  dense
                  class="fit"
                />
              </v-col>
            </v-row>

            <v-row>
              <v-spacer />
              <v-col cols="auto">
                <v-btn :disabled="!isValid" class="primary" @click="addFilter">Filter</v-btn>
              </v-col>
            </v-row>
          </v-form>
        </v-col>
        <template v-if="lcAttrs.length">
          <v-col cols="auto">
            <v-divider vertical />
          </v-col>
          <v-col cols="auto" class="pa-4">
            <div v-for="attrKey in lcAttrs" :key="attrKey">
              <AttrFilterMenu
                :uql="uql"
                :axios-params="axiosParams"
                :attr-key="attrKey"
                :label="attrKey"
                @change="menu = false"
              />
            </div>
          </v-col>
        </template>
      </v-row>
    </v-card>
  </v-menu>
</template>

<script lang="ts">
import { defineComponent, shallowRef, computed, watch, PropType } from '@vue/composition-api'

// Composables
import { AxiosParams } from '@/use/axios'
import { UseSystems } from '@/use/systems'
import { useSuggestions, Suggestion } from '@/use/suggestions'
import { UseUql } from '@/use/uql'

// Components
import SimpleSuggestions from '@/components/SimpleSuggestions.vue'
import AttrFilterMenu from '@/components/uql/AttrFilterMenu.vue'

// Utilities
import { requiredRule } from '@/util/validation'
import { quote } from '@/util/string'
import { xkey } from '@/models/otelattr'

const compOp = {
  contains: 'contains',
  doesNotContain: 'does not contain',

  like: 'like',
  notLike: 'not like',

  exists: 'exists',
  doesNotExist: 'does not exist',
}

export default defineComponent({
  name: 'WhereFilterMenu',
  components: { AttrFilterMenu, SimpleSuggestions },

  props: {
    systems: {
      type: Object as PropType<UseSystems>,
      required: true,
    },
    uql: {
      type: Object as PropType<UseUql>,
      required: true,
    },
    axiosParams: {
      type: Object as PropType<AxiosParams>,
      required: true,
    },
  },

  setup(props) {
    const menu = shallowRef(false)
    const column = shallowRef<Suggestion>()
    const op = shallowRef('')
    const colValue = shallowRef<Suggestion>()

    const form = shallowRef()
    const isValid = shallowRef(false)
    const rules = {
      column: [requiredRule],
      op: [requiredRule],
      colValue: [],
    }

    const columnSuggestions = useSuggestions(
      () => {
        if (!menu.value) {
          return null
        }

        return {
          url: '/api/tracing/suggestions/attributes',
          params: props.axiosParams,
        }
      },
      { suggestSearchInput: true },
    )

    const lcAttrs = computed(() => {
      const candidates = [
        xkey.serviceName,
        xkey.hostName,
        xkey.codeFilepath,
        xkey.codeFunction,
        xkey.rpcMethod,
        xkey.httpMethod,
        xkey.httpStatusCode,
      ] as string[]

      return columnSuggestions.items
        .filter((v) => candidates.indexOf(v.text) >= 0)
        .map((v) => v.text)
        .sort((a, b) => {
          return candidates.indexOf(a) - candidates.indexOf(b)
        })
    })

    const opItems = computed((): string[] => {
      return [
        '=',
        '!=',
        '<',
        '<=',
        '>',
        '>=',
        compOp.contains,
        compOp.doesNotContain,
        compOp.like,
        compOp.notLike,
        compOp.exists,
        compOp.doesNotExist,
      ]
    })

    const valueSuggestions = useSuggestions(
      () => {
        if (!menu.value || !column.value || !column.value.text) {
          return
        }

        return {
          url: `/api/tracing/suggestions/values`,
          params: {
            ...props.axiosParams,
            column: column.value.text,
          },
        }
      },
      { suggestSearchInput: true },
    )

    const valuePlaceholder = computed((): string => {
      switch (op.value) {
        case compOp.like:
        case compOp.notLike:
          return '%substring% or %suffix or prefix%'
        case compOp.contains:
        case compOp.doesNotContain:
          return 'substr1|substr2|substr3'
        default:
          return ''
      }
    })

    const valueHint = computed((): string => {
      switch (op.value) {
        case compOp.like:
        case compOp.notLike:
          return '"%" matches zero or more characters'
        case compOp.contains:
        case compOp.doesNotContain:
          return 'Case-insensitive options separated with "|"'
        default:
          return ''
      }
    })

    const valueDisabled = computed((): boolean => {
      switch (op.value) {
        case compOp.exists:
        case compOp.doesNotExist:
          return true
        default:
          return false
      }
    })

    function addFilter() {
      setTimeout(() => {
        if (!column.value || !op.value) {
          return
        }

        const editor = props.uql.createEditor()

        if (valueDisabled.value) {
          editor.add(`where ${column.value.text} ${op.value}`)
        } else {
          const value = colValue.value?.text ?? ''
          editor.add(`where ${column.value.text} ${op.value} ${quote(value)}`)
        }

        props.uql.commitEdits(editor)

        column.value = undefined
        op.value = ''
        colValue.value = undefined
        form.value.resetValidation()

        menu.value = false
      }, 10)
    }

    watch(valueDisabled, (disabled) => {
      if (disabled) {
        colValue.value = undefined
      }
    })

    return {
      xkey,
      menu,
      lcAttrs,

      form,
      isValid,
      rules,
      columnSuggestions,
      opItems,
      valueSuggestions,
      valuePlaceholder,
      valueHint,
      valueDisabled,

      column,
      op,
      colValue,

      addFilter,
    }
  },
})
</script>

<style lang="scss" scoped>
.v-select.fit {
  min-width: min-content !important;
}

.v-select.fit .v-select__selection--comma {
  text-overflow: unset;
}

.no-transform ::v-deep .v-btn {
  padding: 0 12px !important;
  text-transform: none;
}

.space-around ::v-deep .v-chip {
  margin: 4px;
}
</style>
