<script setup lang="ts">
// 首页组件
</script>

<template>
  <div class="home-page">
    <section class="hero-section">
      <div class="hero-content">
        <h2>在地图上规划和绘制最优路径</h2>
        <p>上传您的地图，添加节点和连接，计算最优路径并以动画方式展示</p>
        <router-link to="/map" class="cta-button">开始使用</router-link>
      </div>
    </section>

    <section class="features-section">
      <h2>主要功能</h2>
      <div class="features-grid">
        <div class="feature-card">
          <div class="feature-icon">📍</div>
          <h3>自定义地图</h3>
          <p>上传并使用您自己的地图图片</p>
        </div>

        <div class="feature-card">
          <div class="feature-icon">🔗</div>
          <h3>节点管理</h3>
          <p>添加、移动和删除路径节点</p>
        </div>

        <div class="feature-card">
          <div class="feature-icon">🔄</div>
          <h3>连接关系</h3>
          <p>创建节点间的双向连接</p>
        </div>

        <div class="feature-card">
          <div class="feature-icon">🛣️</div>
          <h3>最优路径</h3>
          <p>计算两点间的最短路径</p>
        </div>

        <div class="feature-card">
          <div class="feature-icon">📊</div>
          <h3>多点规划</h3>
          <p>支持多点路径规划和优化</p>
        </div>

        <div class="feature-card">
          <div class="feature-icon">✨</div>
          <h3>动态展示</h3>
          <p>动画方式展示路径和方向</p>
        </div>
      </div>
    </section>

    <section class="action-section">
      <h2>立即开始您的路径规划</h2>
      <router-link to="/map" class="cta-button">进入地图路径绘制</router-link>
    </section>
  </div>
</template>

<style scoped>
.home-page {
  width: 100%;
  height: 100%;
}

.hero-section {
  background: linear-gradient(135deg, #3498db, #2c3e50);
  color: white;
  padding: 4rem 3rem;
  text-align: center;
  margin-bottom: 3rem;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.hero-content {
  max-width: 900px;
  margin: 0 auto;
}

.hero-content h2 {
  font-size: 3rem;
  margin-bottom: 1.5rem;
  font-weight: 600;
  letter-spacing: -0.5px;
}

.hero-content p {
  font-size: 1.4rem;
  margin-bottom: 2.5rem;
  opacity: 0.9;
  line-height: 1.6;
}

.cta-button {
  display: inline-block;
  background-color: #e74c3c;
  color: white;
  padding: 1rem 2.5rem;
  border-radius: 4px;
  text-decoration: none;
  font-weight: bold;
  font-size: 1.2rem;
  transition: all 0.3s;
  border: none;
}

.cta-button:hover {
  background-color: #c0392b;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.features-section {
  padding: 3rem 0;
  text-align: center;
}

.features-section h2 {
  font-size: 2.2rem;
  margin-bottom: 3rem;
  color: #2c3e50;
}

.features-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 2rem;
  margin-bottom: 3rem;
}

.feature-card {
  background-color: white;
  border-radius: 8px;
  padding: 2rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.08);
  transition:
    transform 0.3s,
    box-shadow 0.3s;
  height: 100%;
}

.feature-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.1);
}

.feature-icon {
  font-size: 2.5rem;
  margin-bottom: 1.5rem;
}

.feature-card h3 {
  font-size: 1.4rem;
  margin-bottom: 1rem;
  color: #2c3e50;
}

.feature-card p {
  color: #666;
  font-size: 1.1rem;
  line-height: 1.5;
}

.action-section {
  background-color: #f8f9fa;
  padding: 4rem 3rem;
  text-align: center;
  border-radius: 8px;
  margin-top: 3rem;
  box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.05);
}

.action-section h2 {
  font-size: 2rem;
  margin-bottom: 2rem;
  color: #2c3e50;
}
</style>
