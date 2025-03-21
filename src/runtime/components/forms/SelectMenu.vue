<template>
  <component
    :is="searchable ? 'HCombobox' : 'HListbox'"
    v-slot="{ open }"
    :by="by"
    :name="name"
    :model-value="modelValue"
    :multiple="multiple"
    :disabled="disabled || loading"
    as="div"
    :class="wrapperClass"
    @update:model-value="onUpdate"
  >
    <input
      v-if="required"
      :value="modelValue"
      :required="required"
      class="absolute inset-0 w-px opacity-0 cursor-default"
      tabindex="-1"
      aria-hidden="true"
    >

    <component
      :is="searchable ? 'HComboboxButton' : 'HListboxButton'"
      ref="trigger"
      as="div"
      role="button"
      class="inline-flex w-full"
    >
      <slot :open="open" :disabled="disabled" :loading="loading">
        <button :class="selectClass" :disabled="disabled || loading" type="button" v-bind="attrs">
          <span v-if="(isLeading && leadingIconName) || $slots.leading" :class="leadingWrapperIconClass">
            <slot name="leading" :disabled="disabled" :loading="loading">
              <UIcon :name="leadingIconName" :class="leadingIconClass" />
            </slot>
          </span>

          <slot name="label">
            <span v-if="multiple && Array.isArray(modelValue) && modelValue.length" class="block truncate">{{ modelValue.length }} selected</span>
            <span v-else-if="!multiple && modelValue" class="block truncate">{{ typeof modelValue === 'string' ? modelValue : modelValue[optionAttribute] }}</span>
            <span v-else class="block truncate" :class="uiMenu.placeholder">{{ placeholder || '&nbsp;' }}</span>
          </slot>

          <span v-if="(isTrailing && trailingIconName) || $slots.trailing" :class="trailingWrapperIconClass">
            <slot name="trailing" :disabled="disabled" :loading="loading">
              <UIcon :name="trailingIconName" :class="trailingIconClass" aria-hidden="true" />
            </slot>
          </span>
        </button>
      </slot>
    </component>

    <div v-if="open" ref="container" :class="[uiMenu.container, uiMenu.width]">
      <Transition appear v-bind="uiMenu.transition">
        <component :is="searchable ? 'HComboboxOptions' : 'HListboxOptions'" static :class="[uiMenu.base, uiMenu.divide, uiMenu.ring, uiMenu.rounded, uiMenu.shadow, uiMenu.background, uiMenu.padding, uiMenu.height]">
          <HComboboxInput
            v-if="searchable"
            ref="searchInput"
            :display-value="() => query"
            name="q"
            :placeholder="searchablePlaceholder"
            autofocus
            autocomplete="off"
            :class="uiMenu.input"
            @change="query = $event.target.value"
          />
          <component
            :is="searchable ? 'HComboboxOption' : 'HListboxOption'"
            v-for="(option, index) in filteredOptions"
            v-slot="{ active, selected, disabled: optionDisabled }"
            :key="index"
            as="template"
            :value="valueAttribute ? option[valueAttribute] : option"
            :disabled="option.disabled"
          >
            <li :class="[uiMenu.option.base, uiMenu.option.rounded, uiMenu.option.padding, uiMenu.option.size, uiMenu.option.color, active ? uiMenu.option.active : uiMenu.option.inactive, selected && uiMenu.option.selected, optionDisabled && uiMenu.option.disabled]">
              <div :class="uiMenu.option.container">
                <slot name="option" :option="option" :active="active" :selected="selected">
                  <UIcon v-if="option.icon" :name="option.icon" :class="[uiMenu.option.icon.base, active ? uiMenu.option.icon.active : uiMenu.option.icon.inactive, option.iconClass]" aria-hidden="true" />
                  <UAvatar
                    v-else-if="option.avatar"
                    v-bind="{ size: uiMenu.option.avatar.size, ...option.avatar }"
                    :class="uiMenu.option.avatar.base"
                    aria-hidden="true"
                  />
                  <span v-else-if="option.chip" :class="uiMenu.option.chip.base" :style="{ background: `#${option.chip}` }" />

                  <span class="truncate">{{ typeof option === 'string' ? option : option[optionAttribute] }}</span>
                </slot>
              </div>

              <span v-if="selected" :class="[uiMenu.option.selectedIcon.wrapper, uiMenu.option.selectedIcon.padding]">
                <UIcon :name="selectedIcon" :class="uiMenu.option.selectedIcon.base" aria-hidden="true" />
              </span>
            </li>
          </component>

          <component :is="searchable ? 'HComboboxOption' : 'HListboxOption'" v-if="creatable && queryOption && !filteredOptions.length" v-slot="{ active, selected }" :value="queryOption" as="template">
            <li :class="[uiMenu.option.base, uiMenu.option.rounded, uiMenu.option.padding, uiMenu.option.size, uiMenu.option.color, active ? uiMenu.option.active : uiMenu.option.inactive]">
              <div :class="uiMenu.option.container">
                <slot name="option-create" :option="queryOption" :active="active" :selected="selected">
                  <span class="block truncate">Create "{{ queryOption[optionAttribute] }}"</span>
                </slot>
              </div>
            </li>
          </component>
          <p v-else-if="searchable && query && !filteredOptions.length" :class="uiMenu.option.empty">
            <slot name="option-empty" :query="query">
              No results for "{{ query }}".
            </slot>
          </p>
        </component>
      </Transition>
    </div>
  </component>
