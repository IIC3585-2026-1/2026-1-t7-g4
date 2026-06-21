<script lang="ts">
  import { onMount } from 'svelte';

  export let initialProductId: string;

  type DashboardConfig = {
    productId: string;
    pollingEnabled: boolean;
  };

  type Ticker = {
    price: number;
    bid: number;
    ask: number;
    volume: number;
    time: string;
  };

  type Candle = {
    time: number;
    low: number;
    high: number;
    open: number;
    close: number;
    volume: number;
  };

  let productId = initialProductId;
  let pollingEnabled = true;
  let ticker: Ticker | null = null;
  let candles: Candle[] = [];
  let status: 'loading' | 'success' | 'paused' | 'error' = 'loading';
  let errorMessage = '';
  let timer: ReturnType<typeof setTimeout> | undefined;
  let controller: AbortController | undefined;
  let requestVersion = 0;

  const money = new Intl.NumberFormat('es-CL', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 2,
  });

  const number = new Intl.NumberFormat('es-CL', {
    maximumFractionDigits: 2,
  });

  function clearCurrentRequest() {
    if (timer) clearTimeout(timer);
    controller?.abort();
    timer = undefined;
  }

  async function fetchTicker(signal: AbortSignal): Promise<Ticker> {
    const response = await fetch(`https://api.exchange.coinbase.com/products/${productId}/ticker`, { signal });
    if (!response.ok) throw new Error('No fue posible obtener el precio actual.');
    const data = await response.json();

    return {
      price: Number(data.price),
      bid: Number(data.bid),
      ask: Number(data.ask),
      volume: Number(data.volume),
      time: data.time,
    };
  }

  async function fetchCandles(signal: AbortSignal): Promise<Candle[]> {
    const end = new Date();
    const start = new Date(end.getTime() - 24 * 60 * 60 * 1000);
    const params = new URLSearchParams({
      start: start.toISOString(),
      end: end.toISOString(),
      granularity: '3600',
    });
    const response = await fetch(
      `https://api.exchange.coinbase.com/products/${productId}/candles?${params}`,
      { signal },
    );
    if (!response.ok) throw new Error('No fue posible obtener el historial.');
    const data: number[][] = await response.json();

    return data
      .map(([time, low, high, open, close, volume]) => ({ time, low, high, open, close, volume }))
      .sort((a, b) => a.time - b.time);
  }

  async function loadMarket() {
    clearCurrentRequest();
    const version = ++requestVersion;
    controller = new AbortController();
    status = 'loading';
    errorMessage = '';

    try {
      const [nextTicker, nextCandles] = await Promise.all([
        fetchTicker(controller.signal),
        fetchCandles(controller.signal),
      ]);
      if (version !== requestVersion) return;

      ticker = nextTicker;
      candles = nextCandles;
      status = pollingEnabled ? 'success' : 'paused';
      if (pollingEnabled) schedulePoll(version);
    } catch (error) {
      if (error instanceof DOMException && error.name === 'AbortError') return;
      status = 'error';
      errorMessage = error instanceof Error ? error.message : 'Ocurrió un error inesperado.';
    }
  }

  function schedulePoll(version: number) {
    timer = setTimeout(async () => {
      if (!pollingEnabled || version !== requestVersion) return;
      controller = new AbortController();

      try {
        ticker = await fetchTicker(controller.signal);
        status = 'success';
        errorMessage = '';
      } catch (error) {
        if (!(error instanceof DOMException && error.name === 'AbortError')) {
          status = 'error';
          errorMessage = 'No se pudo actualizar. Se conserva el último precio.';
        }
      } finally {
        if (pollingEnabled && version === requestVersion) schedulePoll(version);
      }
    }, 5000);
  }

  function handleConfig(event: Event) {
    const config = (event as CustomEvent<DashboardConfig>).detail;
    const marketChanged = config.productId !== productId;
    const pollingChanged = config.pollingEnabled !== pollingEnabled;
    productId = config.productId;
    pollingEnabled = config.pollingEnabled;

    if (!marketChanged && !pollingChanged) return;

    if (marketChanged) {
      loadMarket();
    } else if (!pollingEnabled) {
      clearCurrentRequest();
      requestVersion += 1;
      status = 'paused';
    } else {
      loadMarket();
    }
  }

  function chartPoints() {
    if (candles.length < 2) return '';
    const closes = candles.map((candle) => candle.close);
    const min = Math.min(...closes);
    const max = Math.max(...closes);
    const range = max - min || 1;

    return closes
      .map((close, index) => {
        const x = (index / (closes.length - 1)) * 700;
        const y = 210 - ((close - min) / range) * 180;
        return `${x},${y}`;
      })
      .join(' ');
  }

  onMount(() => {
    window.addEventListener('crypto:config-change', handleConfig);
    window.dispatchEvent(new CustomEvent('crypto:config-request'));

    // También carga el valor inicial si Vue tarda más en hidratarse.
    loadMarket();

    return () => {
      window.removeEventListener('crypto:config-change', handleConfig);
      clearCurrentRequest();
    };
  });
