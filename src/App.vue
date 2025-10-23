<template>
  <div id="app">
    <ImageGallery />
  </div>
</template>

<script>
import ImageGallery from './components/ImageGallery.vue';
export default {
  name: 'App',
  components: { ImageGallery }
};
</script>

<style>
#app {
  font-family: 'Inter', system-ui;
  text-align: center;
  padding: 20px;
  background: #f8fafc;
}
</style>
