<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png" />
    <query-input />
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import QueryInput from "./components/QueryInput.vue";

export default defineComponent({
  name: "App",
  components: {
    QueryInput
  }
});
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
