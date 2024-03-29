<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'

const frameRef = ref<HTMLIFrameElement>()
const props = defineProps<{ content: string | undefined }>()
const emit = defineEmits<{ alert: [value: any] }>()

watch(
  () => props.content,
  (value) => {
    frameRef?.value?.contentWindow?.postMessage({
      type: 'html',
      value,
    })
  }
)

// This function gets invoked inside the iframe for initialisation.
const initFrame = () => {
  if (!window.frameElement) return
  window.alert = (value) => {
    window.parent.postMessage({ type: 'alert', value }, '*')
  }
  window.addEventListener('error', (ev) => {
    console.warn(ev.error)
    ev.preventDefault()
  })
  window.addEventListener('message', (event) => {
    const { type, value } = event.data
    if (type === 'html') {
      document.body.innerHTML = value
      Array.from(document.body.querySelectorAll('script')).forEach((script) => {
        const scriptEl = document.createElement('script')
        if (script.hasAttribute('src')) {
          scriptEl.src = script.src
        }
        scriptEl.text = script.text
        document.body.appendChild(scriptEl).parentNode?.removeChild(scriptEl)
      })
      // Prevent links from opening inside the iframe.
      document.querySelectorAll<HTMLAnchorElement>('a').forEach((a) => {
        if (!a.href.startsWith('javascript:')) {
          a.target = '_blank'
        }
      })
    }
  })
}

const handleMessage = (event: MessageEvent) => {
  if (event.source === frameRef.value?.contentWindow) {
    event.stopImmediatePropagation()
    const { type, value } = event.data
    if (type === 'alert') {
      emit('alert', value)
    }
  }
}

onMounted(() => {
  const doc = frameRef.value?.contentDocument
  if (doc) {
    const script = document.createElement('script')
    script.text = `(${initFrame})();`
    doc.body.innerHTML = props.content || ''
    doc.body.appendChild(script)
    const style = document.createElement('style')
    style.innerText = `
body {
  font-size: 28px; font-family: Calibri, sans-serif;
}
input {
  font-size: 24px;
}
`
    doc.head.appendChild(style)
  }
  window.addEventListener('message', handleMessage)
  return () => window.removeEventListener('message', handleMessage)
})
</script>

<template>
  <iframe ref="frameRef"></iframe>
</template>

<style scoped>
iframe {
  display: block;
  width: 100%;
  height: 100%;
}
</style>
