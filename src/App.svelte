<script>
  import { onDestroy, onMount, tick } from "svelte";
  import * as d3 from "d3";
  import Sortable from "sortablejs";

  const STORAGE_KEY = "dashboardState";
  const API_BASE = import.meta.env.VITE_API_BASE_URL || "https://rest.coincap.io/v3";
  const API_KEY = import.meta.env.VITE_COINCAP_API_KEY || "";
  const MAX_RETRIES = 3;
  const BASE_BACKOFF_MS = 2000;
  const REQUEST_SPACING_MS = 350;

  const KPI_CONFIG = {
    "price-trend": {
      title: "Evolução do preço",
      build: buildPriceTrend,
      render: renderLine,
      needsCoin: true,
      allowMultiCoin: true
    },
    "cumulative-return": {
      title: "Retorno acumulado (%)",
      build: buildCumulativeReturn,
      render: renderLine,
      needsCoin: true,
      allowMultiCoin: true
    },
    "daily-change": {
      title: "Variação diária",
      build: buildDailyChange,
      render: renderDailyChange,
      needsCoin: true,
      allowMultiCoin: true
    },
    "price-distribution": {
      title: "Distribuição de preços",
      build: buildPriceDistribution,
      render: renderHistogram,
      needsCoin: true,
      allowMultiCoin: false
    },
    "up-down-donut": {
      title: "Dias subida vs descida",
      build: buildUpDownDonut,
      render: renderDonut,
      needsCoin: true,
      allowMultiCoin: false
    },
    "normalized-comparison": {
      title: "Comparação (múltiplas moedas %)",
      build: buildNormalizedComparison,
      render: renderLine,
      needsCoin: true,
      allowMultiCoin: true
    }
  };

  const CHART_COLORS = [
    "#1fb6ff",
    "#00c2a8",
    "#ffb347",
    "#ff6b6b",
    "#c4b5fd",
    "#5eead4",
    "#fcd34d",
    "#93c5fd"
  ];

  const periods = [
    { value: 1, label: "1 dia" },
    { value: 30, label: "1 mês" },
    { value: 365, label: "1 ano" }
  ];

  let availableCoins = [];
  let dashboard = [];
  let toasts = [];
  let lastUpdated = "";
  let loadingRefresh = false;
  let openCoinDropdown = null;
  let kpiList;
  let dashboardGrid;
  let marketsLoaded = false;

  function formatTime(date) {
    return date.toLocaleTimeString("pt-PT", {
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit"
    });
  }

  function delay(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  function createCardId() {
    return `chart-${Date.now()}-${Math.random().toString(36).slice(2)}`;
  }

  function showToast(message, type = "success", duration = 3500) {
    const id = createCardId();
    toasts = [...toasts, { id, message, type }];
    setTimeout(() => {
      toasts = toasts.filter((toast) => toast.id !== id);
    }, duration);
  }

  function removeToast(id) {
    toasts = toasts.filter((toast) => toast.id !== id);
  }

  class ApiError extends Error {
    constructor(status, message) {
      super(message);
      this.name = "ApiError";
      this.status = status;
    }
  }

  async function fetchJson(url) {
    for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
      let response;
      try {
        response = await fetch(url, {
          headers: {
            "Authorization": `Bearer ${API_KEY}`
          }
        });
      } catch (networkError) {
        throw new ApiError(0, "Sem ligação à API");
      }

      if (response.ok) {
        return response.json();
      }

      if (response.status === 429 && attempt < MAX_RETRIES) {
        const retryAfter = Number(response.headers.get("retry-after"));
        const wait = retryAfter
          ? retryAfter * 1000
          : BASE_BACKOFF_MS * Math.pow(2, attempt);

        showToast(
          `Limite da API atingido. Nova tentativa em ${Math.round(wait / 1000)}s...`,
          "warn",
          wait
        );
        await delay(wait);
        continue;
      }

      throw new ApiError(response.status, `Erro ${response.status} na API`);
    }
  }

  async function fetchMarkets() {
    if (marketsLoaded) {
      return;
    }

    const url = `${API_BASE}/assets?limit=50`;
    const response = await fetchJson(url);
    const markets = (response.data && Array.isArray(response.data)) ? response.data : [];
    availableCoins = markets
      .slice(0, 20)
      .map((m) => ({
        id: m.id,
        name: m.name,
        symbol: m.symbol
      }));
    marketsLoaded = true;
  }

  const chartCache = new Map();

  async function fetchMarketChart(coinId, days = 30, forceFetch = false) {
    const cacheKey = `${coinId}_${days}`;
    if (chartCache.has(cacheKey) && !forceFetch) {
      return chartCache.get(cacheKey);
    }
    const now = Date.now();
    const start = now - (days * 24 * 60 * 60 * 1000);
    const end = now;
    const url = `${API_BASE}/assets/${coinId}/history?interval=d1&start=${start}&end=${end}`;
    const response = await fetchJson(url);
    const data = (response.data && Array.isArray(response.data)) ? response.data : [];
    const chart = {
      prices: data.map((d) => [new Date(d.time).getTime(), parseFloat(d.priceUsd) || 0]),
      total_volumes: data.map((d) => [new Date(d.time).getTime(), 0])
    };
    chartCache.set(cacheKey, chart);
    return chart;
  }

  function saveState() {
    const chartStates = dashboard.map((card) => ({
      kpi: card.kpi,
      coinIds: card.coinIds,
      period: card.period
    }));
    localStorage.setItem(STORAGE_KEY, JSON.stringify({ chartStates }));
  }

  function loadState() {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return [];
    try {
      const loaded = JSON.parse(raw).chartStates || [];
      // Filter out invalid KPIs that no longer exist in KPI_CONFIG
      return loaded.filter((card) => card.kpi in KPI_CONFIG);
    } catch {
      return [];
    }
  }

  function updateCard(card) {
    dashboard = dashboard.map((item) => (item.id === card.id ? { ...card } : item));
  }

  function removeChart(cardId) {
    dashboard = dashboard.filter((card) => card.id !== cardId);
    saveState();
  }

  function getKpiActive(kpiId) {
    return dashboard.some((card) => card.kpi === kpiId);
  }

  function createLegend(svg, width, height, items, margin) {
    const legendGroup = svg
      .append("g")
      .attr("class", "legend")
      .attr("transform", `translate(${margin.left}, ${height - 15})`);

    const legendItems = legendGroup
      .selectAll(".legend-item")
      .data(items)
      .join("g")
      .attr("class", "legend-item")
      .attr("transform", (d, i) => `translate(${i * 120}, 0)`);

    legendItems
      .append("rect")
      .attr("width", 12)
      .attr("height", 12)
      .attr("fill", (d, i) => CHART_COLORS[i % CHART_COLORS.length]);

    legendItems
      .append("text")
      .attr("x", 16)
      .attr("y", 10)
      .attr("fill", "#e6eef7")
      .attr("font-size", 10)
      .text((d) => d);
  }

  function renderSvg(container, draw) {
    const rect = container.getBoundingClientRect();
    const width = Math.max(200, Math.floor(rect.width || 260));
    const height = Math.max(180, Math.floor(rect.height || 220));

    container.innerHTML = "";
    const svg = d3
      .select(container)
      .append("svg")
      .attr("width", width)
      .attr("height", height)
      .attr("viewBox", `0 0 ${width} ${height}`)
      .attr("preserveAspectRatio", "xMidYMid meet");

    draw(svg, width, height);
  }

  function renderLoading(container, message = "A carregar...") {
    container.innerHTML = `
      <div class="chart-loading">
        <div class="spinner"></div>
        <span>${message}</span>
      </div>
    `;
  }

  function renderEmpty(container, message = "Sem dados", isError = false) {
    container.innerHTML = `
      <div class="chart-empty ${isError ? "is-error" : ""}">${message}</div>
    `;
  }

  function renderStats(container, stats, coinName) {
    const statsDiv = document.createElement("div");
    statsDiv.className = "chart-stats";

    const formatValue = (val) => {
      if (val >= 1000000) return `$${(val / 1000000).toFixed(2)}M`;
      if (val >= 1000) return `$${(val / 1000).toFixed(2)}K`;
      return `$${val.toFixed(2)}`;
    };

    const changeClass = stats.change >= 0 ? "positive" : "negative";
    const changeIcon = stats.change >= 0 ? "↑" : "↓";

    statsDiv.innerHTML = `
      <div class="stats-title">${coinName}</div>
      <div class="stats-grid">
        <div class="stat-item">
          <span class="stat-label">Média</span>
          <span class="stat-value">${formatValue(stats.avg)}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Máximo</span>
          <span class="stat-value">${formatValue(stats.max)}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Mínimo</span>
          <span class="stat-value">${formatValue(stats.min)}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Variação</span>
          <span class="stat-value ${changeClass}">${changeIcon} ${Math.abs(
            stats.change
          ).toFixed(2)}%</span>
        </div>
      </div>
    `;

    container.appendChild(statsDiv);
  }

  function calculateStats(data) {
    if (!data || data.length === 0) return null;
    const values = data.map((d) => d.value || d.y || d);
    const sum = values.reduce((a, b) => a + b, 0);
    const avg = sum / values.length;
    const max = Math.max(...values);
    const min = Math.min(...values);
    const first = values[0];
    const last = values[values.length - 1];
    const change = ((last - first) / first) * 100;
    return { avg, max, min, change };
  }

  function buildPriceTrend(chartsData) {
    const allSeries = chartsData.map(({ coinName, coinSymbol, chart }) => ({
      name: `${coinName} (${coinSymbol})`,
      data: (chart?.prices || [])
        .slice(-30)
        .map((entry) => ({ time: new Date(entry[0]), value: entry[1] }))
    }));
    return { allSeries };
  }

  function buildUpDownDonut(chartsData) {
    const items = chartsData.map(({ coinName, coinSymbol, chart }) => {
      const prices = chart?.prices || [];
      let up = 0;
      let down = 0;
      for (let i = 1; i < prices.length; i++) {
        if (prices[i][1] > prices[i - 1][1]) up++;
        else down++;
      }
      return {
        label: `${coinName} (${coinSymbol})`,
        up,
        down
      };
    });
    return { items };
  }

  function buildVolumeTreemap(chartsData) {
    const items = chartsData.flatMap(({ coinName, coinSymbol, chart }) =>
      (chart?.total_volumes || [])
        .slice(-8)
        .map((entry, idx) => ({
          name: `${coinSymbol} D${idx + 1}`,
          value: entry[1],
          coin: coinName
        }))
    );
    return { items };
  }

  function buildCumulativeReturn(chartsData) {
    const allSeries = chartsData.map(({ coinName, coinSymbol, chart }) => {
      const prices = chart?.prices || [];
      if (prices.length === 0) return { name: `${coinName} (${coinSymbol})`, data: [] };
      const firstPrice = prices[0][1];
      return {
        name: `${coinName} (${coinSymbol})`,
        data: prices.map((entry) => ({
          time: new Date(entry[0]),
          value: ((entry[1] - firstPrice) / firstPrice) * 100
        }))
      };
    });
    return { allSeries };
  }

  function buildDailyChange(chartsData) {
    const allSeries = chartsData.map(({ coinName, coinSymbol, chart }) => {
      const prices = chart?.prices || [];
      return {
        name: `${coinName} (${coinSymbol})`,
        data: prices.slice(1).map((entry, idx) => ({
          date: new Date(entry[0]).toLocaleDateString("pt-PT", {
            day: "2-digit",
            month: "2-digit"
          }),
          value: ((entry[1] - prices[idx][1]) / prices[idx][1]) * 100
        }))
      };
    });
    return { allSeries };
  }

  function buildPriceDistribution(chartsData) {
    const items = chartsData.map(({ coinName, coinSymbol, chart }) => {
      const prices = (chart?.prices || []).map((p) => p[1]);
      const min = Math.min(...prices);
      const max = Math.max(...prices);
      const bins = 10;
      const binSize = (max - min) / bins;
      const histogram = Array(bins).fill(0);
      prices.forEach((price) => {
        const binIndex = Math.min(Math.floor((price - min) / binSize), bins - 1);
        histogram[binIndex]++;
      });
      return {
        label: `${coinName} (${coinSymbol})`,
        data: histogram.map((count, idx) => ({
          range: `$${(min + idx * binSize).toFixed(2)}-${(min + (idx + 1) * binSize).toFixed(2)}`,
          count
        })),
        min: min.toFixed(2),
        max: max.toFixed(2)
      };
    });
    return { items };
  }

  function buildNormalizedComparison(chartsData) {
    const allSeries = chartsData.map(({ coinName, coinSymbol, chart }) => {
      const prices = chart?.prices || [];
      if (prices.length === 0) return { name: `${coinName} (${coinSymbol})`, data: [] };
      const firstPrice = prices[0][1];
      return {
        name: `${coinName} (${coinSymbol})`,
        data: prices.map((entry) => ({
          time: new Date(entry[0]),
          value: (entry[1] / firstPrice) * 100
        }))
      };
    });
    return { allSeries };
  }

  function renderLine(container, data) {
    if (!data?.allSeries?.length) return renderEmpty(container);
    renderSvg(container, (svg, width, height) => {
      const hasLegend = data.allSeries.length > 1;
      const margin = { top: 10, right: 10, bottom: hasLegend ? 40 : 20, left: 50 };
      const innerWidth = width - margin.left - margin.right;
      const innerHeight = height - margin.top - margin.bottom;
      const allData = data.allSeries.flatMap((s) => s.data);
      const x = d3
        .scaleTime()
        .domain(d3.extent(allData, (d) => d.time))
        .range([0, innerWidth]);
      const values = allData.map((d) => d.value);
      let minVal = d3.min(values);
      let maxVal = d3.max(values);
      if (minVal == null || maxVal == null) {
        minVal = 0;
        maxVal = 1;
      }
      if (minVal === maxVal) {
        const v = maxVal || 1;
        minVal = v - Math.abs(v) * 0.1 - 1;
        maxVal = v + Math.abs(v) * 0.1 + 1;
      } else {
        const pad = (maxVal - minVal) * 0.1;
        minVal -= pad;
        maxVal += pad;
      }
      const y = d3.scaleLinear().domain([minVal, maxVal]).nice().range([innerHeight, 0]);
      const g = svg
        .append("g")
        .attr("transform", `translate(${margin.left}, ${margin.top})`);

      data.allSeries.forEach((series, idx) => {
        g.append("path")
          .datum(series.data)
          .attr("fill", "none")
          .attr("stroke", CHART_COLORS[idx % CHART_COLORS.length])
          .attr("stroke-width", 2)
          .attr("d", d3.line().x((d) => x(d.time)).y((d) => y(d.value)));
      });

      g.append("g")
        .attr("transform", `translate(0, ${innerHeight})`)
        .call(d3.axisBottom(x).ticks(4));
      g.append("g").call(d3.axisLeft(y).ticks(4));

      if (hasLegend) {
        createLegend(svg, width, height, data.allSeries.map((s) => s.name), margin);
      }
    });
  }

  function renderDonut(container, data) {
    if (!data?.items?.length) return renderEmpty(container);
    renderSvg(container, (svg, width, height) => {
      const radius = Math.min(width, height) / 2 - 30;
      const group = svg
        .append("g")
        .attr("transform", `translate(${width / 2}, ${height / 2 - 10})`);
      const pieData = data.items.flatMap((item, idx) => [
        { label: `${item.label} ↑`, value: item.up, color: CHART_COLORS[(idx * 2) % CHART_COLORS.length] },
        { label: `${item.label} ↓`, value: item.down, color: CHART_COLORS[(idx * 2 + 1) % CHART_COLORS.length] }
      ]);
      const arc = d3.arc().innerRadius(radius * 0.6).outerRadius(radius);
      const pie = d3.pie().sort(null).value((d) => d.value);
      group
        .selectAll("path")
        .data(pie(pieData))
        .join("path")
        .attr("d", arc)
        .attr("fill", (d) => d.data.color);

      if (data.items.length > 0) {
        const legendGroup = svg.append("g").attr("transform", `translate(10, ${height - 40})`);
        data.items.forEach((item, i) => {
          const legendItem = legendGroup.append("g").attr("transform", `translate(0, ${i * 15})`);
          legendItem.append("rect").attr("width", 10).attr("height", 10).attr("fill", CHART_COLORS[i % CHART_COLORS.length]);
          legendItem
            .append("text")
            .attr("x", 14)
            .attr("y", 9)
            .attr("fill", "#e6eef7")
            .attr("font-size", 9)
            .text(`${item.label}: ↑${item.up} ↓${item.down}`);
        });
      }
    });
  }

  function renderDailyChange(container, data) {
    if (!data?.allSeries?.length) return renderEmpty(container);
    renderSvg(container, (svg, width, height) => {
      const hasLegend = data.allSeries.length > 1;
      const margin = { top: 10, right: 10, bottom: hasLegend ? 50 : 30, left: 50 };
      const innerWidth = width - margin.left - margin.right;
      const innerHeight = height - margin.top - margin.bottom;
      const allDates = [...new Set(data.allSeries.flatMap((s) => s.data.map((d) => d.date)))];
      const x0 = d3.scaleBand().domain(allDates).range([0, innerWidth]).padding(0.2);
      const x1 = d3.scaleBand().domain(data.allSeries.map((_, i) => i)).range([0, x0.bandwidth()]).padding(0.05);
      const allValues = data.allSeries.flatMap((s) => s.data.map((d) => d.value));
      const minVal = Math.min(...allValues);
      const maxVal = Math.max(...allValues);
      const y = d3.scaleLinear().domain([minVal, maxVal]).nice().range([innerHeight, 0]);
      const g = svg.append("g").attr("transform", `translate(${margin.left}, ${margin.top})`);
      allDates.forEach((date) => {
        const dateGroup = g.append("g").attr("transform", `translate(${x0(date)}, 0)`);
        data.allSeries.forEach((series, idx) => {
          const dataPoint = series.data.find((d) => d.date === date);
          if (dataPoint) {
            const isPositive = dataPoint.value >= 0;
            dateGroup
              .append("rect")
              .attr("x", x1(idx))
              .attr("y", isPositive ? y(dataPoint.value) : y(0))
              .attr("width", x1.bandwidth())
              .attr("height", Math.abs(y(dataPoint.value) - y(0)))
              .attr("fill", isPositive ? "#5eead4" : "#ff6b6b");
          }
        });
      });
      g.append("g")
        .attr("transform", `translate(0, ${y(0)})`)
        .call(d3.axisBottom(x0).tickValues(x0.domain().filter((_, i) => i % 2 === 0)));
      g.append("line").attr("x1", 0).attr("x2", innerWidth).attr("y1", y(0)).attr("y2", y(0)).attr("stroke", "#a9b8c7").attr("stroke-dasharray", "4");
      g.append("g").call(d3.axisLeft(y).ticks(4));
      if (hasLegend) createLegend(svg, width, height, data.allSeries.map((s) => s.name), margin);
    });
  }

  function renderHistogram(container, data) {
    if (!data?.items?.length) return renderEmpty(container);
    renderSvg(container, (svg, width, height) => {
      const item = data.items[0];
      const margin = { top: 10, right: 10, bottom: 50, left: 50 };
      const innerWidth = width - margin.left - margin.right;
      const innerHeight = height - margin.top - margin.bottom;
      const xDomain = item.data.map((d) => d.range);
      const x = d3.scaleBand().domain(xDomain).range([0, innerWidth]).padding(0.2);
      const y = d3.scaleLinear().domain([0, d3.max(item.data, (d) => d.count)]).nice().range([innerHeight, 0]);
      const g = svg.append("g").attr("transform", `translate(${margin.left}, ${margin.top})`);
      g.selectAll("rect")
        .data(item.data)
        .join("rect")
        .attr("x", (d) => x(d.range))
        .attr("y", (d) => y(d.count))
        .attr("width", x.bandwidth())
        .attr("height", (d) => innerHeight - y(d.count))
        .attr("fill", "#00c2a8");
      g.append("g")
        .attr("transform", `translate(0, ${innerHeight})`)
        .call(d3.axisBottom(x).tickValues(xDomain.filter((_, i) => i % 2 === 0)))
        .selectAll("text")
        .attr("font-size", "11px");
      g.append("g").call(d3.axisLeft(y).ticks(4));
      const statsText = svg.append("text").attr("x", 10).attr("y", 20).attr("fill", "#a9b8c7").attr("font-size", "11px").text(`Min: $${item.min} | Max: $${item.max}`);
    });
  }

  async function renderCard(card, forceFetch = false) {
    const body = card.element;
    if (!body) return;

    card.loading = true;
    card.error = null;
    updateCard(card);

    if (KPI_CONFIG[card.kpi]?.needsCoin && card.coinIds.length === 0) {
      card.loading = false;
      card.error = "Escolhe uma ou mais moedas";
      updateCard(card);
      return;
    }

    renderLoading(body);

    try {
      const chartsData = [];
      for (const coinId of card.coinIds) {
        const chart = await fetchMarketChart(coinId, card.period, forceFetch);
        const coinInfo = availableCoins.find((c) => c.id === coinId);
        chartsData.push({
          coinId,
          coinName: coinInfo ? coinInfo.name : coinId,
          coinSymbol: coinInfo ? coinInfo.symbol.toUpperCase() : coinId.toUpperCase(),
          chart
        });
        if (card.coinIds.indexOf(coinId) < card.coinIds.length - 1) {
          await delay(REQUEST_SPACING_MS);
        }
      }

      const config = KPI_CONFIG[card.kpi];
      const data = config.build(chartsData);
      body.innerHTML = "";
      const chartContainer = document.createElement("div");
      chartContainer.className = "chart-svg-container";
      body.appendChild(chartContainer);
      config.render(chartContainer, data, chartsData);

      if (card.kpi === "price-trend" && chartsData.length > 0) {
        chartsData.forEach(({ coinName, coinSymbol, chart }) => {
          const prices = (chart?.prices || []).map((p) => p[1]);
          if (prices.length > 0) {
            const stats = calculateStats(prices.map((v) => ({ value: v })));
            if (stats) {
              renderStats(body, stats, `${coinName} (${coinSymbol})`);
            }
          }
        });
      }

      card.loading = false;
      card.error = null;
      updateCard(card);
    } catch (error) {
      console.error(error);
      const isRateLimit = error instanceof ApiError && error.status === 429;
      renderEmpty(body, isRateLimit ? "Limite da API atingido" : "Erro ao obter dados", true);
      const retry = document.createElement("button");
      retry.className = "retry-btn";
      retry.textContent = "Tentar de novo";
      retry.addEventListener("click", () => renderCard(card, forceFetch));
      body.querySelector(".chart-empty")?.appendChild(retry);
      card.loading = false;
      card.error = isRateLimit ? "Limite da API atingido" : "Erro ao obter dados";
      updateCard(card);
    }
  }

  async function renderAllCharts(forceFetch = false) {
    if (dashboard.length === 0) return;
    if (forceFetch) {
      marketsLoaded = false;
      chartCache.clear();
    }

    try {
      await fetchMarkets();
    } catch (error) {
      console.error(error);
      showToast("Não foi possível carregar a lista de moedas.", "error");
    }

    for (const card of dashboard) {
      await renderCard(card, forceFetch);
      if (dashboard.indexOf(card) < dashboard.length - 1) {
        await delay(REQUEST_SPACING_MS);
      }
    }
  }

  async function addChart(kpiId, coinIds = [], period = 30, render = true, insertIndex = null) {
    const card = {
      id: createCardId(),
      kpi: kpiId,
      coinIds: coinIds || [],
      period,
      loading: false,
      error: null,
      element: null
    };
    if (insertIndex === null || insertIndex >= dashboard.length) {
      dashboard = [...dashboard, card];
    } else if (insertIndex <= 0) {
      dashboard = [card, ...dashboard];
    } else {
      dashboard = [...dashboard.slice(0, insertIndex), card, ...dashboard.slice(insertIndex)];
    }
    if (render) {
      await tick();
      renderCard(card);
    }
    saveState();
    return card;
  }

  function reorderDashboardFromGrid() {
    if (!dashboardGrid) return;
    const newOrderIds = Array.from(dashboardGrid.querySelectorAll(".chart-card")).map(
      (card) => card.dataset.id
    );
    dashboard = newOrderIds
      .map((id) => dashboard.find((card) => card.id === id))
      .filter(Boolean);
    saveState();
  }

  function handleDocumentClick(event) {
    const target = event.target;
    if (target.closest(".coin-selector-container")) return;
    openCoinDropdown = null;
  }

  function toggleDropdown(cardId) {
    openCoinDropdown = openCoinDropdown === cardId ? null : cardId;
  }

  function toggleCoinSelection(card, coinId, checked) {
    const updated = { ...card };
    if (checked) {
      updated.coinIds = [...new Set([...updated.coinIds, coinId])];
    } else {
      updated.coinIds = updated.coinIds.filter((id) => id !== coinId);
    }
    updateCard(updated);
    saveState();
    renderCard(updated);
  }

  function changePeriod(card, period) {
    const updated = { ...card, period };
    updateCard(updated);
    saveState();
    renderCard(updated);
  }

  function markUpdated() {
    lastUpdated = `Atualizado às ${formatTime(new Date())}`;
  }

  async function handleRefresh() {
    if (dashboard.length === 0) {
      showToast("Adiciona um gráfico antes de atualizar.", "warn");
      return;
    }
    loadingRefresh = true;
    try {
      await renderAllCharts(true);
      markUpdated();
      showToast("Dados atualizados a partir da API.", "success");
    } catch (error) {
      console.error(error);
      showToast("Não foi possível atualizar os dados.", "error");
    } finally {
      loadingRefresh = false;
    }
  }

  function setupDragAndDrop() {
    if (!kpiList || !dashboardGrid) {
      return;
    }

    Sortable.create(kpiList, {
      group: { name: "kpi", pull: "clone", put: false },
      sort: false,
      draggable: ".kpi-card",
      animation: 150,
      fallbackOnBody: true,
      delay: 100,
      delayOnTouchOnly: true
    });

    Sortable.create(dashboardGrid, {
      group: { name: "kpi", pull: true, put: true },
      draggable: ".chart-card",
      animation: 150,
      fallbackOnBody: true,
      accept: function(sortable, el) { return el.classList.contains("kpi-card"); },
      onAdd: async (event) => {
        const kpiId = event.clone?.getAttribute("data-kpi") || event.item?.getAttribute("data-kpi");
        const insertIndex = typeof event.newIndex === "number" ? event.newIndex : null;
        if (event.item) event.item.remove();
        if (kpiId && kpiId in KPI_CONFIG) {
          await addChart(kpiId, [], 30, true, insertIndex);
        }
      },
      onEnd: () => reorderDashboardFromGrid()
    });
  }

  onMount(async () => {
    document.addEventListener("click", handleDocumentClick);

    try {
      await fetchMarkets();
    } catch (error) {
      console.error(error);
      showToast("Não foi possível ligar à API. Tenta atualizar.", "error");
    }

    const saved = loadState();
    for (const savedCard of saved) {
      await addChart(savedCard.kpi, savedCard.coinIds, savedCard.period || 30, false);
    }

    if (dashboard.length > 0) {
      await tick();
      await renderAllCharts(false);
    }

    // Setup drag-and-drop AFTER all DOM is ready
    await tick();
    setupDragAndDrop();
  });

  onDestroy(() => {
    document.removeEventListener("click", handleDocumentClick);
  });
