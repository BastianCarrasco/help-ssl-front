<script setup>
import { ref, reactive, onMounted } from "vue";
import "./assets/vista.css";

// URL del Backend en Railway
const API_URL = "https://help-ssl-backend-production.up.railway.app/registros";

const registros = ref([]);
const loading = ref(true);

// 1) HELPERS PARA SEPARAR POR APP (CLIENTE / GLOBAL / BACKOFFICE)
const normalizeApps = (app) => {
  if (!app) return [];
  if (Array.isArray(app)) return app.map((x) => String(x).toUpperCase());
  return [String(app).toUpperCase()];
};

const hasApp = (item, appName) => {
  const apps = normalizeApps(item.app);
  return apps.includes(String(appName).toUpperCase());
};

const registrosCliente = () =>
  registros.value.filter((item) => hasApp(item, "CLIENTE"));

const registrosGlobal = () =>
  registros.value.filter((item) => hasApp(item, "GLOBAL"));

const registrosBackoffice = () =>
  registros.value.filter(
    (item) =>
      hasApp(item, "BACKOFFICE") ||
      hasApp(item, "BACK_OFFICE") ||
      hasApp(item, "BACK OFFICE") ||
      hasApp(item, "BACK-OFFICE")
  );

// 2. OBTENER DATOS (GET)
const fetchRegistros = async () => {
  loading.value = true;
  try {
    const response = await fetch(API_URL);
    if (!response.ok) throw new Error("Error en la respuesta del servidor");
    const data = await response.json();
    registros.value = data;
  } catch (error) {
    console.error("Error cargando registros:", error);
    alert("Error: No se pudo conectar con el servidor de Railway.");
  } finally {
    loading.value = false;
  }
};

// Función para procesar los saltos de línea en la vista
const formatVista = (text) => {
  if (!text) return "";
  return text.split("-/-").join("\n");
};

// Cargar al iniciar la App
onMounted(fetchRegistros);

const showFormModal = ref(false);
const showViewModal = ref(false);
const isEditing = ref(false);
const selectedItem = ref(null);

// MODELO COMPLETO SEGÚN TU JSON
const form = reactive({
  id: null,
  responsable: "",
  endpoint: "",
  tipo_endpoint: "GET",
  Objetivo_funcional: "",
  procedimiento_relacionado: "",
  datos_enviados_back_base: "", // Campo técnico
  app: ["CLIENTE"],
  seccion: "",
  vista: "",
  datos_enviados_front_back: "", // Campo técnico
  usos_filters_querys: "",
  body_json_ejemplo: "",
  response_backend: "",
  comentarios: "",
  dependencia_bloqueo: "",
  tablas_relacionadas: [],
  autor: "",
  autor_actualizacion: "",
  comentarios_actualizacion: "",
});

const openViewModal = (item) => {
  selectedItem.value = item;
  showViewModal.value = true;
};

const openCreateModal = () => {
  isEditing.value = false;
  // Limpiar formulario con valores por defecto
  Object.assign(form, {
    id: null,
    responsable: "",
    endpoint: "",
    tipo_endpoint: "GET",
    Objetivo_funcional: "",
    procedimiento_relacionado: "",
    datos_enviados_back_base: "",
    app: ["CLIENTE"],
    seccion: "",
    vista: "",
    datos_enviados_front_back: "",
    usos_filters_querys: "",
    body_json_ejemplo: "",
    response_backend: "",
    comentarios: "",
    dependencia_bloqueo: "",
    tablas_relacionadas: [],
    autor: "",
    autor_actualizacion: "",
    comentarios_actualizacion: "",
  });
  showFormModal.value = true;
};

const openEditModal = (e, item) => {
  e.stopPropagation();
  isEditing.value = true;
  // Copiamos TODOS los datos del item al formulario
  Object.assign(form, { ...item });
  showFormModal.value = true;
};

// 2. GUARDAR DATOS (POST / PUT)
const saveEntry = async () => {
  const method = isEditing.value ? "PUT" : "POST";
  const url = isEditing.value ? `${API_URL}/${form.id}` : API_URL;
  const hoy = new Date().toLocaleDateString();

  // Preparamos el payload con datos de auditoría
  const payload = {
    ...form,
    fecha_actualizacion: hoy,
    autor_actualizacion: form.responsable, // Asumimos que el responsable actualiza
    autor: isEditing.value ? form.autor : form.responsable,
    fecha_creacion: isEditing.value ? form.fecha_creacion : hoy,
  };

  try {
    const response = await fetch(url, {
      method: method,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });

    if (response.ok) {
      await fetchRegistros();
      showFormModal.value = false;
    } else {
      alert("Error al intentar guardar en el servidor.");
    }
  } catch (error) {
    console.error("Error en la petición:", error);
    alert("Hubo un fallo en la conexión.");
  }
};
</script>

