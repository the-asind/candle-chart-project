<template>
  <div class="container">
    <h1>Свечной график акций Мосбиржи</h1>
    <form @submit.prevent="updateChart" class="form">
      <div class="input-group">
        <label for="ticker">Поиск эмитента:</label>
        <input
          v-model="searchQuery"
          @input="searchEmitent"
          id="ticker"
          autocomplete="off"
          required
          class="input"
        />
        <ul v-if="searchResults.length" class="dropdown">
          <li
            v-for="result in searchResults"
            :key="result.SECID"
            @click="selectEmitent(result.SECID)"
            class="dropdown-item"
          >
            {{ result.SECID }} - {{ result.NAME }}
          </li>
        </ul>
      </div>

      <div class="input-group">
        <label for="startDay">Дата начала:</label>
        <input v-model="startDay" id="startDay" type="date" required class="input" />
      </div>

      <div class="input-group">
        <label for="endDay">Дата окончания (включительно):</label>
        <input v-model="endDay" id="endDay" type="date" required class="input" />
      </div>

      <div class="input-group">
        <label>Интервал:</label>
        <div class="interval-buttons">
          <button
            v-for="int in intervals"
            :key="int.value"
            @click="selectInterval(int.value)"
            :class="['button', { active: interval === int.value }]"
            type="button"
          >
            {{ int.label }}
          </button>
        </div>
      </div>

      <button type="submit" class="button primary">Обновить график!</button>
    </form>

    <div ref="chart" class="chart-container"></div>

    <div v-if="errorMessage" class="alert error">{{ errorMessage }}</div>
    <div v-if="emitentTitle" class="emitent">{{ emitentTitle }}</div>
    <div v-if="warningMessage" class="alert warning">{{ warningMessage }}</div>
  </div>
</template>

<script>
import { createChart } from 'lightweight-charts';

export default {
  name: 'CandleChart',
  data() {
    return {
      chart: null,
      candleSeries: null,
      ticker: 'MGNT',
      searchQuery: 'MGNT',
      searchResults: [],
      startDay: '2024-05-20',
      endDay: '2024-05-24',
      interval: '1',
      errorMessage: '',
      warningMessage: '',
      emitentTitle: 'Публичное акционерное общество "Магнит"',
      intervals: [
        { label: 'День', value: '24' },
        { label: 'Час', value: '1' },
        { label: 'Неделя', value: '7' },
        { label: 'Месяц', value: '31' },
      ]
    };
  },
  mounted() {
    this.chart = createChart(this.$refs.chart, { width: 600, height: 400 });
    this.candleSeries = this.chart.addCandlestickSeries();
    this.fetchData();
  },
  methods: {
    async fetchData() {
      const apiUrl = `https://iss.moex.com/iss/engines/stock/markets/shares/securities/${this.ticker}/candles.json?from=${this.startDay}&till=${this.endDay}&interval=${this.interval}`;

      try {
        const response = await fetch(apiUrl);
        const data = await response.json();

        if (data.candles.data.length === 0) {
          throw new Error('No data available for the provided ticker and date range.');
        }

        const candles = data.candles.data.map(item => ({
          time: new Date(item[6]).getTime() / 1000, // Unix time in seconds
          open: item[0],
          high: item[2],
          low: item[3],
          close: item[1]
        })).sort((a, b) => a.time - b.time); // Ensure data is sorted by time

        this.candleSeries.setData(candles);
        this.errorMessage = '';
        this.checkCandleLimit(candles.length);
      } catch (error) {
        this.errorMessage = `Error: ${error.message}`;
        this.candleSeries.setData([]);
      }
    },
    async searchEmitent() {
      const url = 'http://iss.moex.com/iss/statistics/engines/stock/quotedsecurities.json';

      try {
        const response = await fetch(url);
        const data = await response.json();
        const results = data.quotedsecurities.data.filter(security =>
          security[1].toLowerCase().includes(this.searchQuery.toLowerCase()) ||
          security[2].toLowerCase().includes(this.searchQuery.toLowerCase())
        );

        this.searchResults = results.map(result => ({
          SECID: result[1],
          NAME: result[2]
        }));
      } catch (error) {
        this.errorMessage = `Ошибка: ${error.message}`;
      }
    },
    selectEmitent(secid) {
      this.ticker = secid;
      this.searchQuery = secid;
      this.searchResults = [];
    },
    async checkTicker() {
      const url = `https://iss.moex.com/iss/securities.json?q=${this.ticker}&is_trading=1`;

      try {
        const response = await fetch(url);
        const data = await response.json();
        const securities = data.securities.data;

        if (securities.length === 0) {
          throw new Error('Недействительный или неторгуемый эмитент.');
        }

        const emitent = securities.find(item => item[8] !== null);
        if (emitent) {
          this.emitentTitle = emitent[8];
        } else {
          throw new Error('Эмитент не найден.');
        }

        return true;
      } catch (error) {
        this.errorMessage = `Ошибка: ${error.message}`;
        this.emitentTitle = '';
        return false;
      }
    },
    async updateChart() {
      const start = new Date(this.startDay);
      const end = new Date(this.endDay);

      if (isNaN(start) || isNaN(end)) {
        this.errorMessage = 'Неправильный формат даты.';
        return;
      }

      if (start >= end) {
        this.errorMessage = 'Дата начала должна быть раньше даты окончания.';
        return;
      }

      const isValidTicker = await this.checkTicker();
      if (isValidTicker) {
        this.fetchData();
      }
    },
    checkCandleLimit(candleCount) {
      const maxCandles = 720;
      if (candleCount > maxCandles) {
        this.warningMessage = `Warning: The maximum number of candles is ${maxCandles}. Your request has ${candleCount} candles. Please adjust the date range or interval.`;
      } else {
        this.warningMessage = '';
      }
    },
    selectInterval(value) {
      this.interval = value;
    }
  }
};
</script>

<style>
.container {
  max-width: 800px;
  margin: auto;
  padding: 20px;
  font-family: 'Roboto', sans-serif;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 20px;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.input {
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.button {
  padding: 10px 20px;
  font-size: 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.button.primary {
  background-color: #6200ea;
  color: white;
}

.button.primary:hover {
  background-color: #3700b3;
}

.interval-buttons {
  display: flex;
  gap: 10px;
}

.button.active {
  background-color: #6200ea;
  color: white;
}

.dropdown {
  border: 1px solid #ccc;
  max-height: 150px;
  overflow-y: auto;
  list-style: none;
  margin: 0;
  padding: 0;
}

.dropdown-item {
  padding: 8px;
  cursor: pointer;
}

.dropdown-item:hover {
  background-color: #f0f0f0;
}

.chart-container {
  position: relative;
  margin-top: 20px;
}

.alert {
  padding: 15px;
  margin-top: 10px;
  border-radius: 4px;
  font-size: 16px;
}

.error {
  background-color: #f44336;
  color: white;
}

.warning {
  background-color: #ff9800;
  color: white;
}

.emitent {
  margin-top: 10px;
  font-weight: bold;
  font-size: 16px;
}
</style>
