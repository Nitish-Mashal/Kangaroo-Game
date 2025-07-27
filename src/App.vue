<template>
  <div>
    <kangarooGameComponent />
    <router-view />
  </div>
</template>

<script>
import kangarooGameComponent from './components/kangarooGameComponent.vue'



export default {
  name: 'App',
  components: {
    kangarooGameComponent,
  }
}
</script>
<!-- <style>
*{
  font-family: "Lexend", sans-serif;
  color: #4E4E4E;
}
</style> -->
