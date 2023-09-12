<template>
  <loops msg="Welcome to Your Vue.js App" />
</template>

<script>
import loops from "./components/lines.vue";

export default {
  name: "App",
  components: {
    loops,
  },
};
</script>

<style>
html,
body {
  width: 100%;
  height: 100%;
  font-family: "Courier New", Courier, monospace;
  font-size: 95%;
  margin: 0;
  height: 100%;
  overflow: hidden;
}
</style>
