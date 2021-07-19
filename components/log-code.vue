<template>
  <abbr :title="title">{{ props.code }}{{ props.reason ? ` (${props.reason})` : '' }}</abbr>
</template>
<script lang="ts" setup>
import { defineProps, computed } from 'vue'

const props = defineProps({
  code: String,
  reason: String
})

const title = computed(() => {
  if (props.code === 'RESOLVE') {
    return 'Resolving from this domain.'
  }
  if (props.code === 'REDIRECT') {
    return 'Redirecting away from this domain.'
  }
  if (props.code === 'UNUSED_ENTRY') {
    return 'A redirect resulted in this entry to be ignored.'
  }
  if (props.code === 'CONFLICT_ENTRY') {
    return 'Another entry with the same key was preferred over this entry because the other entry is alphabetically earlier.'
  }
  if (props.code === 'ENDLESS_REDIRECT') {
    return 'Detected endless redirect.'
  }
  if (props.code === 'RECURSIVE_DNSLINK_PREFIX') {
    return 'Only one _dnslink. prefix can be used.'
  }
  if (props.code === 'INVALID_ENTRY') {
    if (props.reason === 'NO_VALUE') {
      return 'An dnslink entry needs to have a value, like: dnslink=/key/value.'
    }
    if (props.reason === 'WRONG_START') {
      return 'A dnslink entry needs to start with a /.'
    }
    if (props.reason === 'KEY_MISSING') {
      return 'A dnslink entry nees to have a key, like: dnslink=/key/value.'
    }
    return 'Entry misformatted, cant be used.'
  }
  return ''
})
</script>