</script>

<div class="app-shell">
  <header class="app-header">
    <h1>Painel de Ações</h1>
    <p>
      Arrasta os KPIs da esquerda para o dashboard para criar gráficos.
      Usa <strong>Atualizar</strong> para recarregar os dados da API.
    </p>
  </header>

  <div class="app-body">
    <aside class="sidebar">
      <div class="panel-title">KPIs</div>
      <div class="kpi-list" bind:this={kpiList}>
        {#each Object.entries(KPI_CONFIG) as [kpiId, item]}
          <div
            class="kpi-card {getKpiActive(kpiId) ? "kpi-active" : ""}"
            data-kpi={kpiId}
          >
            {item.title}
          </div>
        {/each}
      </div>

      <div class="panel-actions">
        <button class="btn" on:click={() => { saveState(); showToast("Dashboard guardado.", "success"); }}>
          Guardar
        </button>
        <button class="btn btn-secondary" class:is-loading={loadingRefresh} on:click={handleRefresh} disabled={loadingRefresh}>
          <span class="btn-label">{loadingRefresh ? "A atualizar..." : "Atualizar"}</span>
          <span class="btn-spinner" aria-hidden="true" class:visible={loadingRefresh}></span>
        </button>
        <p class="last-updated">{lastUpdated}</p>
      </div>
    </aside>

    <main class="dashboard">
      <div class="panel-title">Dashboard</div>
      <div class="dashboard-grid dashboard-dropzone" bind:this={dashboardGrid}>
        {#if dashboard.length === 0}
          <div class="dashboard-hint">
            Arrasta um KPI da esquerda para começar.
          </div>
        {/if}

        {#each dashboard as card (card.id)}
          <div class="chart-card" data-id={card.id}>
            <div class="chart-header">
              <div class="chart-title-row">
                <div class="chart-title">{KPI_CONFIG[card.kpi].title}</div>
                <button class="chart-remove" on:click={() => removeChart(card.id)} title="Remover">
                  ×
                </button>
              </div>

              <div class="chart-selector-row">
                {#if KPI_CONFIG[card.kpi].allowMultiCoin}
                  <div class="coin-selector-container">
                    <button class="coin-dropdown-button" type="button" on:click={() => toggleDropdown(card.id)}>
                      <span>
                        {card.coinIds.length > 0
                          ? `${card.coinIds.length} moeda${card.coinIds.length > 1 ? "s" : ""} selecionada${
                              card.coinIds.length > 1 ? "s" : ""
                            }`
                          : "Selecionar moedas..."}
                      </span>
                      <span class="dropdown-arrow">▼</span>
                    </button>
                    <div class="coin-dropdown-menu" class:show={openCoinDropdown === card.id}>
                      {#each availableCoins as coin}
                        <label class="coin-dropdown-item">
                          <input
                            type="checkbox"
                            checked={card.coinIds.includes(coin.id)}
                          on:change={(event) => toggleCoinSelection(card, coin.id, event.target.checked)}
                        />
                        <span>{coin.name} ({coin.symbol.toUpperCase()})</span>
                      </label>
                    {/each}
                  </div>
                </div>
                {:else}
                  <div class="coin-selector-container">
                    <button class="coin-dropdown-button" type="button" on:click={() => toggleDropdown(card.id)}>
                      <span>
                        {card.coinIds.length > 0
                          ? availableCoins.find((c) => c.id === card.coinIds[0])?.name || "Selecionar"
                          : "Selecionar..."}
                      </span>
                      <span class="dropdown-arrow">▼</span>
                    </button>
                    <div class="coin-dropdown-menu" class:show={openCoinDropdown === card.id}>
                      {#each availableCoins as coin}
                        <button
                          class="coin-dropdown-item"
                          type="button"
                          on:click={() => {
                            card.coinIds = [coin.id];
                            updateCard(card);
                            renderCard(card);
                            openCoinDropdown = null;
                          }}
                        >
                          {coin.name} ({coin.symbol.toUpperCase()})
                        </button>
                      {/each}
                    </div>
                  </div>
                {/if}

                <div class="period-selector-container">
                  <span class="period-label">Período:</span>
                  <select class="period-select" bind:value={card.period} on:change={(event) => changePeriod(card, +event.target.value)}>
                    {#each periods as periodOption}
                      <option value={periodOption.value}>{periodOption.label}</option>
                    {/each}
                  </select>
                </div>
              </div>
            </div>

            <div class="chart-body" bind:this={card.element}>
              {#if card.loading}
                <div class="chart-loading">
                  <div class="spinner"></div>
                  <span>A carregar...</span>
                </div>
              {:else if card.error}
                <div class="chart-empty is-error">{card.error}</div>
              {/if}
            </div>
          </div>
        {/each}
      </div>
    </main>
  </div>
</div>

<div id="toast-container" class="toast-container">
  {#each toasts as toast (toast.id)}
    <button type="button" class="toast toast-{toast.type}" on:click={() => removeToast(toast.id)}>
      <span class="toast-icon">
        {toast.type === "success" ? "✓" : toast.type === "warn" ? "⚠" : "✕"}
      </span>
      <span class="toast-text">{toast.message}</span>
    </button>
  {/each}
</div>