</template>

<script lang="ts">
import { ref, computed, watch, defineComponent } from 'vue'
import type { PropType, ComponentPublicInstance } from 'vue'
import {
  Combobox as HCombobox,
  ComboboxButton as HComboboxButton,
  ComboboxOptions as HComboboxOptions,
  ComboboxOption as HComboboxOption,
  ComboboxInput as HComboboxInput,
  Listbox as HListbox,
  ListboxButton as HListboxButton,
  ListboxOptions as HListboxOptions,
  ListboxOption as HListboxOption
} from '@headlessui/vue'
import { computedAsync, useDebounceFn } from '@vueuse/core'
import { defu } from 'defu'
import { omit } from 'lodash-es'
import { twMerge, twJoin } from 'tailwind-merge'
import UIcon from '../elements/Icon.vue'
import UAvatar from '../elements/Avatar.vue'
import { defuTwMerge } from '../../utils'
import { usePopper } from '../../composables/usePopper'
import { useFormGroup } from '../../composables/useFormGroup'
import type { PopperOptions } from '../../types'
import { useAppConfig } from '#imports'
// TODO: Remove
// @ts-expect-error
import appConfig from '#build/app.config'

// const appConfig = useAppConfig()

export default defineComponent({
  components: {
    HCombobox,
    HComboboxButton,
    HComboboxOptions,
    HComboboxOption,
    HComboboxInput,
    HListbox,
    HListboxButton,
    HListboxOptions,
    HListboxOption,
    UIcon,
    UAvatar
  },
  inheritAttrs: false,
  props: {
    modelValue: {
      type: [String, Number, Object, Array],
      default: ''
    },
    by: {
      type: String,
      default: undefined
    },
    options: {
      type: Array as PropType<{ [key: string]: any, disabled?: boolean }[] | string[]>,
      default: () => []
    },
    name: {
      type: String,
      default: null
    },
    required: {
      type: Boolean,
      default: false
    },
    icon: {
      type: String,
      default: null
    },
    loadingIcon: {
      type: String,
      default: () => appConfig.ui.input.default.loadingIcon
    },
    leadingIcon: {
      type: String,
      default: null
    },
    trailingIcon: {
      type: String,
      default: () => appConfig.ui.select.default.trailingIcon
    },
    trailing: {
      type: Boolean,
      default: false
    },
    leading: {
      type: Boolean,
      default: false
    },
    loading: {
      type: Boolean,
      default: false
    },
    selectedIcon: {
      type: String,
      default: () => appConfig.ui.selectMenu.default.selectedIcon
    },
    disabled: {
      type: Boolean,
      default: false
    },
    multiple: {
      type: Boolean,
      default: false
    },
    searchable: {
      type: [Boolean, Function] as PropType<boolean | ((query: string) => Promise<any[]> | any[])>,
      default: false
    },
    searchablePlaceholder: {
      type: String,
      default: 'Search...'
    },
    debounce: {
      type: Number,
      default: 200
    },
    creatable: {
      type: Boolean,
      default: false
    },
    placeholder: {
      type: String,
      default: null
    },
    padded: {
      type: Boolean,
      default: true
    },
    size: {
      type: String,
      default: () => appConfig.ui.select.default.size,
      validator (value: string) {
        return Object.keys(appConfig.ui.select.size).includes(value)
      }
    },
    color: {
      type: String,
      default: () => appConfig.ui.select.default.color,
      validator (value: string) {
        return [...appConfig.ui.colors, ...Object.keys(appConfig.ui.select.color)].includes(value)
      }
    },
    variant: {
      type: String,
      default: () => appConfig.ui.select.default.variant,
      validator (value: string) {
        return [
          ...Object.keys(appConfig.ui.select.variant),
          ...Object.values(appConfig.ui.select.color).flatMap(value => Object.keys(value))
        ].includes(value)
      }
    },
    optionAttribute: {
      type: String,
      default: 'label'
    },
    valueAttribute: {
      type: String,
      default: null
    },
    searchAttributes: {
      type: Array,
      default: null
    },
    popper: {
      type: Object as PropType<PopperOptions>,
      default: () => ({})
    },
    selectClass: {
      type: String,
      default: null
    },
    ui: {
      type: Object as PropType<Partial<typeof appConfig.ui.select>>,
      default: () => ({})
    },
    uiMenu: {
      type: Object as PropType<Partial<typeof appConfig.ui.selectMenu>>,
      default: () => ({})
    }
  },
  emits: ['update:modelValue', 'open', 'close', 'change'],
  setup (props, { emit, attrs, slots }) {
    // TODO: Remove
    const appConfig = useAppConfig()

    const ui = computed<Partial<typeof appConfig.ui.select>>(() => defuTwMerge({}, props.ui, appConfig.ui.select))
    const uiMenu = computed<Partial<typeof appConfig.ui.selectMenu>>(() => defuTwMerge({}, props.uiMenu, appConfig.ui.selectMenu))

    const popper = computed<PopperOptions>(() => defu({}, props.popper, uiMenu.value.popper as PopperOptions))

    const [trigger, container] = usePopper(popper.value)
    const { emitFormBlur, emitFormChange, formGroup } = useFormGroup()
    const color = computed(() => formGroup?.error?.value ? 'red' : props.color)
    const size = computed(() => formGroup?.size?.value ?? props.size)

    const query = ref('')
    const searchInput = ref<ComponentPublicInstance<HTMLElement>>()

    const wrapperClass = computed(() => twMerge(ui.value.wrapper, attrs.class as string))

    const selectClass = computed(() => {
      const variant = ui.value.color?.[color.value as string]?.[props.variant as string] || ui.value.variant[props.variant]

      return twMerge(twJoin(
        ui.value.base,
        ui.value.rounded,
        'text-left cursor-default',
        ui.value.size[size.value],
        ui.value.gap[size.value],
        props.padded ? ui.value.padding[size.value] : 'p-0',
        variant?.replaceAll('{color}', color.value),
        (isLeading.value || slots.leading) && ui.value.leading.padding[size.value],
        (isTrailing.value || slots.trailing) && ui.value.trailing.padding[size.value],
        'inline-flex items-center'
      ), props.selectClass)
    })

    const isLeading = computed(() => {
      return (props.icon && props.leading) || (props.icon && !props.trailing) || (props.loading && !props.trailing) || props.leadingIcon
    })

    const isTrailing = computed(() => {
      return (props.icon && props.trailing) || (props.loading && props.trailing) || props.trailingIcon
    })

    const leadingIconName = computed(() => {
      if (props.loading) {
        return props.loadingIcon
      }

      return props.leadingIcon || props.icon
    })

    const trailingIconName = computed(() => {
      if (props.loading && !isLeading.value) {
        return props.loadingIcon
      }

      return props.trailingIcon || props.icon
    })

    const leadingWrapperIconClass = computed(() => {
      return twJoin(
        ui.value.icon.leading.wrapper,
        ui.value.icon.leading.pointer,
        ui.value.icon.leading.padding[size.value]
      )
    })

    const leadingIconClass = computed(() => {
      return twJoin(
        ui.value.icon.base,
        appConfig.ui.colors.includes(color.value) && ui.value.icon.color.replaceAll('{color}', color.value),
        ui.value.icon.size[size.value],
        props.loading && 'animate-spin'
      )
    })

    const trailingWrapperIconClass = computed(() => {
      return twJoin(
        ui.value.icon.trailing.wrapper,
        ui.value.icon.trailing.pointer,
        ui.value.icon.trailing.padding[size.value]
      )
    })

    const trailingIconClass = computed(() => {
      return twJoin(
        ui.value.icon.base,
        appConfig.ui.colors.includes(color.value) && ui.value.icon.color.replaceAll('{color}', color.value),
        ui.value.icon.size[size.value],
        props.loading && !isLeading.value && 'animate-spin'
      )
    })

    const debouncedSearch = typeof props.searchable === 'function' ? useDebounceFn(props.searchable, props.debounce) : undefined

    const filteredOptions = computedAsync(async () => {
      if (props.searchable && debouncedSearch) {
        return await debouncedSearch(query.value)
      }

      if (query.value === '') {
        return props.options
      }

      return (props.options as any[]).filter((option: any) => {
        return (props.searchAttributes?.length ? props.searchAttributes : [props.optionAttribute]).some((searchAttribute: any) => {
          return typeof option === 'string' ? option.search(new RegExp(query.value, 'i')) !== -1 : (option[searchAttribute] && option[searchAttribute].search(new RegExp(query.value, 'i')) !== -1)
        })
      })
    })

    const queryOption = computed(() => {
      return query.value === '' ? null : { [props.optionAttribute]: query.value }
    })

    watch(container, (value) => {
      if (value) {
        emit('open')
      } else {
        emit('close')
        emitFormBlur()
      }
    })

    function onUpdate (event: any) {
      if (query.value && searchInput.value?.$el) {
        query.value = ''
        // explicitly set input text because `ComboboxInput` `displayValue` is not reactive
        searchInput.value.$el.value = ''
      }
      emit('update:modelValue', event)
      emit('change', event)
      emitFormChange()
    }

    return {
      attrs: omit(attrs, ['class']),
      // eslint-disable-next-line vue/no-dupe-keys
      uiMenu,
      trigger,
      container,
      isLeading,
      isTrailing,
      wrapperClass,
      // eslint-disable-next-line vue/no-dupe-keys
      selectClass,
      leadingIconName,
      leadingIconClass,
      leadingWrapperIconClass,
      trailingIconName,
      trailingIconClass,
      trailingWrapperIconClass,
      filteredOptions,
      queryOption,
      query,
      onUpdate
    }
  }
})
</script>
