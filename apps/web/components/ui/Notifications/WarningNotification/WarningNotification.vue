<template>
  <div
    role="alert"
    data-testid="WarningNotification"
    class="flex items-start md:items-center max-w-[600px] shadow-md pr-2 pl-4 ring-1 typography-text-sm md:typography-text-base py-1 rounded-md bg-warning-100 ring-warning-300"
  >
    <SfIconWarning class="mr-2 text-warning-700 shrink-0" />

    <p class="py-2 mr-2">{{ notification.message }}</p>
    <button
      v-if="notification?.action?.onClick"
      @click="notification?.action?.onClick"
      type="button"
      class="py-1.5 px-3 md:py-2 md:px-4 rounded-md ml-auto font-medium focus-visible:outline focus-visible:outline-offset text-warning-700 hover:bg-warning-200 active:bg-warning-300 hover:text-warning-800 active:text-warning-900"
    >
      {{ notification?.action?.text }}
    </button>
    <button
      type="button"
      @click="notification.dismiss"
      class="p-1.5 md:p-2 ml-2 rounded-md focus-visible:outline focus-visible:outline-offset text-warning-700 hover:bg-warning-200 active:bg-warning-300 hover:text-warning-800 active:text-warning-900"
      :class="{ 'ml-auto': !notification?.action?.text }"
      aria-label="Close warning alert"
    >
      <SfIconClose class="hidden md:block" />
      <SfIconClose size="sm" class="block md:hidden" />
    </button>
  </div>
</template>
<script setup lang="ts">
import { SfIconClose, SfIconWarning } from '@storefront-ui/vue';
import { WarningNotificationProps } from './types';

defineProps<WarningNotificationProps>();
</script>
