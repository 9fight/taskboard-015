<template>
  <div ref="root" class="custom-select" :class="{ open, compact, block }">
    <button
      class="select-trigger"
      type="button"
      :aria-expanded="open"
      aria-haspopup="listbox"
      @click="open = !open"
      @keydown.esc="open = false"
    >
      <span class="option-label">
        <i v-if="selectedOption?.tone" :class="selectedOption.tone" />
        {{ selectedOption?.label }}
      </span>
      <svg viewBox="0 0 24 24" aria-hidden="true"><path d="m7 9 5 5 5-5" /></svg>
    </button>

    <Transition name="dropdown">
      <div v-if="open" class="select-menu" role="listbox">
        <button
          v-for="option in options"
          :key="option.value"
          type="button"
          class="select-option"
          :class="{ selected: option.value === modelValue }"
          role="option"
          :aria-selected="option.value === modelValue"
          @click="selectOption(option.value)"
        >
          <span class="option-label">
            <i v-if="option.tone" :class="option.tone" />
            {{ option.label }}
          </span>
          <svg v-if="option.value === modelValue" viewBox="0 0 24 24"><path d="m5 12 4 4L19 6" /></svg>
        </button>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const props = defineProps({
  modelValue: { type: String, default: '' },
  options: { type: Array, required: true },
  compact: Boolean,
  block: Boolean,
})
const emit = defineEmits(['update:modelValue'])
const open = ref(false)
const root = ref(null)
const selectedOption = computed(() => props.options.find((option) => option.value === props.modelValue) || props.options[0])

function selectOption(value) {
  emit('update:modelValue', value)
  open.value = false
}

function closeOnOutside(event) {
  if (!root.value?.contains(event.target)) open.value = false
}

onMounted(() => document.addEventListener('pointerdown', closeOnOutside))
onBeforeUnmount(() => document.removeEventListener('pointerdown', closeOnOutside))
</script>
