<template>
  <div :class="computedClasses" class="btn-group">
    <BButton
      :id="computedId"
      ref="splitButton"
      :variant="splitVariant || variant"
      :size="size"
      :class="buttonClasses"
      :disabled="splitDisabledBoolean || disabled"
      :type="splitButtonType"
      :aria-label="ariaLabel"
      :aria-expanded="splitBoolean ? undefined : modelValueBoolean"
      :aria-haspopup="splitBoolean ? undefined : 'menu'"
      :href="splitBoolean ? splitHref : undefined"
      :to="splitBoolean && splitTo ? splitTo : undefined"
      @click="onSplitClick"
    >
      <slot name="button-content">
        {{ text }}
      </slot>
    </BButton>
    <BButton
      v-if="splitBoolean"
      ref="button"
      :variant="variant"
      :size="size"
      :disabled="disabled"
      :class="[toggleClass, ...[modelValueBoolean ? 'show' : undefined]]"
      class="dropdown-toggle-split dropdown-toggle"
      :aria-expanded="modelValueBoolean"
      aria-haspopup="menu"
      @click="onButtonClick"
    >
      <span class="visually-hidden">
        <slot name="toggle-text">
          {{ toggleText }}
        </slot>
      </span>
    </BButton>
    <ul
      v-if="!lazyBoolean || modelValueBoolean"
      v-show="lazyBoolean || modelValueBoolean"
      ref="floating"
      :style="{
        position: strategy === 'absolute' ? undefined : 'fixed',
        top: `${y}px`,
        left: `${x}px`,
        width: 'max-content',
      }"
      class="dropdown-menu show"
      :class="dropdownMenuClasses"
      :aria-labelledby="computedId"
      :role="role"
      @click="onClickInside"
    >
      <slot />
    </ul>
  </div>
</template>

<script setup lang="ts">
import {
  autoUpdate,
  type Boundary,
  flip,
  offset as floatingOffset,
  type Middleware,
  type RootBoundary,
  shift,
  type Strategy,
  useFloating,
} from '@floating-ui/vue'
import {onClickOutside, onKeyStroke, useToNumber, useVModel} from '@vueuse/core'
import {computed, provide, readonly, ref, toRef, watch} from 'vue'
import {useBooleanish, useId} from '../../composables'
import type {Booleanish, ButtonType, ButtonVariant, ClassValue, Size} from '../../types'
import {BvEvent, dropdownInjectionKey, resolveFloatingPlacement} from '../../utils'
import BButton from '../BButton/BButton.vue'
import type {RouteLocationRaw} from 'vue-router'

// TODO add navigation through keyboard events
// TODO standardize keydown vs keyup events globally

const props = withDefaults(
  defineProps<{
    ariaLabel?: string
    id?: string
    menuClass?: ClassValue
    size?: Size
    splitClass?: ClassValue
    splitVariant?: ButtonVariant | null
    text?: string
    toggleClass?: ClassValue
    autoClose?: boolean | 'inside' | 'outside'
    block?: Booleanish
    disabled?: Booleanish
    isNav?: Booleanish
    dropup?: Booleanish
    dropend?: Booleanish
    dropstart?: Booleanish
    center?: Booleanish
    end?: Booleanish
    noFlip?: Booleanish
    noShift?: Booleanish
    offset?:
      | number
      | string
      | {mainAxis?: number; crossAxis?: number; alignmentAxis?: number | null}
    role?: string
    split?: Booleanish
    splitButtonType?: ButtonType
    splitHref?: string
    splitDisabled?: Booleanish
    noCaret?: Booleanish
    toggleText?: string
    variant?: ButtonVariant | null
    modelValue?: Booleanish
    lazy?: Booleanish
    strategy?: Strategy
    floatingMiddleware?: Middleware[]
    splitTo?: RouteLocationRaw
    boundary?: Boundary | RootBoundary
  }>(),
  {
    ariaLabel: undefined,
    id: undefined,
    menuClass: undefined,
    size: 'md',
    splitClass: undefined,
    splitVariant: undefined,
    text: undefined,
    toggleClass: undefined,
    floatingMiddleware: undefined,
    splitDisabled: undefined,
    autoClose: true,
    block: false,
    disabled: false,
    isNav: false,
    dropup: false,
    dropend: false,
    dropstart: false,
    center: false,
    end: false,
    noFlip: false,
    lazy: false,
    noShift: false,
    offset: 0,
    role: 'menu',
    split: false,
    splitButtonType: 'button',
    splitHref: undefined,
    noCaret: false,
    toggleText: 'Toggle dropdown',
    variant: 'secondary',
    modelValue: false,
    strategy: 'absolute',
    splitTo: undefined,
    boundary: 'clippingAncestors',
  }
)

const emit = defineEmits<{
  'show': [value: BvEvent]
  'shown': []
  'hide': [value: BvEvent]
  'hidden': []
  'hide-prevented': []
  'show-prevented': []
  'click': [event: MouseEvent]
  'toggle': []
  'update:modelValue': [value: boolean]
}>()

defineSlots<{
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  'default'?: (props: Record<string, never>) => any
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  'button-content'?: (props: Record<string, never>) => any
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  'toggle-text'?: (props: Record<string, never>) => any
}>()

const computedId = useId(() => props.id, 'dropdown')

const modelValue = useVModel(props, 'modelValue', emit, {passive: true})

