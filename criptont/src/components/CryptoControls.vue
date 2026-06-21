<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue';

type Market = {
  id: string;
  displayName: string;
};

const props = defineProps<{
  markets: Market[];
  initialProductId: string;
}>();

const selectedProduct = ref(props.initialProductId);
const pollingEnabled = ref(true);

function publishConfig() {
  window.dispatchEvent(new CustomEvent('crypto:config-change', {
    detail: {
      productId: selectedProduct.value,
      pollingEnabled: pollingEnabled.value,
    },
  }));
}

function answerConfigRequest() {
  publishConfig();
}

onMounted(() => {
  window.addEventListener('crypto:config-request', answerConfigRequest);
  publishConfig();
});

onUnmounted(() => {
  window.removeEventListener('crypto:config-request', answerConfigRequest);
});
</script>

<template>
  <form class="mt-6 grid gap-4" @submit.prevent>
    <div class="grid gap-2">
      <label for="market" class="text-sm font-semibold text-slate-700">Mercado</label>
      <select
        id="market"
        v-model="selectedProduct"
        class="h-12 w-full rounded-xl border border-slate-200 bg-white px-4 text-sm text-slate-900 shadow-sm outline-none transition duration-200 hover:border-slate-300 focus:border-indigo-500 focus:ring-4 focus:ring-indigo-100"
        @change="publishConfig"
      >
        <option v-for="market in markets" :key="market.id" :value="market.id">
          {{ market.displayName }}
        </option>
      </select>
    </div>

    <button
      type="button"
      class="inline-flex h-12 w-full items-center justify-center rounded-xl px-4 text-sm font-semibold shadow-sm transition duration-200 focus:outline-none focus:ring-4"
      :class="pollingEnabled
        ? 'bg-indigo-600 text-white hover:bg-indigo-500 focus:ring-indigo-100'
        : 'bg-slate-100 text-slate-700 hover:bg-slate-200 focus:ring-slate-200'"
      @click="pollingEnabled = !pollingEnabled; publishConfig()"
    >
      <span
        class="mr-2 h-2.5 w-2.5 rounded-full"
        :class="pollingEnabled ? 'bg-green-500' : 'bg-yellow-500'"
        aria-hidden="true"
      ></span>
      {{ pollingEnabled ? 'Detener actualizaciones' : 'Reanudar actualizaciones' }}
    </button>

    <p class="text-sm leading-6 text-slate-500">
      Vue controla este formulario y avisa los cambios a la isla Svelte.
    </p>
  </form>
</template>
