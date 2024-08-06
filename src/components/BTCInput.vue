<template>
  <div class="btc-input__wrapper">
    <label class="btc-input__label">BTC Value</label>
    <input
      v-model="btcValue"
      class="btc-input__input"
      :class="btcInputClasses"
      @keydown="handleKeyDown"
      @paste="handlePaste"
    />
    <p v-if="error" class="btc-input__error-message">{{ errorMessage }}</p>
    <template v-if="usdValue">
      <p class="btc-input__price">USD Value: {{ usdValue }}</p>
      <p class="btc-input__asof">As of: {{ displayDate }}</p>
    </template>
  </div>
</template>

<script setup>
import { computed, onMounted, ref } from 'vue';

const ERROR_MESSAGE = 'Invalid input. Only digits and up to 18 decimals allowed.';
const REGEX = /^[0-9]*\.?\d{0,18}$/;
const ALLOWED_KEYS = ['Backspace', 'Delete', 'ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown', 'Tab', 'Meta', 'Shift', 'Control', 'Alt'];

const btcValue = ref('');
const btcPrice = ref(null);
const btcPriceAsOf = ref(null);
const error = ref(false);
const errorMessage = ref('');

const usdValue = computed(() => {
  if (btcValue.value === '') return null;
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(btcValue.value * btcPrice.value);
});

const displayDate = computed(() => {
  if (btcPriceAsOf.value === null) return null;
  return new Date(btcPriceAsOf.value).toLocaleString();
});

const btcInputClasses = computed(() => ({
  'btc-input__input--error': error.value,
}));

const showError = (message, delay = 2000) => {
  error.value = true;
  errorMessage.value = message;
  setTimeout(() => {
    error.value = false;
    errorMessage.value = '';
  }, delay);
};

const handleKeyDown = (e) => {
  // check if more than 2 minutes passed after last fetch
  const lastFetch = new Date(btcPriceAsOf.value);
  const now = new Date();
  if (now - lastFetch > 2 * 60 * 1000) {
    fetchBTCPrice();
  }

  // allow copy, cut, paste, undo, redo
  if ((e.metaKey || e.ctrlKey) && ['a', 'c', 'x', 'v', 'z'].includes(e.key)) return;

  if (ALLOWED_KEYS.includes(e.key)) return;

  if (!REGEX.test(btcValue.value + e.key)) {
    e.preventDefault();
    showError(ERROR_MESSAGE);
    return;
  }

  error.value = false;
  errorMessage.value = '';
};

const handlePaste = (e) => {
  const pastedData = (e.clipboardData || window.clipboardData).getData('Text');
  
  if (!REGEX.test(pastedData)) {
    e.preventDefault();
    showError(ERROR_MESSAGE);
  }
};

const fetchBTCPrice = async () => {
  if (error.value) return;
  try {
    const response = await fetch('https://api.coindesk.com/v1/bpi/currentprice/BTC.json');
    const data = await response.json();
    btcPrice.value = data.bpi.USD.rate_float;
    btcPriceAsOf.value = data.time.updatedISO;
  } catch (err) {
    console.error('Error fetching BTC price:', err);
  }
};

onMounted(fetchBTCPrice);
</script>

<style scoped lang="postcss">
.btc-input {
  &__wrapper {
    @apply max-w-sm mx-auto;
  }

  &__label {
    @apply block text-gray-700;
  }

  &__input {
    @apply mt-1 block w-full px-3 py-2;
    @apply border rounded-md shadow-sm focus:outline-none;

    &--error {
      @apply border-red-500;
    }
  }

  &__error-message {
    @apply text-red-500 text-sm mt-1;
  }

  &__price {
    @apply mt-4;
  }

  &__asof {
    @apply mt-1;
    @apply text-sm text-gray-500;
  }
}
</style>
