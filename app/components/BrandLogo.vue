<script setup lang="ts">
import logo from '~/assets/logo1.png'

const props = withDefaults(defineProps<{
  size?: 'small' | 'medium' | 'large' | 'hero'
}>(), {
  size: 'medium',
})
</script>

<template>
  <img :src="logo" alt="DAN’STAMP" class="brand-logo" :class="`brand-logo--${props.size}`" />
</template>

<style scoped>
.brand-logo {
  display: block;
  flex: 0 0 auto;
  object-fit: contain;
}

.brand-logo--small {
  width: 30px;
  height: 30px;
}

.brand-logo--medium {
  width: 44px;
  height: 44px;
}

.brand-logo--large {
  width: 64px;
  height: 64px;
}

.brand-logo--hero {
  width: 148px;
  height: 148px;
}
</style>
