<template>
  <div id="app">
    <CandleChart />
  </div>
</template>

<script>
import CandleChart from './components/CandleChart.vue';

export default {
  name: 'App',
  components: {
    CandleChart
  }
};
</script>

<style>
#app {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}
</style>
