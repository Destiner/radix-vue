<script setup lang="ts">
import { Primitive, usePrimitiveElement } from '@/Primitive'
import type { PrimitiveProps } from '@/Primitive'
import { type ComponentPublicInstance, computed, inject, onMounted, ref, toRefs, watchEffect } from 'vue'
import { TOAST_PROVIDER_INJECTION_KEY } from './ToastProvider.vue'
import { onKeyStroke, unrefElement } from '@vueuse/core'
import FocusProxy from './FocusProxy.vue'
import { focusFirst, getTabbableCandidates } from '@/FocusScope/utils'
import { useCollection } from '@/shared'
import { VIEWPORT_DEFAULT_HOTKEY, VIEWPORT_PAUSE, VIEWPORT_RESUME } from './utils'
import { DismissableLayerBranch } from '@/DismissableLayer'

export interface ToastViewportProps extends PrimitiveProps {
  /**
   * The keys to use as the keyboard shortcut that will move focus to the toast viewport.
   * @defaultValue ['F8']
   */
  hotkey?: string[]
  /**
   * An author-localized label for the toast viewport to provide context for screen reader users
   * when navigating page landmarks. The available `{hotkey}` placeholder will be replaced for you.
   * @defaultValue 'Notifications ({hotkey})'
   */
  label?: string
}

const props = withDefaults(defineProps<ToastViewportProps>(), {
  hotkey: () => VIEWPORT_DEFAULT_HOTKEY,
  label: 'Notifications ({hotkey})',
  as: 'ol',
})
const { hotkey, label } = toRefs(props)

const { primitiveElement, currentElement } = usePrimitiveElement()
const { createCollection } = useCollection()
const collections = createCollection(currentElement)
const context = inject(TOAST_PROVIDER_INJECTION_KEY)
const hasToasts = computed(() => context!.toastCount.value > 0)
const headFocusProxyRef = ref<HTMLElement>()
const tailFocusProxyRef = ref<HTMLElement>()

onKeyStroke(hotkey.value, () => {
  currentElement.value.focus()
})

onMounted(() => {
  context?.onViewportChange(currentElement.value)
})

watchEffect((cleanupFn) => {
  const viewport = currentElement.value
  if (hasToasts.value && viewport && context) {
    const handlePause = () => {
      if (!context.isClosePausedRef.value) {
        const pauseEvent = new CustomEvent(VIEWPORT_PAUSE)
        viewport.dispatchEvent(pauseEvent)
        context.isClosePausedRef.value = true
      }
    }

    const handleResume = () => {
      if (context.isClosePausedRef.value) {
        const resumeEvent = new CustomEvent(VIEWPORT_RESUME)
        viewport.dispatchEvent(resumeEvent)
        context.isClosePausedRef.value = false
      }
    }

    const handleFocusOutResume = (event: FocusEvent) => {
      const isFocusMovingOutside = !viewport.contains(event.relatedTarget as HTMLElement)
      if (isFocusMovingOutside)
        handleResume()
    }

    const handlePointerLeaveResume = () => {
      const isFocusInside = viewport.contains(document.activeElement)
      if (!isFocusInside)
        handleResume()
    }

    // We programmatically manage tabbing as we are unable to influence
    // the source order with portals, this allows us to reverse the
    // tab order so that it runs from most recent toast to least
    const handleKeyDown = (event: KeyboardEvent) => {
      const isMetaKey = event.altKey || event.ctrlKey || event.metaKey
      const isTabKey = event.key === 'Tab' && !isMetaKey

      if (isTabKey) {
        const focusedElement = document.activeElement
        const isTabbingBackwards = event.shiftKey
        const targetIsViewport = event.target === viewport

        // If we're back tabbing after jumping to the viewport then we simply
        // proxy focus out to the preceding document
        if (targetIsViewport && isTabbingBackwards) {
          headFocusProxyRef.value?.focus()
          return
        }

        const tabbingDirection = isTabbingBackwards ? 'backwards' : 'forwards'
        const sortedCandidates = getSortedTabbableCandidates({ tabbingDirection })
        const index = sortedCandidates.findIndex(candidate => candidate === focusedElement)
        if (focusFirst(sortedCandidates.slice(index + 1))) {
          event.preventDefault()
        }
        else {
          // If we can't focus that means we're at the edges so we
          // proxy to the corresponding exit point and let the browser handle
          // tab/shift+tab keypress and implicitly pass focus to the next valid element in the document
          isTabbingBackwards
            ? headFocusProxyRef.value?.focus()
            : tailFocusProxyRef.value?.focus()
        }
      }
    }

    viewport.addEventListener('focusin', handlePause)
    viewport.addEventListener('focusout', handleFocusOutResume)
    viewport.addEventListener('pointermove', handlePause)
    viewport.addEventListener('pointerleave', handlePointerLeaveResume)
    viewport.addEventListener('keydown', handleKeyDown)
    window.addEventListener('blur', handlePause)
    window.addEventListener('focus', handleResume)

    cleanupFn(() => {
      viewport.removeEventListener('focusin', handlePause)
      viewport.removeEventListener('focusout', handleFocusOutResume)
      viewport.removeEventListener('pointermove', handlePause)
      viewport.removeEventListener('pointerleave', handlePointerLeaveResume)
      viewport.removeEventListener('keydown', handleKeyDown)
      window.removeEventListener('blur', handlePause)
      window.removeEventListener('focus', handleResume)
    })
  }
})

function getSortedTabbableCandidates({ tabbingDirection }: { tabbingDirection: 'forwards' | 'backwards' }) {
  const toastItems = collections.value
  const tabbableCandidates = toastItems.map((toastNode) => {
    const toastTabbableCandidates = [toastNode, ...getTabbableCandidates(toastNode)]
    return tabbingDirection === 'forwards'
      ? toastTabbableCandidates
      : toastTabbableCandidates.reverse()
  })
  return (
    tabbingDirection === 'forwards' ? tabbableCandidates.reverse() : tabbableCandidates
  ).flat()
}
</script>

<script lang="ts">
export default {
  inheritAttrs: false,
}
</script>

<template>
  <DismissableLayerBranch
    role="region"
    :aria-label="label.replace('{hotkey}', hotkey.join('+').replace(/Key/g, '').replace(/Digit/g, ''))"
    tabindex="-1"
    :style="{
      // incase list has size when empty (e.g. padding), we remove pointer events so
      // it doesn't prevent interactions with page elements that it overlays
      pointerEvents: hasToasts ? undefined : 'none',
    }"
  >
    <FocusProxy
      v-if="hasToasts"
      :ref="(node: ComponentPublicInstance) => {
        headFocusProxyRef = unrefElement(node) as HTMLElement
        return undefined
      }"
      @focus-from-outside-viewport="() => {
        const tabbableCandidates = getSortedTabbableCandidates({
          tabbingDirection: 'forwards',
        })
        focusFirst(tabbableCandidates)
      }"
    />
    <Primitive
      ref="primitiveElement"
      tabindex="-1"
      :as="as"
      :as-child="asChild"
      v-bind="$attrs"
    >
      <slot />
    </Primitive>
    <FocusProxy
      v-if="hasToasts"
      :ref="(node: ComponentPublicInstance) => {
        tailFocusProxyRef = unrefElement(node) as HTMLElement
        return undefined
      }"
      @focus-from-outside-viewport="() => {
        const tabbableCandidates = getSortedTabbableCandidates({
          tabbingDirection: 'backwards',
        })
        focusFirst(tabbableCandidates)
      }"
    />
  </DismissableLayerBranch>
</template>