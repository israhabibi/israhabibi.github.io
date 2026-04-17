---
title: Home
---

<style>
.hero {
  text-align: center;
  padding: 4rem 1rem 3rem;
}
.hero-title {
  font-size: 2.8rem;
  font-weight: 700;
  line-height: 1.2;
  margin-bottom: 0.5rem;
}
.hero-sub {
  font-size: 1.15rem;
  color: var(--md-default-fg-color--light);
  margin-bottom: 2rem;
}
.hero-badges {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 0.5rem;
  margin-bottom: 2.5rem;
}
.badge {
  display: inline-block;
  padding: 0.3rem 0.8rem;
  border-radius: 2rem;
  font-size: 0.78rem;
  font-weight: 600;
  letter-spacing: 0.03em;
  background: var(--md-primary-fg-color);
  color: var(--md-primary-bg-color);
  opacity: 0.85;
}
.badge.outline {
  background: transparent;
  border: 1.5px solid var(--md-primary-fg-color);
  color: var(--md-primary-fg-color);
}
.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.2rem;
  max-width: 860px;
  margin: 0 auto 3.5rem;
  text-align: left;
}
.card {
  border: 1px solid var(--md-default-fg-color--lightest);
  border-radius: 0.75rem;
  padding: 1.4rem 1.4rem 1.1rem;
  transition: box-shadow 0.2s, transform 0.2s;
  text-decoration: none !important;
  color: inherit !important;
  display: block;
  background: var(--md-default-bg-color);
}
.card:hover {
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  transform: translateY(-2px);
}
.card-icon { font-size: 1.8rem; margin-bottom: 0.6rem; display: block; }
.card-title { font-size: 1rem; font-weight: 700; margin-bottom: 0.3rem; }
.card-desc { font-size: 0.83rem; color: var(--md-default-fg-color--light); line-height: 1.5; }
.divider {
  max-width: 860px;
  margin: 0 auto 2.5rem;
  border: none;
  border-top: 1px solid var(--md-default-fg-color--lightest);
}
.stack-section {
  max-width: 860px;
  margin: 0 auto 3.5rem;
  text-align: center;
}
.stack-section h2 {
  font-size: 1.1rem;
  font-weight: 600;
  color: var(--md-default-fg-color--light);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  margin-bottom: 1.2rem;
}
.stack-pills { display: flex; flex-wrap: wrap; justify-content: center; gap: 0.6rem; }
.pill {
  padding: 0.35rem 0.9rem;
  border-radius: 0.4rem;
  font-size: 0.82rem;
  font-weight: 500;
  background: var(--md-code-bg-color);
  color: var(--md-code-fg-color);
  font-family: var(--md-code-font-family);
}
.recent-section {
  max-width: 860px;
  margin: 0 auto 4rem;
  text-align: left;
}
.recent-section h2 {
  font-size: 1.1rem;
  font-weight: 600;
  color: var(--md-default-fg-color--light);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  margin-bottom: 1rem;
  text-align: center;
}
.recent-item {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  padding: 0.9rem 0;
  border-bottom: 1px solid var(--md-default-fg-color--lightest);
  text-decoration: none !important;
  color: inherit !important;
}
.recent-item:last-child { border-bottom: none; }
.recent-date {
  font-size: 0.78rem;
  color: var(--md-default-fg-color--lighter);
  font-family: var(--md-code-font-family);
  min-width: 90px;
  padding-top: 0.15rem;
}
.recent-label {
  display: inline-block;
  font-size: 0.7rem;
  font-weight: 700;
  padding: 0.1rem 0.5rem;
  border-radius: 0.3rem;
  margin-right: 0.4rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
.label-blog  { background: #dbeafe; color: #1d4ed8; }
.label-til   { background: #fef9c3; color: #854d0e; }
.label-tool  { background: #f3e8ff; color: #7e22ce; }
.recent-title { font-size: 0.93rem; font-weight: 500; }
.footer-note {
  text-align: center;
  font-size: 0.82rem;
  color: var(--md-default-fg-color--lighter);
  margin-bottom: 2rem;
}
</style>

<div class="hero">
  <div class="hero-title">👋 Hey, I'm Isra</div>
  <div class="hero-sub">Data Engineer · Building pipelines, breaking things, writing it all down.</div>
  <div class="hero-badges">
    <span class="badge">AWS</span>
    <span class="badge">Spark / EMR</span>
    <span class="badge">Airflow</span>
    <span class="badge">ClickHouse</span>
    <span class="badge">Kubernetes</span>
    <span class="badge outline">always learning</span>
  </div>
</div>

<div class="cards-grid">
  <a class="card" href="blog/">
    <span class="card-icon">✍️</span>
    <div class="card-title">Blog</div>
    <div class="card-desc">Long-form posts — benchmarks, deep dives, and investigations.</div>
  </a>
  <a class="card" href="til/">
    <span class="card-icon">💡</span>
    <div class="card-title">TIL</div>
    <div class="card-desc">Single discoveries. Too small for a post, too useful to forget.</div>
  </a>
</div>

<hr class="divider">

<div class="stack-section">
  <h2>Daily Stack</h2>
  <div class="stack-pills">
    <span class="pill">Python</span>
    <span class="pill">PySpark</span>
    <span class="pill">SQL</span>
    <span class="pill">Airflow 3</span>
    <span class="pill">EMR on EKS</span>
    <span class="pill">S3</span>
    <span class="pill">Redshift</span>
    <span class="pill">ClickHouse</span>
    <span class="pill">Dynatrace</span>
    <span class="pill">Metabase</span>
    <span class="pill">k3d</span>
    <span class="pill">Docker</span>
    <span class="pill">MySQL 8</span>
    <span class="pill">Hudi</span>
  </div>
</div>

<hr class="divider">

<div class="recent-section">
  <h2>Recently Added</h2>

  <a class="recent-item" href="blog/2026/04/16/how-i-use-claude-as-a-data-engineer/">
    <span class="recent-date">2026-04-16</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">How I Use Claude as a Data Engineer</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2026/04/16/clickhouse-docker-vs-k3d-benchmark/">
    <span class="recent-date">2026-04-16</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">ClickHouse: Docker vs Kubernetes (k3d) Benchmark</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2026/04/10/setting-up-an-old-ubuntu-desktop-as-a-home-server/">
    <span class="recent-date">2026-04-10</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">Setting Up an Old Ubuntu Desktop as a Home Server</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2025/10/20/2-laptop-ha-homelab-auto-failover-with-cloudflare-syncthing/">
    <span class="recent-date">2025-10-20</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">2-Laptop HA Homelab: Auto-Failover with Cloudflare + Syncthing</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2025/10/10/wake-on-lan-on-an-ubuntu-laptop-server/">
    <span class="recent-date">2025-10-10</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">Wake-on-LAN on an Ubuntu Laptop Server</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2025/10/05/k3s-on-gcp-free-tier-single-node-to-3-node-cluster/">
    <span class="recent-date">2025-10-05</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">K3s on GCP: Free-Tier Single Node to 3-Node Cluster</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2025/09/25/supervisor-manager-one-command-python-app-deployment/">
    <span class="recent-date">2025-09-25</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">Supervisor Manager: One-Command Python App Deployment</span>
    </div>
  </a>

  <a class="recent-item" href="blog/2025/09/15/pyspark-hudi-minio-local-development-setup/">
    <span class="recent-date">2025-09-15</span>
    <div>
      <span class="recent-label label-blog">Blog</span>
      <span class="recent-title">PySpark + Hudi + MinIO Local Development Setup</span>
    </div>
  </a>

  <a class="recent-item" href="til/">
    <span class="recent-date">2026-04-16</span>
    <div>
      <span class="recent-label label-til">TIL</span>
      <span class="recent-title">k3d nginx routes to node port, not NodePort</span>
    </div>
  </a>
</div>

<p class="footer-note">Updated whenever something breaks and I figure out why.</p>