<template>
  <div class="main-wrapper">
    <div class="container">
      <header class="header">
        <div>
          <h1>Service Documentation <span class="version">v2.0</span></h1>
          <p v-if="loading">Conectando con Railway API...</p>
          <p v-else>Repositorio centralizado de Endpoints y Procedimientos SSL</p>
        </div>
        <button @click="openCreateModal" class="btn-add-pro">
          + New Object
        </button>
      </header>

      <div v-if="loading" class="loader-box">
        <div class="spinner"></div>
        <p>Sincronizando registros...</p>
      </div>

      <!-- GRID CON 3 COLUMNAS (CLIENTE / GLOBAL / BACKOFFICE) -->
      <div v-else class="grid by-app-grid">
        <!-- COLUMNA CLIENTE -->
        <section class="app-col">
          <h2 class="app-col-title">CLIENTE</h2>

          <div v-for="item in registrosCliente()" :key="`cliente-${item.id}`" class="card-pro"
            @click="openViewModal(item)">
            <div class="card-pro-top">
              <span :class="[
                'method-tag',
                (item.tipo_endpoint || 'GET').toLowerCase(),
              ]">{{ item.tipo_endpoint || "GET" }}</span>
              <code class="url-text">{{ item.endpoint }}</code>
              <button @click="(e) => openEditModal(e, item)" class="edit-icon-btn">
                ✎
              </button>
            </div>
            <h3 class="card-pro-title">{{ item.Objetivo_funcional }}</h3>
            <div class="card-pro-badges">
              <span class="badge-app">APP: {{ item.app ? item.app[0] : "N/A" }}</span>
              <span class="badge-sec">{{ item.seccion }}</span>
            </div>
          </div>
        </section>

        <!-- COLUMNA GLOBAL -->
        <section class="app-col">
          <h2 class="app-col-title">GLOBAL</h2>

          <div v-for="item in registrosGlobal()" :key="`global-${item.id}`" class="card-pro"
            @click="openViewModal(item)">
            <div class="card-pro-top">
              <span :class="[
                'method-tag',
                (item.tipo_endpoint || 'GET').toLowerCase(),
              ]">{{ item.tipo_endpoint || "GET" }}</span>
              <code class="url-text">{{ item.endpoint }}</code>
              <button @click="(e) => openEditModal(e, item)" class="edit-icon-btn">
                ✎
              </button>
            </div>
            <h3 class="card-pro-title">{{ item.Objetivo_funcional }}</h3>
            <div class="card-pro-badges">
              <span class="badge-app">APP: {{ item.app ? item.app[0] : "N/A" }}</span>
              <span class="badge-sec">{{ item.seccion }}</span>
            </div>
          </div>
        </section>

        <!-- COLUMNA BACKOFFICE -->
        <section class="app-col">
          <h2 class="app-col-title">BACKOFFICE</h2>

          <div v-for="item in registrosBackoffice()" :key="`backoffice-${item.id}`" class="card-pro"
            @click="openViewModal(item)">
            <div class="card-pro-top">
              <span :class="[
                'method-tag',
                (item.tipo_endpoint || 'GET').toLowerCase(),
              ]">{{ item.tipo_endpoint || "GET" }}</span>
              <code class="url-text">{{ item.endpoint }}</code>
              <button @click="(e) => openEditModal(e, item)" class="edit-icon-btn">
                ✎
              </button>
            </div>
            <h3 class="card-pro-title">{{ item.Objetivo_funcional }}</h3>
            <div class="card-pro-badges">
              <span class="badge-app">APP: {{ item.app ? item.app[0] : "N/A" }}</span>
              <span class="badge-sec">{{ item.seccion }}</span>
            </div>
          </div>
        </section>
      </div>

      <!-- VIEW MODAL (DETALLES) -->
      <div v-if="showViewModal" class="modal-overlay" @click.self="showViewModal = false">
        <div class="modal-pro view-mode">
          <div class="modal-pro-header">
            <div class="header-main">
              <span :class="[
                'method-tag',
                (selectedItem.tipo_endpoint || 'GET').toLowerCase(),
              ]">
                {{ selectedItem.tipo_endpoint }}
              </span>
              <code class="url-large">{{ selectedItem.endpoint }}</code>
            </div>
            <button class="close-pro" @click="showViewModal = false">
              &times;
            </button>
          </div>

          <div class="modal-scroll-area">
            <h2 class="title-large">{{ selectedItem.Objetivo_funcional }}</h2>

            <div class="detail-box">
              <label>Stored Procedure & Database Logic</label>
              <pre class="code-sql">{{
                selectedItem.procedimiento_relacionado
              }}</pre>
            </div>

            <div class="stats-grid">
              <div class="stat-item">
                <label>Atributos Procedimiento (Back)</label>
                <p class="text-tag-style">
                  {{ selectedItem.datos_enviados_back_base || "N/A" }}
                </p>
              </div>
              <div class="stat-item">
                <label>Tablas Origen (GET o Verificación)</label>
                <div class="mini-tags-wrap">
                  <span v-for="(t, i) in selectedItem.tablas_relacionadas" :key="i" class="m-tag">
                    {{ Array.isArray(t) ? t[0] : t }}
                  </span>
                </div>
              </div>
              <div class="stat-item">
                <label>Tablas Destino (POST/PUT/DELETE)</label>
                <div class="mini-tags-wrap">
                  <span v-for="(t, i) in selectedItem.tablas_relacionadas" :key="i" class="m-tag destination">
                    {{ Array.isArray(t) ? t[1] : "N/A" }}
                  </span>
                </div>
              </div>
            </div>

            <div class="detail-box">
              <label>Ruta de Vista & Flujo de Navegación</label>
              <div class="vista-container-box">
                <p class="vista-text" style="white-space: pre-wrap">
                  {{ formatVista(selectedItem.vista) }}
                </p>
              </div>
            </div>

            <div class="stats-grid">
              <div class="stat-item">
                <label>Datos Front -> Back</label>
                <p>{{ selectedItem.datos_enviados_front_back || "N/A" }}</p>
              </div>
              <div class="stat-item">
                <label>Filtros / Querys Permitidos</label>
                <p>
                  <code>{{ selectedItem.usos_filters_querys || "Ninguno" }}</code>
                </p>
              </div>
              <div class="stat-item">
                <label>Dependencia / Bloqueo</label>
                <p>
                  {{
                    selectedItem.dependencia_bloqueo || "Sin dependencias"
                  }}
                </p>
              </div>
            </div>

            <div class="json-flex">
              <div class="json-half">
                <label>Body Request Example</label>
                <pre class="code-json">{{ selectedItem.body_json_ejemplo }}</pre>
              </div>
              <div class="json-half">
                <label>Backend Response Example</label>
                <pre class="code-json res">{{
                  selectedItem.response_backend
                }}</pre>
              </div>
            </div>

            <div class="audit-section-modal">
              <div class="detail-box mt-4">
                <label>Comentarios Generales</label>
                <p class="comment-text">{{ selectedItem.comentarios }}</p>
              </div>

              <div class="audit-grid-mini">
                <div>
                  <label class="mini-label">Creado por:</label>
                  <span>{{ selectedItem.autor }} ({{ selectedItem.fecha_creacion }})</span>
                </div>
                <div>
                  <label class="mini-label">Última Actualización:</label>
                  <span>{{ selectedItem.autor_actualizacion }} ({{
                    selectedItem.fecha_actualizacion
                    }})</span>
                </div>
              </div>

              <div v-if="selectedItem.comentarios_actualizacion" class="update-note">
                <label class="mini-label">Nota de la versión:</label>
                <p style="color: #f1f5f9">
                  {{ selectedItem.comentarios_actualizacion }}
                </p>
              </div>
            </div>
          </div>

          <div class="modal-pro-footer">
            <div class="footer-tags">
              <span class="badge-app">APP: {{ selectedItem.app ? selectedItem.app[0] : "N/A" }}</span>
              <span class="badge-sec">SECCIÓN: {{ selectedItem.seccion }}</span>
            </div>
            <span class="id-tag">Object ID: #{{ selectedItem.id }}</span>
          </div>
        </div>
      </div>

      <!-- FORM MODAL -->
      <div v-if="showFormModal" class="modal-overlay" @click.self="showFormModal = false">
        <div class="modal-pro">
          <div class="modal-pro-header">
            <h3 class="title-modal-fix">
              {{ isEditing ? "Edit Service" : "Create Service" }}
            </h3>
            <button class="close-pro" @click="showFormModal = false">
              &times;
            </button>
          </div>

          <div class="modal-scroll-area">
            <div class="form-grid-pro">
              <div class="field">
                <label>Responsable</label><input v-model="form.responsable" type="text" />
              </div>
              <div class="field">
                <label>Method</label>
                <select v-model="form.tipo_endpoint">
                  <option>GET</option>
                  <option>POST</option>
                  <option>PUT</option>
                  <option>DELETE</option>
                </select>
              </div>
              <div class="field full">
                <label>Endpoint URL</label><input v-model="form.endpoint" type="text" />
              </div>
              <div class="field full">
                <label>Nombre Función / Objetivo</label>
                <input v-model="form.Objetivo_funcional" type="text" />
              </div>

              <div class="field full">
                <label>Stored Procedure Relacionado</label>
                <textarea v-model="form.procedimiento_relacionado" rows="2"></textarea>
              </div>
              <div class="field full">
                <label>Atributos Procedimiento (Base Datos)</label>
                <textarea v-model="form.datos_enviados_back_base" rows="2"
                  placeholder="nidempresa, scodobra..."></textarea>
              </div>
              <div class="field">
                <label>Sección App</label>
                <input v-model="form.seccion" type="text" placeholder="Ej: Solicitudes de Pedido" />
              </div>
              <div class="field">
                <label>Dependencia / Bloqueo</label>
                <input v-model="form.dependencia_bloqueo" type="text" />
              </div>

              <div class="field full">
                <label>Flujo de Vista (Usa -/- para saltos)</label>
                <textarea v-model="form.vista" rows="3"></textarea>
              </div>
              <div class="field full">
                <label>Datos Enviados (Front -> Back)</label>
                <textarea v-model="form.datos_enviados_front_back" rows="2"
                  placeholder="Primaria: ... Modificables: ..."></textarea>
              </div>
              <div class="field full">
                <label>Body JSON Ejemplo</label>
                <textarea v-model="form.body_json_ejemplo" rows="4" class="mono"></textarea>
              </div>
              <div class="field full">
                <label>Response Backend Ejemplo</label>
                <textarea v-model="form.response_backend" rows="2" class="mono"></textarea>
              </div>

              <div class="field full">
                <label>Comentarios Generales</label>
                <textarea v-model="form.comentarios" rows="2"></textarea>
              </div>
              <div class="field full" v-if="isEditing">
                <label>Comentarios de esta actualización</label>
                <input v-model="form.comentarios_actualizacion" type="text" />
              </div>
            </div>
          </div>

          <div class="modal-actions-pro">
            <button @click="showFormModal = false" class="btn-cancel">
              Cancel
            </button>
            <button @click="saveEntry" class="btn-save">Save Data</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.audit-section-modal {
  margin-top: 30px;
  padding: 20px;
  background: rgba(15, 23, 42, 0.5);
  border-radius: 12px;
  border: 1px solid #334155;
}

