<template>
  <main class="app-shell" v-if="initialized">
    <header>
      <h1>FX Position Size Calculator</h1>
      <p class="subtitle">Babypips-style workflow powered by live ask prices and smart search.</p>
    </header>
    <div class="grid-two-col">
      <section class="panel">
        <h2>Trade Setup</h2>
        <div class="pair-search-group">
          <label>
            Search Currency Pair
            <input
              type="text"
              v-model="pairSearch"
              placeholder="Try typing usdgp or eurjpy"
              autocomplete="off"
            />
          </label>
          <p v-if="searchMessage" class="search-message">{{ searchMessage }}</p>
        </div>
        <div class="form-grid">
          <label>
            Currency Pair
            <select v-model="form.pair" :disabled="!filteredPairs.length">
              <option v-for="pair in filteredPairs" :key="pair.symbol" :value="pair.symbol">
                {{ pair.symbol }}
              </option>
            </select>
          </label>
          <label>
            Account Currency
            <select v-model="form.accountCurrency">
              <option v-for="currency in accountCurrencies" :key="currency" :value="currency">
                {{ currency }}
              </option>
            </select>
          </label>
          <label>
            Account Balance
            <input type="number" step="0.01" min="0" v-model="form.balance" />
          </label>
          <label>
            Risk % Per Trade
            <input type="number" step="0.1" min="0" max="100" v-model="form.riskPercent" />
          </label>
          <label>
            Take Profit (TP)
            <input type="number" step="0.0001" v-model="form.tp" />
          </label>
          <label>
            Entry
            <input type="number" step="0.0001" v-model="form.entry" />
          </label>
          <label>
            Stop Loss (SL)
            <input type="number" step="0.0001" v-model="form.sl" />
          </label>
          <label>
            Contract Size (units per lot)
            <input type="number" step="1" min="1" v-model="form.contractSize" />
          </label>
        </div>
        <div class="info-banner">
          Rates provided by AwesomeAPI (awesomeapi.com.br) and cached for a few minutes. Refresh before you size the trade.
        </div>
        <div class="stat-cards">
          <div class="stat-card">
            <small>Pip Precision</small>
            <strong>{{ pipPrecisionLabel }}</strong>
          </div>
          <div class="stat-card">
            <small>Ask Price</small>
            <strong>{{ askPriceDisplay }}</strong>
          </div>
          <div class="stat-card">
            <small>Last Update</small>
            <strong>{{ askUpdatedDisplay }}</strong>
          </div>
        </div>
        <div class="ask-panel">
          <div class="ask-panel__values">
            <span class="ask-panel__label">Current Ask</span>
            <span class="ask-panel__value">{{ askPriceDisplay }}</span>
            <span class="ask-panel__timestamp">{{ askUpdatedDisplay || "Awaiting refresh" }}</span>
          </div>
          <button class="ghost-button" type="button" @click="refreshAskPrice" :disabled="askLoading || !form.pair">
            {{ askLoading ? "Fetching Price..." : "Refresh Price" }}
          </button>
        </div>
        <button class="primary-button" type="button" @click="handleSubmit" :disabled="!filteredPairs.length || askLoading">
          Calculate Position Size
        </button>
        <p v-if="error" class="error-message">{{ error }}</p>
      </section>
      <section class="panel">
        <h2>Results</h2>
        <div v-if="results" class="results-grid">
          <div class="result-row">
            <span>Reward (pips)</span>
            <span>{{ results.rewardPips }}</span>
          </div>
          <div class="result-row">
            <span>Risk (pips)</span>
            <span>{{ results.riskPips }}</span>
          </div>
          <div class="result-row">
            <span>R (Reward / Risk)</span>
            <span>{{ results.ratio }}</span>
          </div>
          <div class="result-row">
            <span>Risk Capital</span>
            <span>{{ results.riskCapital }}</span>
          </div>
          <div class="result-row">
            <span>Pip Value per Lot</span>
            <span>{{ results.pipValuePerLot }}</span>
          </div>
          <div class="result-row">
            <span>Position Size (Lots)</span>
            <span>{{ results.lotSize }}</span>
          </div>
          <div class="result-row">
            <span>Position Size (Units)</span>
            <span>{{ results.units }}</span>
          </div>
        </div>
        <p v-else class="subtitle">
          Fill out the trade details, fetch the latest ask price, and your position sizing breakdown will appear here.
        </p>
      </section>
    </div>
    <footer>Inputs persist locally so you can iterate on setups faster.</footer>
  </main>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      defaultPairs: [
        { symbol: "EUR/USD", base: "EUR", quote: "USD", digits: 4 },
        { symbol: "GBP/USD", base: "GBP", quote: "USD", digits: 4 },
        { symbol: "AUD/USD", base: "AUD", quote: "USD", digits: 4 },
        { symbol: "NZD/USD", base: "NZD", quote: "USD", digits: 4 },
        { symbol: "USD/JPY", base: "USD", quote: "JPY", digits: 2 },
        { symbol: "USD/CHF", base: "USD", quote: "CHF", digits: 4 },
        { symbol: "USD/CAD", base: "USD", quote: "CAD", digits: 4 },
        { symbol: "EUR/GBP", base: "EUR", quote: "GBP", digits: 4 },
        { symbol: "EUR/JPY", base: "EUR", quote: "JPY", digits: 2 },
        { symbol: "EUR/CHF", base: "EUR", quote: "CHF", digits: 4 },
        { symbol: "EUR/AUD", base: "EUR", quote: "AUD", digits: 4 },
        { symbol: "EUR/CAD", base: "EUR", quote: "CAD", digits: 4 },
        { symbol: "EUR/NZD", base: "EUR", quote: "NZD", digits: 4 },
        { symbol: "GBP/JPY", base: "GBP", quote: "JPY", digits: 2 },
        { symbol: "GBP/CHF", base: "GBP", quote: "CHF", digits: 4 },
        { symbol: "GBP/AUD", base: "GBP", quote: "AUD", digits: 4 },
        { symbol: "GBP/CAD", base: "GBP", quote: "CAD", digits: 4 },
        { symbol: "GBP/NZD", base: "GBP", quote: "NZD", digits: 4 },
        { symbol: "AUD/JPY", base: "AUD", quote: "JPY", digits: 2 },
        { symbol: "AUD/NZD", base: "AUD", quote: "NZD", digits: 4 },
        { symbol: "AUD/CAD", base: "AUD", quote: "CAD", digits: 4 },
        { symbol: "AUD/CHF", base: "AUD", quote: "CHF", digits: 4 },
        { symbol: "NZD/JPY", base: "NZD", quote: "JPY", digits: 2 },
        { symbol: "NZD/CAD", base: "NZD", quote: "CAD", digits: 4 },
        { symbol: "NZD/CHF", base: "NZD", quote: "CHF", digits: 4 },
        { symbol: "CAD/JPY", base: "CAD", quote: "JPY", digits: 2 },
        { symbol: "CAD/CHF", base: "CAD", quote: "CHF", digits: 4 },
        { symbol: "CHF/JPY", base: "CHF", quote: "JPY", digits: 2 }
      ],
      currencyPairs: [],
      accountCurrencies: ["USD", "EUR", "GBP", "JPY", "AUD", "NZD", "CAD", "CHF"],
      pairSearch: "",
      form: {
        pair: "",
        accountCurrency: "USD",
        balance: "",
        riskPercent: "",
        tp: "",
        entry: "",
        sl: "",
        contractSize: "100000"
      },
      pipDigits: "-",
      askPrice: null,
      askUpdated: "",
      askLoading: false,
      error: "",
      results: null,
      storageKey: "pipCalculatorForm",
      rateCache: new Map(),
      cacheDuration: 3 * 60 * 1000,
      lastPricePair: null,
      lastAskTimestamp: null,
      initialized: false
    };
  },
  computed: {
    filteredPairs() {
      const query = this.pairSearch.trim();
      if (!query) return this.currencyPairs;
      return this.currencyPairs.filter((pair) => this.isFuzzyMatch(query, pair.symbol));
    },
    searchMessage() {
      if (!this.pairSearch.trim()) return "";
      if (!this.filteredPairs.length) {
        return `No pairs match "${this.pairSearch}".`;
      }
      return "";
    },
    pipPrecisionLabel() {
      return this.pipDigits === "-" ? "-" : `${this.pipDigits} decimals`;
    },
    askPriceDisplay() {
      if (!Number.isFinite(this.askPrice)) return "--";
      return Number(this.askPrice).toFixed(this.pipDigits === 2 ? 3 : 5);
    },
    askUpdatedDisplay() {
      if (!this.askUpdated) return "";
      return new Date(this.askUpdated).toLocaleTimeString();
    }
  },
  watch: {
    filteredPairs: {
      handler(newPairs) {
        if (!newPairs.length) {
          this.form.pair = "";
          return;
        }
        if (!newPairs.some((pair) => pair.symbol === this.form.pair)) {
          this.form.pair = newPairs[0].symbol;
        }
      },
      immediate: true
    },
    "form.pair"(symbol) {
      this.setDigitsFromPair(symbol);
      this.results = null;
    },
    form: {
      handler() {
        this.persistState();
      },
      deep: true
    },
    pairSearch() {
      this.persistState();
    }
  },
  async mounted() {
    await this.initializeApp();
  },
  methods: {
    async initializeApp() {
      await this.loadCurrencyPairs();
      this.restoreState();
      if (!this.currencyPairs.length) {
        this.currencyPairs = [...this.defaultPairs];
      }
      if (!this.form.pair && this.currencyPairs.length) {
        this.form.pair = this.currencyPairs[0].symbol;
      }
      if (!this.accountCurrencies.includes(this.form.accountCurrency)) {
        this.form.accountCurrency = this.accountCurrencies[0];
      }
      this.setDigitsFromPair(this.form.pair);
      await this.refreshAskPrice({ silent: true });
      this.initialized = true;
    },
    getPairMeta(symbol) {
      return this.currencyPairs.find((pair) => pair.symbol === symbol);
    },
    normalizeSymbol(text) {
      return text.replace(/[^A-Z]/gi, "").toUpperCase();
    },
    isFuzzyMatch(query, symbol) {
      const cleanQuery = this.normalizeSymbol(query);
      if (!cleanQuery) return true;
      const target = this.normalizeSymbol(symbol);
      let index = 0;
      for (const char of cleanQuery) {
        index = target.indexOf(char, index);
        if (index === -1) return false;
        index += 1;
      }
      return true;
    },
    setDigitsFromPair(symbol) {
      const pair = this.getPairMeta(symbol);
      if (!pair) {
        this.pipDigits = "-";
        this.askPrice = null;
        this.askUpdated = "";
        return;
      }
      this.pipDigits = pair.digits;
      this.askPrice = null;
      this.askUpdated = "";
      this.lastPricePair = null;
    },
    formatValue(value, digits = 2) {
      return Number(value).toFixed(digits);
    },
    formatCurrency(value, currency) {
      if (!Number.isFinite(value)) return "-";
      try {
        return new Intl.NumberFormat("en-US", {
          style: "currency",
          currency,
          maximumFractionDigits: 2
        }).format(value);
      } catch {
        return `${currency} ${this.formatValue(value, 2)}`;
      }
    },
    formatInteger(value) {
      if (!Number.isFinite(value)) return "-";
      return new Intl.NumberFormat("en-US", { maximumFractionDigits: 0 }).format(value);
    },
    showError(message) {
      this.error = message;
    },
    clearError() {
      this.error = "";
    },
    persistState() {
      try {
        const payload = {
          form: this.form,
          pairSearch: this.pairSearch
        };
        localStorage.setItem(this.storageKey, JSON.stringify(payload));
      } catch (error) {
        console.warn("Unable to persist state", error);
      }
    },
    restoreState() {
      let saved = null;
      try {
        saved = localStorage.getItem(this.storageKey);
      } catch {
        saved = null;
      }
      if (!saved) return;
      try {
        const parsed = JSON.parse(saved);
        if (parsed.form) {
          Object.assign(this.form, parsed.form);
        }
        if (parsed.pairSearch) {
          this.pairSearch = parsed.pairSearch;
        }
      } catch (error) {
        console.warn("Failed to restore state", error);
      }
    },
    async loadCurrencyPairs() {
      try {
        const response = await fetch("currency_pairs.json", { cache: "no-store" });
        if (!response.ok) throw new Error("Failed to load currency pairs");
        const data = await response.json();
        if (Array.isArray(data) && data.length) {
          this.currencyPairs = data;
          return;
        }
        this.currencyPairs = [...this.defaultPairs];
      } catch (error) {
        console.warn("Falling back to default pairs", error);
        this.currencyPairs = [...this.defaultPairs];
      }
    },
    async requestQuote(base, quote) {
      const formatted = `${base}-${quote}`;
      const response = await fetch(`https://economia.awesomeapi.com.br/last/${formatted}`);
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      const data = await response.json();
      const record = data[`${base}${quote}`];
      if (!record) {
        throw new Error("Pair not supported by price feed");
      }
      const ask = Number(record.ask);
      const bid = Number(record.bid);
      const timestamp = Number(record.timestamp) * 1000 || Date.now();
      if (!Number.isFinite(ask)) {
        throw new Error("Missing ask price");
      }
      return { ask, bid: Number.isFinite(bid) ? bid : null, timestamp };
    },
    getCacheKey(from, to) {
      return `${from}_${to}`;
    },
    getCachedRate(from, to) {
      const key = this.getCacheKey(from, to);
      const cached = this.rateCache.get(key);
      if (cached && Date.now() - cached.timestamp < this.cacheDuration) {
        return cached.value;
      }
      return null;
    },
    storeRate(from, to, value) {
      const key = this.getCacheKey(from, to);
      this.rateCache.set(key, { value, timestamp: Date.now() });
    },
    async fetchConversionRate(from, to) {
      if (from === to) return 1;
      const cached = this.getCachedRate(from, to);
      if (cached) return cached;
      const quote = await this.requestQuote(from, to);
      this.storeRate(from, to, quote.ask);
      return quote.ask;
    },
    async refreshAskPrice({ silent = false } = {}) {
      const pair = this.getPairMeta(this.form.pair);
      if (!pair) {
        this.showError("Please select a supported currency pair.");
        return null;
      }
      if (!silent) {
        this.askLoading = true;
      }
      try {
        const quote = await this.requestQuote(pair.base, pair.quote);
        this.askPrice = quote.ask;
        this.askUpdated = quote.timestamp;
        this.lastPricePair = pair.symbol;
        this.lastAskTimestamp = quote.timestamp;
        this.clearError();
        return quote.ask;
      } catch (error) {
        if (!silent) {
          this.showError(`Price fetch failed: ${error.message}`);
        }
        return null;
      } finally {
        if (!silent) {
          this.askLoading = false;
        }
      }
    },
    calculateRewardRisk(tp, entry, sl) {
      return {
        reward: Math.abs(tp - entry),
        risk: Math.abs(entry - sl)
      };
    },
    async getPipValuePerLot(pair, askPrice, accountCurrency, contractSize) {
      const pipSize = pair.digits === 2 ? 0.01 : 0.0001;
      const pipValueQuote = pipSize * contractSize;
      if (accountCurrency === pair.quote) {
        return pipValueQuote;
      }
      if (accountCurrency === pair.base) {
        return pipValueQuote / askPrice;
      }
      const conversion = await this.fetchConversionRate(pair.quote, accountCurrency);
      if (!Number.isFinite(conversion)) {
        throw new Error("Unable to convert pip value to account currency.");
      }
      return pipValueQuote * conversion;
    },
    async handleSubmit() {
      this.clearError();
      const tp = Number(this.form.tp);
      const entry = Number(this.form.entry);
      const sl = Number(this.form.sl);
      const balance = Number(this.form.balance);
      const riskPercent = Number(this.form.riskPercent);
      const contractSize = Number(this.form.contractSize);
      const accountCurrency = this.form.accountCurrency;
      const pair = this.getPairMeta(this.form.pair);

      if ([tp, entry, sl, balance, riskPercent, contractSize].some((value) => Number.isNaN(value))) {
        this.showError("Please enter valid numeric values for every field.");
        return;
      }
      if (!pair) {
        this.showError("Selected currency pair is not supported.");
        return;
      }
      if (balance <= 0) {
        this.showError("Account balance must be greater than 0.");
        return;
      }
      if (riskPercent <= 0 || riskPercent > 100) {
        this.showError("Risk percent must be between 0 and 100.");
        return;
      }
      if (contractSize <= 0) {
        this.showError("Contract size must be greater than 0.");
        return;
      }

      let askPrice = this.askPrice;
      const isStale = !this.lastAskTimestamp || Date.now() - this.lastAskTimestamp > this.cacheDuration;
      if (!Number.isFinite(askPrice) || this.lastPricePair !== pair.symbol || isStale) {
        askPrice = await this.refreshAskPrice({ silent: true });
      }
      if (!Number.isFinite(askPrice)) {
        this.showError("Fetch the latest ask price before calculating.");
        return;
      }

      const { reward, risk } = this.calculateRewardRisk(tp, entry, sl);
      if (risk === 0) {
        this.showError("Entry and SL must not match.");
        return;
      }

      const multiplier = pair.digits === 2 ? 100 : 10000;
      const rewardPips = reward * multiplier;
      const riskPips = risk * multiplier;
      const riskCapital = (balance * riskPercent) / 100;

      let pipValuePerLot;
      try {
        pipValuePerLot = await this.getPipValuePerLot(pair, askPrice, accountCurrency, contractSize);
      } catch (error) {
        this.showError(error.message);
        return;
      }

      const riskPerLot = riskPips * pipValuePerLot;
      if (!Number.isFinite(riskPerLot) || riskPerLot <= 0) {
        this.showError("Risk per lot could not be calculated.");
        return;
      }

      const lotSize = riskCapital / riskPerLot;
      const units = lotSize * contractSize;
      const ratio = reward / risk;

      this.results = {
        rewardPips: this.formatValue(rewardPips, 2),
        riskPips: this.formatValue(riskPips, 2),
        ratio: Number.isFinite(ratio) ? this.formatValue(ratio, 2) : "∞",
        riskCapital: this.formatCurrency(riskCapital, accountCurrency),
        pipValuePerLot: this.formatCurrency(pipValuePerLot, accountCurrency),
        lotSize: this.formatValue(lotSize, 3),
        units: this.formatInteger(units)
      };
    }
  }
};
</script>