const modelValueBoolean = useBooleanish(modelValue)
const blockBoolean = useBooleanish(() => props.block)
const dropupBoolean = useBooleanish(() => props.dropup)
const dropendBoolean = useBooleanish(() => props.dropend)
const isNavBoolean = useBooleanish(() => props.isNav)
const dropstartBoolean = useBooleanish(() => props.dropstart)
const centerBoolean = useBooleanish(() => props.center)
const endBoolean = useBooleanish(() => props.end)
const splitBoolean = useBooleanish(() => props.split)
const noCaretBoolean = useBooleanish(() => props.noCaret)
const noFlipBoolean = useBooleanish(() => props.noFlip)
const noShiftBoolean = useBooleanish(() => props.noShift)
const lazyBoolean = useBooleanish(() => props.lazy)
const splitDisabledBoolean = useBooleanish(() => props.splitDisabled)

const computedOffset = computed(() =>
  typeof props.offset === 'string' || typeof props.offset === 'number' ? props.offset : NaN
)
const offsetToNumber = useToNumber(computedOffset)

const floating = ref<HTMLElement | null>(null)
const button = ref<HTMLElement | null>(null)
const splitButton = ref<HTMLElement | null>(null)

const boundary = computed<Boundary | undefined>(() =>
  props.boundary === 'document' || props.boundary === 'viewport' ? undefined : props.boundary
)
const rootBoundary = computed<RootBoundary | undefined>(() =>
  props.boundary === 'document' || props.boundary === 'viewport' ? props.boundary : undefined
)

onKeyStroke(
  'Escape',
  () => {
    modelValue.value = !modelValueBoolean
  },
  {target: splitButton}
)

const referencePlacement = computed(() => (!splitBoolean.value ? splitButton.value : button.value))
const floatingPlacement = computed(() =>
  resolveFloatingPlacement({
    top: dropupBoolean.value,
    start: dropstartBoolean.value,
    end: dropendBoolean.value,
    alignCenter: centerBoolean.value,
    alignEnd: endBoolean.value,
  })
)
const floatingMiddleware = computed<Middleware[]>(() => {
  if (props.floatingMiddleware !== undefined) {
    return props.floatingMiddleware
  }
  const localOffset =
    typeof props.offset === 'string' || typeof props.offset === 'number'
      ? offsetToNumber.value
      : props.offset
  const arr: Middleware[] = [floatingOffset(localOffset)]
  if (noFlipBoolean.value === false) {
    arr.push(
      flip({
        boundary: boundary.value,
        rootBoundary: rootBoundary.value,
      })
    )
  }
  if (noShiftBoolean.value === false) {
    arr.push(
      shift({
        boundary: boundary.value,
        rootBoundary: rootBoundary.value,
      })
    )
  }
  return arr
})
const {x, y, strategy, update} = useFloating(referencePlacement, floating, {
  placement: floatingPlacement,
  middleware: floatingMiddleware,
  strategy: readonly(toRef(props, 'strategy')),
  whileElementsMounted: autoUpdate,
})

const computedClasses = computed(() => ({
  'd-grid': blockBoolean.value,
  'dropup': dropupBoolean.value,
  'dropend': dropendBoolean.value,
  'dropstart': dropstartBoolean.value,
  'd-flex': blockBoolean.value && splitBoolean.value,
  'position-static': props.boundary !== 'clippingAncestors' && !isNavBoolean.value,
}))

const buttonClasses = computed(() => [
  splitBoolean.value ? props.splitClass : props.toggleClass,
  {
    'nav-link': isNavBoolean.value,
    'dropdown-toggle': !splitBoolean.value,
    'dropdown-toggle-no-caret': noCaretBoolean.value && !splitBoolean.value,
    'w-100': splitBoolean.value && blockBoolean.value,
    'show': splitBoolean.value ? undefined : modelValueBoolean.value,
  },
])

const dropdownMenuClasses = computed(() => props.menuClass)

const onButtonClick = () => {
  emit('toggle')
  const currentModelValue = modelValueBoolean.value
  const e = new BvEvent(currentModelValue ? 'hide' : 'show')
  currentModelValue ? emit('hide', e) : emit('show', e)
  if (e.defaultPrevented) {
    currentModelValue ? emit('hide-prevented') : emit('show-prevented')
    return
  }
  modelValue.value = !currentModelValue
  currentModelValue ? emit('hidden') : emit('shown')
}

const onSplitClick = (event: MouseEvent) => {
  splitBoolean.value ? emit('click', event) : onButtonClick()
}

onClickOutside(
  floating,
  () => {
    if (modelValueBoolean.value && (props.autoClose === true || props.autoClose === 'outside')) {
      toggle()
    }
  },
  {ignore: [button, splitButton]}
)
const onClickInside = () => {
  if (modelValueBoolean.value && (props.autoClose === true || props.autoClose === 'inside')) {
    toggle()
  }
}

const close = () => {
  modelValue.value = false
}
const open = () => {
  modelValue.value = true
}
const toggle = () => {
  modelValue.value = !modelValueBoolean.value
}

watch(modelValueBoolean, update)

defineExpose({
  close,
  open,
  toggle,
})

provide(dropdownInjectionKey, {
  id: computedId,
  open,
  close,
  toggle,
  visible: modelValueBoolean,
  isNav: isNavBoolean,
})
</script>
