<template>
  <div>
    <NuxtPage />
  </div>
</template>

<script lang="ts" setup>
import { useScriptTag } from '@vueuse/core'

const config = useRuntimeConfig()

if (config.public.FATHOM) {
  useScriptTag('https://cdn.usefathom.com/script.js', () => {}, {
    attrs: {
      'data-site': config.public.FATHOM as string,
    },
    defer: true,
  })
}
</script>

<style lang="scss">
@use '~/scss/styles.scss';
</style>