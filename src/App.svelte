<script>
  import Router from 'svelte-spa-router'
  import Home from './pages/Home.svelte'
  import Background from './pages/Background.svelte'
  import Weapon from './pages/Weapon.svelte'

  const routes = {
    '/': Home,
    '/background': Background,
    '/weapon': Weapon,
  }
</script>

<main>
  <Router {routes} />
</main>

<style>
  :global(html) {
    height: 100%;
  }
  
  :global(body) {
    margin: 0;
    padding: 0;
    height: 100%;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
      'Ubuntu', 'Cantarell', 'Open Sans', 'Helvetica Neue', sans-serif;
  }
  
  :global(#app) {
    height: 100%;
  }
  
  main {
    height: 100%;
  }
</style>