.audit-grid-mini {
  display: flex;
  justify-content: space-between;
  margin-top: 15px;
  font-size: 0.8rem;
  color: #94a3b8;
}

.mini-label {
  font-weight: 800;
  text-transform: uppercase;
  font-size: 0.65rem;
  color: #6366f1;
  margin-right: 8px;
}

.text-tag-style {
  background: #0f172a;
  padding: 8px;
  border-radius: 6px;
  font-family: "JetBrains Mono", monospace;
  font-size: 0.8rem;
  color: #cbd5e1;
}

.id-tag {
  font-family: "JetBrains Mono", monospace;
  font-weight: bold;
  color: #475569;
}

.m-tag.destination {
  border-color: #a855f7;
  color: #a855f7;
  background: rgba(168, 85, 247, 0.1);
}

/* NUEVAS CLASES (SIN CAMBIAR LAS EXISTENTES) */
.by-app-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(260px, 1fr));
  gap: 18px;
  align-items: start;
}

.app-col {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.app-col-title {
  font-size: 0.85rem;
  font-weight: 900;
  letter-spacing: 0.08em;
  color: #94a3b8;
  text-transform: uppercase;
  padding: 10px 12px;
  border: 1px solid #334155;
  border-radius: 12px;
  background: rgba(15, 23, 42, 0.35);
}
</style>