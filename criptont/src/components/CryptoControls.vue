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
  <form @submit.prevent>
    <label for="market">Mercado</label>
    <select id="market" v-model="selectedProduct" @change="publishConfig">
      <option v-for="market in markets" :key="market.id" :value="market.id">
        {{ market.displayName }}
      </option>
    </select>

    <button type="button" :class="{ stopped: !pollingEnabled }" @click="pollingEnabled = !pollingEnabled; publishConfig()">
      {{ pollingEnabled ? 'Detener actualizaciones' : 'Reanudar actualizaciones' }}
    </button>

    <p class="hint">
      Vue controla este formulario y avisa los cambios a la isla Svelte.
    </p>
  </form>
</template>

<style scoped>
form {
  display: grid;
  gap: 12px;
  margin-top: 24px;
}

label {
  color: #3c465b;
  font-size: 0.88rem;
  font-weight: 700;
}

select,
button {
  width: 100%;
  min-height: 46px;
  border-radius: 10px;
}

select {
  padding: 0 12px;
  color: #172033;
  background: #f8f9fc;
  border: 1px solid #d9dfeb;
}

button {
  margin-top: 6px;
  color: white;
  background: #635bff;
  border: 0;
  cursor: pointer;
  font-weight: 700;
}

button:hover {
  background: #5148ed;
}

button.stopped {
  color: #394258;
  background: #e8ebf2;
}

.hint {
  margin: 4px 0 0;
  color: #7a8498;
  font-size: 0.82rem;
  line-height: 1.5;
}
</style>
