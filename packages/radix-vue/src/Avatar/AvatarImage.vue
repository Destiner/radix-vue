<script setup lang="ts">
import { inject, onMounted } from "vue";
import {
  AVATAR_INJECTION_KEY,
  type AvatarProvideValue,
} from "./AvatarRoot.vue";

const injectedValue = inject<AvatarProvideValue>(AVATAR_INJECTION_KEY);

onMounted(() => {
  injectedValue!.imageLoadingStatus.value = "loaded";
});
</script>

<template>
  <img />
</template>