<template>
    <div id="app">
        <Solid/>
    </div>
</template>

<script>
import Solid from './components/Solid.vue'

export default {
    name: 'App',
    components: {
        Solid
    }
}
</script>

<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    height: 100vh;
}
</style>
