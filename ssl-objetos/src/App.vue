<template>
  <div class="page">
    <header class="hero">
      <div class="hero-top">
        <div>
          <h1 class="title">SSL Objetos Data</h1>
          <p class="subtitle">
            Catálogo de endpoints y objetos. Filtra por APP o busca por endpoint,
            vista, responsable, etc.
          </p>
        </div>

        <div class="stats" v-if="!loading && !error">
          <div class="stat">
            <div class="stat-num">{{ rows.length }}</div>
            <div class="stat-label">Registros</div>
          </div>
          <div class="stat">
            <div class="stat-num">{{ apps.length }}</div>
            <div class="stat-label">Apps</div>
          </div>
        </div>
      </div>

      <div class="toolbar">
        <div class="field">
          <label class="label">Buscar</label>
          <input v-model="query" class="input" placeholder="Ej: buscar-solicitudes, cotización, back office..." />
        </div>

        <div class="field">
          <label class="label">APP</label>
          <select v-model="selectedApp" class="select">
            <option value="__ALL__">Todas</option>
            <option v-for="app in apps" :key="app" :value="app">
              {{ app }}
            </option>
          </select>
        </div>

        <div class="field">
          <label class="label">Orden</label>
          <select v-model="sortMode" class="select">
            <option value="endpoint">ENDPOINT</option>
            <option value="vista">VISTA</option>
            <option value="fecha">Fecha creación</option>
          </select>
        </div>
      </div>
    </header>

    <main>
      <div v-if="loading" class="state">
        <div class="spinner" />
        <div>Cargando datos…</div>
      </div>

      <div v-else-if="error" class="state error">
        <div class="error-title">No se pudo cargar el CSV</div>
        <div class="error-msg">{{ error }}</div>
        <div class="hint">
          Revisa `VITE_URL_CSV` en tu `.env` y reinicia `npm run dev`.
        </div>
      </div>

      <div v-else>
        <div v-if="filteredRows.length === 0" class="state">
          No hay resultados para ese filtro/búsqueda.
        </div>

        <section v-for="app in visibleApps" :key="app" class="section">
          <div class="section-head">
            <div class="section-title">{{ app }}</div>
            <div class="badge">{{ groupedFiltered[app]?.length ?? 0 }}</div>
          </div>

          <div class="cards">
            <article v-for="item in groupedFiltered[app]" :key="item.__id" class="card" @click="open(item)">
              <div class="card-head">
                <div class="endpoint">
                  {{ item.ENDPOINT || "(sin endpoint)" }}
                </div>
                <span class="chip method" :class="methodClass(item.ENDPOINT)">
                  {{ guessMethod(item.ENDPOINT) }}
                </span>
              </div>

              <div class="card-body">
                <div class="line">
                  <span class="k">Vista</span>
                  <span class="v">{{ item.VISTA || "-" }}</span>
                </div>
                <div class="line">
                  <span class="k">Responsable</span>
                  <span class="v">{{ item.Responsable || "-" }}</span>
                </div>
                <div class="line">
                  <span class="k">Qué hace</span>
                  <span class="v clamp">
                    {{ item["QUE HACE?"] || "-" }}
                  </span>
                </div>
              </div>

              <div class="card-foot">
                <span class="chip" v-if="hasValue(item['USOS/FILTROS/QUERYS'])">
                  tiene filtros
                </span>
                <span class="chip" v-if="hasValue(item['BODY JSON EJEMPLO'])">
                  body json
                </span>
                <span class="chip" v-if="hasValue(item['RESPONSE BACKEND'])">
                  response
                </span>

                <span class="cta">Ver detalle</span>
              </div>
            </article>
          </div>
        </section>
      </div>
    </main>

    <!-- Modal detalle -->
    <div v-if="active" class="modal-backdrop" @click.self="close">
      <div class="modal">
        <div class="modal-head">
          <div>
            <div class="modal-title">
              {{ active.ENDPOINT || "(sin endpoint)" }}
            </div>
            <div class="modal-sub">
              <span class="chip soft">{{ active.APP || "Sin APP" }}</span>
              <span class="dot">•</span>
              <span class="muted">Vista:</span> {{ active.VISTA || "-" }}
            </div>
          </div>

          <button class="btn-close" @click="close">Cerrar</button>
        </div>

        <div class="modal-content">
          <div class="grid2">
            <div class="info">
              <div class="info-title">Resumen</div>
              <div class="info-row">
                <div class="k">Responsable</div>
                <div class="v">{{ active.Responsable || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Qué hace</div>
                <div class="v">{{ active["QUE HACE?"] || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Procedimiento relacionado</div>
                <div class="v">{{ active["PROCEDIMIENTO RELACIONADO"] || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Datos enviados front → back</div>
                <div class="v">{{ active["DATOS ENVIADOS front -> back"] || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Datos enviados back → BD</div>
                <div class="v">{{ active["DATOS ENVIADOS back -> base datos"] || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Usos / filtros / querys</div>
                <div class="v">{{ active["USOS/FILTROS/QUERYS"] || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Comentarios</div>
                <div class="v">{{ active.COMENTARIOS || "-" }}</div>
              </div>
              <div class="info-row">
                <div class="k">Dependencia / bloqueo</div>
                <div class="v">{{ active["DEPENDENCIA/BLOQUEO"] || "-" }}</div>
              </div>
            </div>

            <div class="code">
              <div class="info-title">Ejemplos</div>

              <div class="code-block">
                <div class="code-head">BODY JSON EJEMPLO</div>
                <pre class="pre">{{ active["BODY JSON EJEMPLO"] || "-" }}</pre>
              </div>

              <div class="code-block">
                <div class="code-head">RESPONSE BACKEND</div>
                <pre class="pre">{{ active["RESPONSE BACKEND"] || "-" }}</pre>
              </div>
            </div>
          </div>

          <div class="meta">
            <div class="meta-item">
              <div class="k">Autor</div>
              <div class="v">{{ active.Autor || "-" }}</div>
            </div>
            <div class="meta-item">
              <div class="k">Fecha creación</div>
              <div class="v">{{ active["Fecha creación"] || "-" }}</div>
            </div>
            <div class="meta-item">
              <div class="k">Autor actualización</div>
              <div class="v">{{ active["Autor actualización"] || "-" }}</div>
            </div>
            <div class="meta-item">
              <div class="k">Fecha actualización</div>
              <div class="v">{{ active["Fecha actualización"] || "-" }}</div>
            </div>
            <div class="meta-item meta-wide">
              <div class="k">Comentarios actualización</div>
              <div class="v">{{ active["COMENTARIOS ACTUALIZAC"] || "-" }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, ref } from "vue";
import Papa from "papaparse";

const CSV_URL =
  import.meta.env.VITE_URL_CSV ??
  "https://docs.google.com/spreadsheets/d/e/2PACX-1vRadaqk-RjbNemEdRovbfttZM-bsJ5ogrG0iQYxVpop0r_E_LG4_F1VYBP7oeP41ONp4ovnbERFLISY/pub?gid=0&single=true&output=csv";

const headers = ref([]);
const rows = ref([]);
const loading = ref(true);
const error = ref("");

const selectedApp = ref("__ALL__");
const query = ref("");
const sortMode = ref("endpoint");

const active = ref(null);

const hasValue = (v) => String(v ?? "").trim().length > 0;

const open = (item) => {
  active.value = item;
};
const close = () => {
  active.value = null;
};

const normalize = (s) =>
  String(s ?? "")
    .toLowerCase()
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "");

const guessMethod = (endpoint) => {
  const s = normalize(endpoint);
  if (s.startsWith("get ")) return "GET";
  if (s.startsWith("post ")) return "POST";
  if (s.startsWith("put ")) return "PUT";
  if (s.startsWith("delete ")) return "DELETE";
  return "API";
};

const methodClass = (endpoint) => {
  const m = guessMethod(endpoint);
  if (m === "GET") return "get";
  if (m === "POST") return "post";
  if (m === "PUT") return "put";
  if (m === "DELETE") return "del";
  return "api";
};

const apps = computed(() => {
  const set = new Set(rows.value.map((r) => (r.APP || "Sin APP").trim?.() || r.APP || "Sin APP"));
  return Array.from(set).sort((a, b) => a.localeCompare(b, "es"));
});

const filteredRows = computed(() => {
  const q = normalize(query.value);
  return rows.value.filter((r) => {
    const app = (r.APP && String(r.APP).trim()) || "Sin APP";
    if (selectedApp.value !== "__ALL__" && app !== selectedApp.value) return false;

    if (!q) return true;

    const haystack = normalize(
      [
        r.APP,
        r.ENDPOINT,
        r.VISTA,
        r.Responsable,
        r["QUE HACE?"],
        r["PROCEDIMIENTO RELACIONADO"],
        r["USOS/FILTROS/QUERYS"],
      ].join(" | ")
    );

    return haystack.includes(q);
  });
});

const groupedFiltered = computed(() => {
  const out = {};
  for (const r of filteredRows.value) {
    const app = (r.APP && String(r.APP).trim()) || "Sin APP";
    if (!out[app]) out[app] = [];
    out[app].push(r);
  }

  // Orden dentro del grupo
  const cmp = (a, b) => {
    if (sortMode.value === "vista") {
      return String(a.VISTA ?? "").localeCompare(String(b.VISTA ?? ""), "es");
    }
    if (sortMode.value === "fecha") {
      return String(a["Fecha creación"] ?? "").localeCompare(
        String(b["Fecha creación"] ?? ""),
        "es"
      );
    }
    return String(a.ENDPOINT ?? "").localeCompare(String(b.ENDPOINT ?? ""), "es");
  };

  for (const k of Object.keys(out)) out[k].sort(cmp);

  return out;
});

const visibleApps = computed(() => {
  if (selectedApp.value === "__ALL__") {
    return Object.keys(groupedFiltered.value).sort((a, b) =>
      a.localeCompare(b, "es")
    );
  }
  return Object.keys(groupedFiltered.value);
});

onMounted(async () => {
  try {
    const res = await fetch(CSV_URL);
    if (!res.ok) throw new Error(`HTTP ${res.status} al pedir el CSV`);

    const csvText = await res.text();
    const parsed = Papa.parse(csvText, { header: true, skipEmptyLines: true });

    if (parsed.errors?.length) console.warn(parsed.errors);

    const data = (parsed.data || [])
      .filter((r) => Object.values(r).some((v) => hasValue(v)))
      .map((r, i) => ({
        __id: `${i}-${r.ENDPOINT ?? ""}-${r.APP ?? ""}`,
        ...r,
      }));

    rows.value = data;
    headers.value =
      parsed.meta?.fields?.filter(Boolean) ?? Object.keys(data?.[0] ?? {});
  } catch (e) {
    error.value = e?.message ?? String(e);
  } finally {
    loading.value = false;
  }
});
</script>

<style scoped>
/* Base */
.page {
  min-height: 100vh;
  background: linear-gradient(180deg, #31608f 0%, rgb(99, 108, 150) 40%);
  color: #0f172a;
  padding: 18px;
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto,
    "Helvetica Neue", Arial, "Noto Sans", "Liberation Sans", sans-serif;
}

/* Hero */
.hero {
  background: radial-gradient(1200px 400px at 10% 0%,
      rgba(99, 102, 241, 0.12),
      transparent 60%),
    radial-gradient(1200px 400px at 90% 0%,
      rgba(16, 185, 129, 0.12),
      transparent 60%),
    #e0e0e0;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.06);
  margin-bottom: 16px;
}

.hero-top {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: start;
  flex-wrap: wrap;
}

.title {
  margin: 0;
  font-size: 28px;
  letter-spacing: -0.02em;
}

.subtitle {
  margin: 6px 0 0;
  color: #475569;
  max-width: 72ch;
}

.stats {
  display: flex;
  gap: 10px;
}

.stat {
  background: #0b1220;
  color: #e5e7eb;
  border-radius: 14px;
  padding: 10px 12px;
  min-width: 110px;
}

.stat-num {
  font-size: 20px;
  font-weight: 800;
}

.stat-label {
  font-size: 12px;
  color: #cbd5e1;
}

/* Toolbar */
.toolbar {
  margin-top: 14px;
  display: grid;
  grid-template-columns: 1.4fr 0.8fr 0.8fr;
  gap: 12px;
}

@media (max-width: 900px) {
  .toolbar {
    grid-template-columns: 1fr;
  }
}

.field {
  display: grid;
  gap: 6px;
}

.label {
  font-size: 12px;
  color: #334155;
  font-weight: 700;
}

.input,
.select {
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 10px 12px;
  outline: none;
  background: #fff;
}

.input:focus,
.select:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.12);
}

/* States */
.state {
  border: 1px dashed #e2e8f0;
  border-radius: 16px;
  padding: 18px;
  color: #475569;
  display: flex;
  gap: 10px;
  align-items: center;
  justify-content: center;
  background: #fff;
}

.state.error {
  border-style: solid;
  border-color: #fecaca;
  background: #fff1f2;
  color: #7f1d1d;
  flex-direction: column;
}

.error-title {
  font-weight: 800;
}

.error-msg {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    "Liberation Mono", "Courier New", monospace;
  font-size: 12px;
  opacity: 0.9;
}

.hint {
  font-size: 13px;
  color: #991b1b;
}

.spinner {
  width: 16px;
  height: 16px;
  border: 2px solid #cbd5e1;
  border-top-color: #6366f1;
  border-radius: 999px;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Sections */
.section {
  background: #d8e1e7ce;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 12px;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.04);
  margin-bottom: 14px;
}

.section-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 6px 6px 10px;
}

.section-title {
  font-size: 15px;
  font-weight: 900;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  color: #0f172a;
}

.badge {
  font-size: 12px;
  font-weight: 800;
  background: #eef2ff;
  color: #3730a3;
  padding: 4px 10px;
  border-radius: 999px;
  border: 1px solid #e0e7ff;
}

/* Cards grid: 3 columns */
.cards {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
}

@media (max-width: 1200px) {
  .cards {
    grid-template-columns: repeat(2, minmax(0, 1fr));

  }
}

@media (max-width: 800px) {
  .cards {
    grid-template-columns: 1fr;
  }
}

/* Card */
.card {
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 12px;
  background: linear-gradient(180deg, #ffffff 0%, #fbfdff 100%);
  cursor: pointer;
  transition: transform 0.08s ease, box-shadow 0.12s ease,
    border-color 0.12s ease;
}

.card:hover {
  transform: translateY(-1px);
  border-color: #c7d2fe;
  box-shadow: 0 14px 40px rgba(15, 23, 42, 0.08);
}

.card-head {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  align-items: start;
}

.endpoint {
  font-weight: 900;
  overflow-wrap: anywhere;
  letter-spacing: -0.01em;
}

.card-body {
  margin-top: 10px;
  display: grid;
  gap: 6px;
}

.line {
  display: grid;
  grid-template-columns: 110px 1fr;
  gap: 10px;
  font-size: 13px;
}

.k {
  color: #64748b;
  font-weight: 800;
}

.v {
  color: #0f172a;
  overflow-wrap: anywhere;
}

.clamp {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.card-foot {
  margin-top: 10px;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
}

.cta {
  color: #4f46e5;
  font-weight: 900;
  font-size: 13px;
}

/* Chips */
.chip {
  font-size: 12px;
  font-weight: 800;
  background: #f1f5f9;
  color: #0f172a;
  padding: 4px 10px;
  border-radius: 999px;
  border: 1px solid #e2e8f0;
}

.chip.soft {
  background: #eef2ff;
  border-color: #e0e7ff;
  color: #3730a3;
}

.chip.method {
  border: none;
}

.chip.method.get {
  background: rgba(16, 185, 129, 0.14);
  color: #065f46;
}

.chip.method.post {
  background: rgba(99, 102, 241, 0.16);
  color: #3730a3;
}

.chip.method.put {
  background: rgba(245, 158, 11, 0.18);
  color: #92400e;
}

.chip.method.del {
  background: rgba(239, 68, 68, 0.16);
  color: #7f1d1d;
}

.chip.method.api {
  background: rgba(148, 163, 184, 0.25);
  color: #334155;
}

/* Modal */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(2, 6, 23, 0.6);
  display: grid;
  place-items: center;
  padding: 18px;
  z-index: 50;
}

.modal {
  width: min(1100px, 100%);
  max-height: 86vh;
  overflow: auto;
  background: #dadada;
  border-radius: 18px;
  border: 1px solid #e5e7eb;
  box-shadow: 0 30px 80px rgba(15, 23, 42, 0.35);
}

.modal-head {
  position: sticky;
  top: 0;
  background: #ffffff;
  border-bottom: 1px solid #e5e7eb;
  padding: 14px 14px 12px;
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: start;
}

.modal-title {
  font-weight: 1000;
  font-size: 16px;
  overflow-wrap: anywhere;
}

.modal-sub {
  margin-top: 6px;
  color: #475569;
  font-size: 13px;
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.dot {
  color: #94a3b8;
}

.muted {
  color: #64748b;
}

.btn-close {
  border: 1px solid #e2e8f0;
  background: #fff;
  border-radius: 12px;
  padding: 8px 10px;
  cursor: pointer;
  font-weight: 800;
}

.btn-close:hover {
  background: #f8fafc;
}

.modal-content {
  padding: 14px;
}

.grid2 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 14px;
}

@media (max-width: 900px) {
  .grid2 {
    grid-template-columns: 1fr;
  }
}

.info,
.code {
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 12px;
  background: #ffffff;
}

.info-title {
  font-weight: 1000;
  margin-bottom: 10px;
}

.info-row {
  display: grid;
  grid-template-columns: 220px 1fr;
  gap: 12px;
  padding: 8px 0;
  border-bottom: 1px dashed #e2e8f0;
}

.info-row:last-child {
  border-bottom: none;
}

.code-block {
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  overflow: hidden;
  margin-bottom: 12px;
}

.code-head {
  background: #f8fafc;
  border-bottom: 1px solid #e5e7eb;
  padding: 8px 10px;
  font-weight: 900;
  color: #334155;
}

.pre {
  margin: 0;
  padding: 10px;
  background: #0b1220;
  color: #e5e7eb;
  white-space: pre-wrap;
  word-break: break-word;
  max-height: 260px;
  overflow: auto;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    "Liberation Mono", "Courier New", monospace;
  font-size: 12px;
  line-height: 1.3;
}

/* Meta */
.meta {
  margin-top: 14px;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 12px;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

@media (max-width: 900px) {
  .meta {
    grid-template-columns: 1fr;
  }
}

.meta-item {
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  padding: 10px;
  background: #fff;
}

.meta-wide {
  grid-column: 1 / -1;
}
</style>