</script>

<div class="mt-5 flex items-center gap-3 rounded-2xl border border-slate-200 bg-slate-50 px-4 py-3 text-sm text-slate-600">
  <span
    class={`h-2.5 w-2.5 rounded-full ${status === 'success' ? 'bg-green-500' : status === 'paused' ? 'bg-yellow-500' : 'bg-slate-400'}`}
  ></span>
  <span class="font-medium text-slate-700">
    {status === 'loading' ? 'Cargando…' : status === 'paused' ? 'Actualizaciones detenidas' : status === 'error' ? 'Problema de conexión' : 'Actualización automática'}
  </span>
</div>

{#if ticker}
  <div class="mt-5 flex flex-wrap items-end justify-between gap-4 border-b border-slate-100 pb-5">
    <div class="grid gap-1">
      <span class="text-xs font-extrabold uppercase tracking-[0.2em] text-slate-500">{productId}</span>
      <strong class="text-3xl font-semibold tracking-tight text-slate-950 sm:text-4xl lg:text-5xl">{money.format(ticker.price)}</strong>
    </div>
    <small class="text-sm text-slate-500">Actualizado {new Date(ticker.time).toLocaleTimeString('es-CL')}</small>
  </div>

  <div class="mt-5 grid gap-3 sm:grid-cols-3">
    <div class="grid gap-2 rounded-2xl border border-slate-100 bg-white p-4 shadow-sm">
      <span class="text-xs font-medium uppercase tracking-[0.16em] text-slate-500">Compra</span>
      <strong class="text-sm font-semibold text-slate-900">{money.format(ticker.bid)}</strong>
    </div>
    <div class="grid gap-2 rounded-2xl border border-slate-100 bg-white p-4 shadow-sm">
      <span class="text-xs font-medium uppercase tracking-[0.16em] text-slate-500">Venta</span>
      <strong class="text-sm font-semibold text-slate-900">{money.format(ticker.ask)}</strong>
    </div>
    <div class="grid gap-2 rounded-2xl border border-slate-100 bg-white p-4 shadow-sm">
      <span class="text-xs font-medium uppercase tracking-[0.16em] text-slate-500">Volumen 24 h</span>
      <strong class="text-sm font-semibold text-slate-900">{number.format(ticker.volume)}</strong>
    </div>
  </div>
{:else if status === 'loading'}
  <div class="mt-5 h-[230px] animate-pulse rounded-2xl border border-slate-100 bg-gradient-to-r from-slate-200 via-slate-100 to-slate-200" aria-label="Cargando datos"></div>
{/if}

{#if errorMessage}
  <p class="mt-4 rounded-xl border border-rose-200 bg-rose-50 px-4 py-3 text-sm text-rose-700" role="status">{errorMessage}</p>
{/if}

{#if candles.length > 1}
  <div class="mt-7 flex items-end justify-between gap-3">
    <h3 class="text-base font-semibold text-slate-950">Últimas 24 horas</h3>
    <span class="text-xs text-slate-500">{candles.length} puntos por hora</span>
  </div>
  <div class="mt-3 overflow-hidden rounded-2xl border border-slate-100 bg-gradient-to-b from-indigo-50 via-white to-white shadow-sm">
    <svg class="block h-[220px] w-full" viewBox="0 0 700 240" role="img" aria-label={`Evolución de ${productId} durante las últimas 24 horas`}>
      <line class="stroke-slate-200 stroke-2" x1="0" y1="210" x2="700" y2="210"></line>
      <polyline class="fill-none stroke-indigo-600 stroke-[5] [stroke-linecap:round] [stroke-linejoin:round]" points={chartPoints()}></polyline>
    </svg>
  </div>
{/if}
