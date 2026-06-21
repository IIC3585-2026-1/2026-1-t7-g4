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

  const compactMoney = new Intl.NumberFormat('es-CL', {
    style: 'currency',
    currency: 'USD',
    notation: 'compact',
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
      ticker = null;
      candles = [];
      loadMarket();
    } else if (!pollingEnabled) {
      clearCurrentRequest();
      requestVersion += 1;
      status = 'paused';
    } else {
      loadMarket();
    }
  }

  function formatChartTime(timestamp: number) {
    return new Date(timestamp * 1000).toLocaleString('es-CL', {
      day: '2-digit',
      month: 'short',
      hour: '2-digit',
      minute: '2-digit',
    });
  }

  function buildChart(data: Candle[]) {
    if (data.length < 2) {
      return { points: '', min: 0, max: 0, start: '', middle: '', end: '' };
    }

    const closes = data.map((candle) => candle.close);
    const min = Math.min(...closes);
    const max = Math.max(...closes);
    const range = max - min || 1;

    const points = closes
      .map((close, index) => {
        const x = 75 + (index / (closes.length - 1)) * 605;
        const y = 190 - ((close - min) / range) * 160;
        return `${x},${y}`;
      })
      .join(' ');

    return {
      points,
      min,
      max,
      start: formatChartTime(data[0].time),
      middle: formatChartTime(data[Math.floor(data.length / 2)].time),
      end: formatChartTime(data[data.length - 1].time),
    };
  }

  $: chart = buildChart(candles);

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

<div class="status-row">
  <span class:online={status === 'success'} class:paused={status === 'paused'} class="status-dot"></span>
  <span>
    {status === 'loading' ? 'Cargando…' : status === 'paused' ? 'Actualizaciones detenidas' : status === 'error' ? 'Problema de conexión' : 'Actualización automática'}
  </span>
</div>

{#if ticker}
  <div class="price-row">
    <div>
      <span class="label">{productId}</span>
      <strong>{money.format(ticker.price)}</strong>
    </div>
    <small>Actualizado {new Date(ticker.time).toLocaleTimeString('es-CL')}</small>
  </div>

  <div class="metrics">
    <div><span>Compra</span><strong>{money.format(ticker.bid)}</strong></div>
    <div><span>Venta</span><strong>{money.format(ticker.ask)}</strong></div>
    <div><span>Volumen 24 h</span><strong>{number.format(ticker.volume)}</strong></div>
  </div>
{:else if status === 'loading'}
  <div class="skeleton" aria-label="Cargando datos"></div>
{/if}

{#if errorMessage}
  <p class="error" role="status">{errorMessage}</p>
{/if}

{#if candles.length > 1}
  <div class="chart-header">
    <h3>{productId} · últimas 24 horas</h3>
    <span>{candles.length} puntos por hora</span>
  </div>
  <div class="chart">
    {#key productId}
      <svg viewBox="0 0 700 240" role="img" aria-label={`Evolución de ${productId} durante las últimas 24 horas`}>
        <line class="grid-line" x1="75" y1="30" x2="680" y2="30"></line>
        <line class="grid-line" x1="75" y1="110" x2="680" y2="110"></line>
        <line class="axis-line" x1="75" y1="190" x2="680" y2="190"></line>
        <line class="axis-line" x1="75" y1="30" x2="75" y2="190"></line>

        <text class="axis-label y-label" x="67" y="34">{compactMoney.format(chart.max)}</text>
        <text class="axis-label y-label" x="67" y="194">{compactMoney.format(chart.min)}</text>
        <text class="axis-title" x="18" y="115" transform="rotate(-90 18 115)">Precio USD</text>

        <text class="axis-label x-label" x="75" y="218">{chart.start}</text>
        <text class="axis-label x-label middle" x="377" y="218">{chart.middle}</text>
        <text class="axis-label x-label end" x="680" y="218">{chart.end}</text>

        <polyline points={chart.points}></polyline>
      </svg>
    {/key}
  </div>
{/if}

<style>
  .status-row,
  .price-row,
  .chart-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
  }

  .status-row {
    justify-content: flex-start;
    margin: 18px 0;
    color: #6d778c;
    font-size: 0.85rem;
  }

  .status-dot {
    width: 9px;
    height: 9px;
    background: #f0a52b;
    border-radius: 50%;
  }

  .status-dot.online { background: #1ca66c; }
  .status-dot.paused { background: #8992a4; }

  .price-row {
    align-items: end;
    padding-bottom: 20px;
  }

  .price-row div {
    display: grid;
    gap: 5px;
  }

  .price-row strong {
    font-size: clamp(2rem, 7vw, 3.3rem);
    letter-spacing: -0.05em;
  }

  .price-row small,
  .label,
  .chart-header span {
    color: #778197;
  }

  .label {
    font-size: 0.82rem;
    font-weight: 800;
    letter-spacing: 0.08em;
  }

  .metrics {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
  }

  .metrics div {
    display: grid;
    gap: 7px;
    padding: 14px;
    background: #f6f8fc;
    border-radius: 10px;
  }

  .metrics span {
    color: #778197;
    font-size: 0.76rem;
  }

  .metrics strong {
    font-size: 0.9rem;
  }

  .chart-header {
    align-items: baseline;
    margin-top: 28px;
  }

  .chart-header h3 {
    margin: 0;
    font-size: 1rem;
  }

  .chart-header span {
    font-size: 0.75rem;
  }

  .chart {
    margin-top: 8px;
    overflow: hidden;
    background: linear-gradient(180deg, #f7f6ff, #fff);
    border-radius: 12px;
  }

  svg {
    display: block;
    width: 100%;
    min-height: 190px;
  }

  .grid-line {
    stroke: #dfe3ed;
    stroke-dasharray: 5 5;
    stroke-width: 1;
  }

  .axis-line {
    stroke: #aeb6c5;
    stroke-width: 1.5;
  }

  .axis-label,
  .axis-title {
    fill: #707a8f;
    font-family: inherit;
    font-size: 11px;
  }

  .y-label {
    text-anchor: end;
  }

  .x-label.middle {
    text-anchor: middle;
  }

  .x-label.end {
    text-anchor: end;
  }

  .axis-title {
    font-size: 10px;
    text-anchor: middle;
  }

  polyline {
    fill: none;
    stroke: #635bff;
    stroke-linecap: round;
    stroke-linejoin: round;
    stroke-width: 5;
  }

  .error {
    padding: 10px 12px;
    color: #9d2d2d;
    background: #fff0f0;
    border-radius: 8px;
    font-size: 0.85rem;
  }

  .skeleton {
    height: 230px;
    background: linear-gradient(90deg, #f0f2f7, #fafbfc, #f0f2f7);
    border-radius: 12px;
  }

  @media (max-width: 560px) {
    .price-row {
      display: block;
    }

    .price-row small {
      display: block;
      margin-top: 8px;
    }

    .metrics {
      grid-template-columns: 1fr;
    }
  }
</style>
