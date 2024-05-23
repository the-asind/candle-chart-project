<template>
    <div>
      <h1>Stock Candle Chart Widget</h1>
      <form @submit.prevent="updateChart">
        <label for="ticker">Ticker:</label>
        <input v-model="ticker" id="ticker" required />
        
        <label for="startDay">Start Date:</label>
        <input v-model="startDay" id="startDay" type="date" required />
        
        <label for="endDay">End Date:</label>
        <input v-model="endDay" id="endDay" type="date" required />
        
        <label for="interval">Interval:</label>
        <select v-model="interval" id="interval" required>
          <option value="24">Day</option>
          <option value="60">Hour</option>
          <option value="7">Week</option>
          <option value="31">Month</option>
        </select>
        
        <button type="submit">Update Chart</button>
      </form>
      <div ref="chart" class="chart-container"></div>
      <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
      <div v-if="emitentTitle" class="emitent">{{ emitentTitle }}</div>
      <div v-if="warningMessage" class="warning">{{ warningMessage }}</div>
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
        ticker: 'YNDX',
        startDay: '2024-02-01',
        endDay: '2024-02-07',
        interval: '24',
        errorMessage: '',
        warningMessage: '',
        emitentTitle: ''
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
            time: new Date(item[6]).getTime() / 1000,
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
      async checkTicker() {
        const url = `https://iss.moex.com/iss/securities.json?q=${this.ticker}&is_trading=1`;
  
        try {
          const response = await fetch(url);
          const data = await response.json();
          const securities = data.securities.data;
  
          if (securities.length === 0) {
            throw new Error('Invalid or non-trading ticker.');
          }
  
          const emitent = securities.find(item => item[8] !== null);
          if (emitent) {
            this.emitentTitle = emitent[8];
          } else {
            throw new Error('Emitent title not found.');
          }
  
          return true;
        } catch (error) {
          this.errorMessage = `Error: ${error.message}`;
          this.emitentTitle = '';
          return false;
        }
      },
      async updateChart() {
        const start = new Date(this.startDay);
        const end = new Date(this.endDay);
  
        if (isNaN(start) || isNaN(end)) {
          this.errorMessage = 'Invalid date format.';
          return;
        }
  
        if (start >= end) {
          this.errorMessage = 'Start date must be earlier than end date.';
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
      }
    }
  };
  </script>
  
  <style>
  .chart-container {
    position: relative;
    margin-top: 20px;
  }
  form {
    display: flex;
    flex-direction: column;
    gap: 10px;
    margin-bottom: 20px;
  }
  .error {
    color: red;
  }
  .warning {
    color: orange;
  }
  .emitent {
    margin-top: 10px;
    font-weight: bold;
  }
  </style>
  