<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div class="app-container">
    <header>
      <div class="header-content">
        <h1 class="app-title">地图路径绘制工具</h1>
        <nav>
          <RouterLink to="/">首页</RouterLink>
          <RouterLink to="/map">地图路径绘制</RouterLink>
        </nav>
      </div>
    </header>

    <main class="main-content">
      <RouterView />
    </main>

    <footer>
      <p>地图路径绘制工具 © 2025</p>
    </footer>
  </div>
</template>

<style>
:root {
  --header-height: 60px;
  --footer-height: 50px;
  --max-content-width: 1600px;
  --side-padding: 2rem;
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  width: 100%;
}

header {
  background-color: #5d6a7d;
  color: white;
  height: var(--header-height);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
}

.header-content {
  max-width: var(--max-content-width);
  height: 100%;
  margin: 0 auto;
  padding: 0 var(--side-padding);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-title {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 500;
  color: white;
}

nav {
  display: flex;
  gap: 2.5rem;
}

nav a {
  color: #cfd9e0;
  text-decoration: none;
  font-weight: 500;
  font-size: 1.1rem;
  padding: 0.5rem 0;
  position: relative;
  transition: color 0.3s;
}

nav a:hover {
  color: white;
}

nav a.router-link-active {
  color: #3498db;
}

nav a.router-link-active::after {
  content: '';
  position: absolute;
  bottom: -5px;
  left: 0;
  width: 100%;
  height: 3px;
  background-color: #3498db;
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: var(--max-content-width);
  margin: var(--header-height) auto 0;
  padding: 2rem var(--side-padding);
  height: calc(100vh - var(--header-height) - var(--footer-height));
  overflow-y: auto;
}

footer {
  background-color: #2f3d48;
  color: white;
  height: var(--footer-height);
  box-shadow: 0 -2px 8px rgba(0, 0, 0, 0.1);
  position: fixed; /* 默认位置，跟header不同，footer是静态的 */
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
}
</style>
