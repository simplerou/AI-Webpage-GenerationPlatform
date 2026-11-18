<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import axios from 'axios'
import { message } from 'ant-design-vue'
import { useI18n } from 'vue-i18n'
import { translateZhToEn } from '@/api/translateController'

type TranslateTarget = HTMLInputElement | HTMLTextAreaElement

const { t } = useI18n()
const activeEl = ref<TranslateTarget | null>(null)
const visible = ref(false)
const loading = ref(false)
const position = ref({ top: 0, left: 0 })

const BUTTON_WIDTH = 124
const BUTTON_HEIGHT = 28
const BUTTON_MARGIN = 8

const containsChinese = (value: string) => /[\u4e00-\u9fff]/.test(value)

const isTranslateTarget = (el: Element | null): el is TranslateTarget => {
  if (!el) return false
  if (el instanceof HTMLElement) {
    const disabledBySelf = el.getAttribute('data-translate')
    if (disabledBySelf === 'false' || disabledBySelf === 'off' || disabledBySelf === '0') return false
    if (el.closest('[data-translate="false"], [data-translate="off"], [data-translate="0"]')) return false
    if (el.closest('.monaco-editor, .cm-editor, .ace_editor, .CodeMirror')) return false
  }
  if (el instanceof HTMLInputElement) {
    const type = (el.type || 'text').toLowerCase()
    if (
      [
        'password',
        'number',
        'date',
        'time',
        'datetime-local',
        'file',
        'checkbox',
        'radio',
        'range',
        'color',
      ].includes(type)
    ) {
      return false
    }
    if (el.closest('.ant-input-number')) return false
    if (el.readOnly || el.disabled) return false
    return true
  }
  if (el instanceof HTMLTextAreaElement) {
    if (el.readOnly || el.disabled) return false
    return true
  }
  return false
}

const clamp = (value: number, min: number, max: number) => Math.min(Math.max(value, min), max)

const updatePosition = () => {
  const el = activeEl.value
  if (!el) return
  const rect = el.getBoundingClientRect()
  if (rect.width <= 0 || rect.height <= 0) {
    visible.value = false
    return
  }
  let left = rect.right + BUTTON_MARGIN
  if (left + BUTTON_WIDTH > window.innerWidth - BUTTON_MARGIN) {
    left = rect.right - BUTTON_WIDTH - BUTTON_MARGIN
  }
  let top = rect.top + BUTTON_MARGIN
  if (top + BUTTON_HEIGHT > window.innerHeight - BUTTON_MARGIN) {
    top = rect.bottom - BUTTON_HEIGHT - BUTTON_MARGIN
  }
  left = clamp(left, BUTTON_MARGIN, window.innerWidth - BUTTON_WIDTH - BUTTON_MARGIN)
  top = clamp(top, BUTTON_MARGIN, window.innerHeight - BUTTON_HEIGHT - BUTTON_MARGIN)
  position.value = { top, left }
}

const updateState = () => {
  const el = activeEl.value
  if (!el) {
    visible.value = false
    return
  }
  visible.value = containsChinese(el.value || '')
  if (visible.value) {
    updatePosition()
  }
}

const applyValue = (el: TranslateTarget, value: string) => {
  el.value = value
  el.dispatchEvent(new Event('input', { bubbles: true }))
  el.dispatchEvent(new Event('change', { bubbles: true }))
}

const handleTranslate = async () => {
  const el = activeEl.value
  if (!el || loading.value) return
  const text = el.value || ''
  if (!containsChinese(text)) {
    visible.value = false
    return
  }
  loading.value = true
  try {
    const res = await translateZhToEn({ text })
    const data = res?.data
    if (data?.code === 0 && typeof data.data === 'string') {
      applyValue(el, data.data)
      updateState()
      return
    }
    message.error(data?.message || t('common.translationFailed'))
  } catch (err: unknown) {
    if (axios.isAxiosError(err) && err.code === 'ECONNABORTED') {
      message.error(t('common.translationTimedOut'))
    } else {
      message.error(t('common.translationFailed'))
    }
  } finally {
    loading.value = false
  }
}

const handleFocusIn = (event: FocusEvent) => {
  const target = event.target as Element | null
  if (!isTranslateTarget(target)) {
    activeEl.value = null
    visible.value = false
    return
  }
  activeEl.value = target
  updateState()
}

const handleFocusOut = () => {
  setTimeout(() => {
    const current = document.activeElement as Element | null
    if (isTranslateTarget(current)) {
      activeEl.value = current
      updateState()
      return
    }
    activeEl.value = null
    visible.value = false
  }, 0)
}

const handleInput = (event: Event) => {
  if (event.target === activeEl.value) {
    updateState()
  }
}

const handleScroll = () => {
  if (visible.value) {
    updatePosition()
  }
}

onMounted(() => {
  document.addEventListener('focusin', handleFocusIn, true)
  document.addEventListener('focusout', handleFocusOut, true)
  document.addEventListener('input', handleInput, true)
  window.addEventListener('scroll', handleScroll, true)
  window.addEventListener('resize', handleScroll)
})

onBeforeUnmount(() => {
  document.removeEventListener('focusin', handleFocusIn, true)
  document.removeEventListener('focusout', handleFocusOut, true)
  document.removeEventListener('input', handleInput, true)
  window.removeEventListener('scroll', handleScroll, true)
  window.removeEventListener('resize', handleScroll)
})
</script>

<template>
  <div
    v-if="visible"
    class="translate-fab"
    :style="{ top: `${position.top}px`, left: `${position.left}px` }"
  >
    <a-button size="small" type="primary" :loading="loading" @mousedown.prevent @click="handleTranslate">
      {{ loading ? t('common.translating') : t('common.translateToEn') }}
    </a-button>
  </div>
</template>

<style scoped>
.translate-fab {
  position: fixed;
  z-index: 2000;
  pointer-events: auto;
}

.translate-fab :deep(.ant-btn) {
  height: 28px;
  padding: 0 10px;
  font-size: 12px;
  border-radius: 999px;
  box-shadow: 0 6px 16px rgba(15, 23, 42, 0.12);
}
</style>
