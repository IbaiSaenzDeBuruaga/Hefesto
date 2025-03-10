<template>
  <!-- Stats Cards -->
  <div class="row g-4 mb-4">
    <!-- Tarjeta de incidencias pendientes -->
    <div class="col-sm-6 col-md-3">
      <div class="card glassmorphic-card colored-shadow-pending">
        <div class="card-body">
          <h6>Incidencias pendientes</h6>
          <div class="d-flex justify-content-between align-items-center">
            <h2 class="pendiente-h2">{{ incidenciasPendientesCount }}</h2>
             <!-- Icono de incidencias pendientes -->
            <img src="../assets/images/icons/pendientes.svg">
          </div>
        </div>
      </div>
    </div>
     <!-- Tarjeta de incidencias en curso -->
    <div class="col-sm-6 col-md-3">
      <div class="card glassmorphic-card colored-shadow-in-progress">
        <div class="card-body">
          <h6>Incidencias en curso</h6>
          <div class="d-flex justify-content-between align-items-center">
            <h2 class="curso-h2">{{ incidenciasEnCursoCount }}</h2>
             <!-- Icono de incidencias en curso -->
            <img src="../assets/images/icons/curso.svg">
          </div>
        </div>
      </div>
    </div>
    <!-- Tarjeta de incidencias cerradas -->
    <div class="col-sm-6 col-md-3">
      <div class="card glassmorphic-card colored-shadow-closed">
        <div class="card-body">
          <h6>Incidencias cerradas</h6>
          <div class="d-flex justify-content-between align-items-center">
            <h2 class="cerrado-h2">{{ incidenciasCerradasCount }}</h2>
             <!-- Icono de incidencias cerradas -->
            <img src="../assets/images/icons/cerrados.svg">
          </div>
        </div>
      </div>
    </div>
    <!-- Tarjeta de total de incidencias -->
    <div class="col-sm-6 col-md-3">
      <div class="card glassmorphic-card colored-shadow-total">
        <div class="card-bodyTotal">
          <h6>Total de incidencias</h6>
          <div class="d-flex justify-content-between align-items-center">
            <h2 class="total-h2">{{ totalIncidenciasCount }}</h2>
            <!-- Icono de total de incidencias -->
            <img src="../assets/images/icons/total.svg">
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Main Content Area -->
  <div class="row g-4">
    <!-- Left Column - Charts -->
    <div class="col-md-4">
       <!-- Tarjeta de incidencias abiertas hoy -->
      <div class="card glassmorphic-card mb-4">
        <div class="card-body">
          <div class="d-flex justify-content-between mb-4">
             <!-- Título e información de incidencias abiertas hoy -->
            <div class="incidenciasabiertos">
              <h6 class="text-muted mb-1">Incidencias abiertas</h6>
              <div class="d-flex align-items-baseline">
                 <!-- Contador de incidencias abiertas hoy -->
                <span class="abiertos-h3 mb-0">{{ incidenciasAbiertasHoy }}</span>
                <span class="text-muted ms-2" style="color: rgba(116, 116, 116, 1) !important;">hoy</span>
              </div>
            </div>
          </div>
             <!-- Gráfico de líneas de incidencias -->
          <canvas ref="lineChart" height="100"></canvas>
        </div>
      </div>
       <!-- Tarjeta de estadísticas -->
      <div class="card glassmorphic-card">
        <div class="card-body">
          <h5 class="card-title">Estadísticas</h5>
          <!-- Gráfico de barras de estadísticas -->
          <canvas ref="barChart"></canvas>
        </div>
      </div>
    </div>

    <!-- Right Column - Incidencias List -->
    <div class="col-md-8">
         <!-- Lista de incidencias -->
        <div class="incidencia-list glassmorphic-card">
           <!-- Encabezado de la lista de incidencias -->
            <div class="incidencia-list-header">
                <div>Incidencias</div>
                <!-- Leyenda de prioridades -->
                <div class="priority-legend">
                    <span><span class="priority-dot alta"></span> Alta</span>
                    <span><span class="priority-dot media"></span> Media</span>
                    <span><span class="priority-dot baja"></span> Baja</span>
                </div>
            </div>
            <!-- Mensaje de carga -->
            <div v-if="loading">Cargando incidencias...</div>
            <!-- Mensaje de error -->
            <div v-else-if="error">Error al cargar las incidencias.</div>
              <!-- Contenedor de la lista de incidencias -->
             <div v-else class="incidencias-container">
               <!-- Iteración sobre las incidencias filtradas -->
                <div v-for="incidencia in incidenciasPanelFiltradas" :key="incidencia.id" class="incidencia-item">
                  <!-- Marcador de prioridad -->
                    <div class="priority-marker" :class="incidencia.priority"></div>
                      <!-- Contenido de cada incidencia -->
                      <div class="incidencia-content">
                          <div class="incidencia-date">
                            <span>{{ incidencia.date }}</span>
                            <span>{{ incidencia.time }}</span>
                          </div>
                          <div class="incidencia-text">
                            <div>{{ incidencia.titulo }}</div>
                            <small class="text-muted">{{ incidencia.subtitulo }}</small>
                          </div>
                            <!-- Estado de la incidencia -->
                            <div class="incidencia-status-box">
                              <span class="incidencia-status" :class="incidencia.status.toLowerCase().replace(' ', '-')">
                                {{ incidencia.status }}
                              </span>
                            </div>
                      </div>
                   </div>
             </div>

        </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { Chart } from 'chart.js/auto';
import axios from 'axios';

// Referencias para los elementos del gráfico
const lineChart = ref(null);
const barChart = ref(null);

// Referencias para los datos de las incidencias
const incidencias = ref([]);
const incidenciasPanel = ref([]); // Referencia para las incidencias del panel
const loading = ref(true); // Control de carga
const error = ref(null); // Control de errores
const API_AUTH_URL = import.meta.env.VITE_API_AUTH_URL; // URL base de la API
const ALL_INCIDENCIAS_URL = `${API_AUTH_URL}/incidencia/all`; // URL para obtener todas las incidencias
const ALL_INCIDENCIAS_URL_CERRADAS = `${API_AUTH_URL}/incidencia/all_cerradas`; // URL para obtener las incidencias cerradas
const ME_URL = `${API_AUTH_URL}/auth/me`; // URL para obtener información del usuario
const userRole = ref(null); // Referencia para el rol del usuario
const userId = localStorage.getItem('id'); // Obtiene el ID del usuario desde el almacenamiento local

// Variables computadas para determinar el rol del usuario
const isTecnico = computed(() => userRole.value === 'tecnico');
const isAdmin = computed(() => userRole.value === 'administrador');


// Función para obtener la prioridad en formato de clase CSS
const obtenerPrioridad = (prioridad) => {
    if (prioridad === "alta" || prioridad ==="media" || prioridad === "baja") {
        return prioridad;
    } else {
        return "baja";
    }
};

// Función para obtener el estado en formato de texto
const obtenerEstado = (estado) => {
    switch (estado) {
        case 0:
            return 'Nueva';
        case 1:
            return 'Pendiente';
        case 2:
            return 'En curso';
        case 3:
            return 'Cerrada';
        case 4:
            return 'Mantenimiento';
        default:
            return '';
    }
};

// Función para formatear la fecha
const formatDate = (dateString) => {
    const date = new Date(dateString);
    const options = { year: 'numeric', month: 'long', day: 'numeric' };
    return date.toLocaleDateString(undefined, options);
};

// Función para formatear la hora
const formatTime = (dateString) => {
    const date = new Date(dateString);
    const options = { hour: '2-digit', minute: '2-digit' };
    return date.toLocaleTimeString(undefined, options);
};

// Filtra las incidencias para no mostrar las cerradas ni en mantenimiento y muestra solo 8 incidencias
const incidenciasFiltradas = computed(() => {
    return incidencias.value
        .slice(0, 8);
});
const incidenciasPanelFiltradas = computed(() => {
    return incidenciasPanel.value
        .filter(incidencia => {
            const estado = obtenerEstado(incidencia.estado);
            return estado !== 'Cerrada' && estado !== 'Mantenimiento';
        })
        .slice(0, 8);
});


// Cálculo del total de incidencias, incluyendo cerradas y en mantenimiento
const totalIncidenciasCount = computed(() => {
    return incidencias.value.length;
});

// Cálculo de incidencias abiertas hoy
const incidenciasAbiertasHoy = computed(() => {
    const today = new Date();
    return incidencias.value.filter(incidencia => {
        const fechaIncidencia = new Date(incidencia.fecha_apertura);
        return fechaIncidencia.getDate() === today.getDate() &&
            fechaIncidencia.getMonth() === today.getMonth() &&
            fechaIncidencia.getFullYear() === today.getFullYear() &&
            obtenerEstado(incidencia.estado) === 'Nueva';
    }).length;
});

// Cálculo de incidencias pendientes
const incidenciasPendientesCount = computed(() => {
    return incidencias.value.filter(incidencia => {
        const estado = obtenerEstado(incidencia.estado);
        return estado === 'Pendiente' || estado === 'Nueva';
    }).length;
});

// Cálculo de incidencias en curso
const incidenciasEnCursoCount = computed(() => {
    return incidencias.value.filter(incidencia => obtenerEstado(incidencia.estado) === 'En curso').length;
});

// Cálculo de incidencias cerradas
const incidenciasCerradasCount = computed(() => {
    return incidencias.value.filter(incidencia => obtenerEstado(incidencia.estado) === 'Cerrada').length;
});


// Función para generar datos para el gráfico de barras
const generateChartData = (incidencias) => {
    const today = new Date();
    const labels = [];
    const resolvedData = [];
    const inProgressData = [];

    for (let i = 6; i >= 0; i--) {
        const date = new Date(today);
        date.setDate(today.getDate() - i);
        const day = String(date.getDate()).padStart(2, '0');
        const month = String(date.getMonth() + 1).padStart(2, '0');
        const formattedDate = `${day}/${month}`;
        labels.push(formattedDate);

        const resolved = incidencias.filter(incidencia => {
            if (obtenerEstado(incidencia.estado) === 'Cerrada' && incidencia.fecha_cierre) {
                const fechaCierre = new Date(incidencia.fecha_cierre);
                return fechaCierre.getDate() === date.getDate() &&
                    fechaCierre.getMonth() === date.getMonth() &&
                    fechaCierre.getFullYear() === date.getFullYear()
            }
            return false;

        }).length;

        const inProgress = incidencias.filter(incidencia => {
            const fechaIncidencia = new Date(incidencia.fecha_apertura);
            return fechaIncidencia.getDate() === date.getDate() &&
                fechaIncidencia.getMonth() === date.getMonth() &&
                fechaIncidencia.getFullYear() === date.getFullYear() &&
                (obtenerEstado(incidencia.estado) === 'En curso' || obtenerEstado(incidencia.estado) === 'Pendiente' || obtenerEstado(incidencia.estado) === 'Nueva')
        }).length;
        resolvedData.push(resolved);
        inProgressData.push(inProgress)


    }
    return { labels: labels, resolved: resolvedData, opened: inProgressData };
};


// Función para generar datos para el gráfico de líneas
const generateLineChartData = (incidencias) => {
    const today = new Date();
    const hourlyData = Array(24).fill(0); // Inicializa un array con 24 elementos para almacenar los datos horarios

    incidencias.forEach(incidencia => {
        const fechaIncidencia = new Date(incidencia.fecha_apertura);
        // Comprueba si la incidencia fue creada hoy
        if (fechaIncidencia.getDate() === today.getDate() &&
            fechaIncidencia.getMonth() === today.getMonth() &&
            fechaIncidencia.getFullYear() === today.getFullYear()) {
            const hour = fechaIncidencia.getHours();
            hourlyData[hour]++; // Si es hoy, incrementa el conteo en esta hora
        }

    });

    return hourlyData.slice(0, 24) // Devuelve los primeros 24 elementos del array en caso de que haya incidencias con datos incorrectos
};

// Función para obtener los datos del usuario (rol)
const fetchUserData = async () => {
    const token = localStorage.getItem('token');
    if (token) {
        try {
            const response = await axios.get(ME_URL, {
                headers: {
                    Authorization: `Bearer ${token}`,
                },
            });
            userRole.value = response.data.rol;

        } catch (error) {
            console.error('Error fetching user data:', error);
        }
    }
};

// Función que verifica si el usuario tiene el rol de técnico o administrador
const checkUserHasReclamada = () => {
    if (isTecnico.value || isAdmin.value) {
    }
}

// Ciclo de vida del componente: onMounted (cuando el componente se monta)
onMounted(async () => {
    try {
        // Obtiene la información del usuario (rol)
        await fetchUserData()
        const token = localStorage.getItem('token');
        if (token) {
            try {
                // Petición para las incidencias del panel (all)
                const responsePanel = await axios.post(ALL_INCIDENCIAS_URL, {}, {
                    headers: {
                        Authorization: `Bearer ${token}`
                    }
                });
                // Mapea los datos de las incidencias del panel
                incidenciasPanel.value = responsePanel.data.map(incidencia => ({
                    ...incidencia,
                    priority: obtenerPrioridad(incidencia.prioridad),
                    status: obtenerEstado(incidencia.estado),
                    date: formatDate(incidencia.fecha_apertura),
                    time: formatTime(incidencia.fecha_apertura),
                    descripcion: incidencia.descripcion,
                    titulo: incidencia.titulo,
                    subtitulo: incidencia.subtitulo,
                    fecha_cierre: incidencia.fecha_cierre,
                    id_tecnico: incidencia.id_mantenimiento
                }));
                // Petición para las incidencias del dashboard (all_cerradas)
                const responseDashboard = await axios.post(ALL_INCIDENCIAS_URL_CERRADAS,{}, {
                    headers: {
                        Authorization: `Bearer ${token}`
                    }
                });
                 // Mapea los datos de las incidencias del dashboard
                incidencias.value = responseDashboard.data.map(incidencia => ({
                    ...incidencia,
                    priority: obtenerPrioridad(incidencia.prioridad),
                    status: obtenerEstado(incidencia.estado),
                    date: formatDate(incidencia.fecha_apertura),
                    time: formatTime(incidencia.fecha_apertura),
                    descripcion: incidencia.descripcion,
                    titulo: incidencia.titulo,
                    subtitulo: incidencia.subtitulo,
                    fecha_cierre: incidencia.fecha_cierre,
                    id_tecnico: incidencia.id_mantenimiento
                }));

                // Genera los datos para los gráficos
                const chartData = generateChartData(incidencias.value);
                const lineChartData = generateLineChartData(incidencias.value);

                 // Crea el gráfico de líneas para las incidencias por hora
                new Chart(lineChart.value, {
                    type: 'line',
                    data: {
                        labels: ['0:00', '1:00', '2:00', '3:00', '4:00', '5:00', '6:00', '7:00', '8:00', '9:00', '10:00', '11:00', '12:00', '13:00', '14:00', '15:00', '16:00', '17:00', '18:00', '19:00', '20:00', '21:00', '22:00', '23:00'],
                        datasets: [{
                            label: 'Incidencias',
                            data: lineChartData,
                            borderColor: '#600484',
                            backgroundColor: 'rgba(139, 92, 246, 0.1)',
                            tension: 0.4,
                            fill: true,
                            borderWidth: 2
                        }]
                    },
                     // Opciones del gráfico de líneas
                    options: {
                        responsive: true,
                        maintainAspectRatio: false,
                        plugins: {
                            legend: {
                                display: false
                            }
                        },
                        scales: {
                            y: {
                                beginAtZero: true,
                                grid: {
                                    color: 'rgba(116, 116, 116, 1)',
                                    drawBorder: false
                                },
                                ticks: {
                                    color: 'rgba(116, 116, 116, 1)',
                                    padding: 10
                                }
                            },
                            x: {
                                grid: {
                                    display: false
                                },
                                ticks: {
                                    color: 'rgba(116, 116, 116, 1)',
                                    padding: 10
                                }
                            }
                        }
                    }
                });

                 // Crea el gráfico de barras para las estadísticas de incidencias resueltas y abiertas
                new Chart(barChart.value, {
                    type: 'bar',
                    data: {
                        labels: chartData.labels,
                        datasets: [
                            {
                                label: 'Promedio incidencias resueltas',
                                data: chartData.resolved,
                                backgroundColor: '#600484',
                                borderRadius: 10,
                                stack: 'combined',
                            },
                            {
                                label: 'Promedio incidencias abiertas',
                                data: chartData.opened,
                                backgroundColor: '#a470c2',
                                borderRadius: 10,
                                stack: 'combined',
                            }
                        ]
                    },
                    // Opciones del gráfico de barras
                    options: {
                        responsive: true,
                        scales: {
                            y: {
                                beginAtZero: true,
                                max: 40,
                                ticks: {
                                    stepSize: 10,
                                }
                            }
                        },
                        plugins: {
                            legend: {
                                display: true,
                            },
                            roundedBars: true
                        },
                    }
                });
            } catch (err) {
                error.value = err;
            } finally {
                loading.value = false;
            }
            checkUserHasReclamada();
        }
    } catch (err) {
        error.value = err;
        loading.value = false;
    }
});

// Observador para cuando cambian las incidencias
watch(incidencias, () => {
    checkUserHasReclamada();
});
</script>

<style lang="scss" scoped>
// Definición de colores
$color-white: #fff;
$color-light-gray: #f8f8f8;
$color-gray: #ddd;
$color-dark-gray: #666;
$color-shadow: rgba(0, 0, 0, 0.1);
$color-shadow-hover: rgba(0, 0, 0, 0.2);
$color-high-priority: #FF5252;
$color-medium-priority: #FFCA28;
$color-low-priority: #4CAF50;
$color-new: #B89B00;
$color-pending: #B89B00;
$color-maintenance: #000000;
$color-closed: #000000;
$color-in-progress: #600484;
$color-text-muted: rgba(54, 54, 54, 0.6);

.incidencia-panel {
  width: 100%;
  padding: 20px;
  background-color: $color-light-gray;
  border-radius: 10px;
  box-shadow: 0 2px 4px $color-shadow;
}

.incidencia-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
}

.create-incidencia-btn {
  background-color: $color-white;
  border: 1px solid $color-gray;
  border-radius: 5px;
  padding: 12px 20px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;

  &:hover {
    background-color: #f0f0f0;
  }
}

.plus-icon {
  font-size: 1.5rem;
  margin-top: -4px;
}

.incidencia-stats {
  display: flex;
  gap: 10px;
}

.incidencia-stat-box {
  background-color: $color-white;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 15px;
  border-radius: 5px;
  min-width: 140px;
  gap: 5px;
  box-shadow: 0 2px 4px $color-shadow;

  &.active {
    background-color: #e3fae8;
    box-shadow: 0 0 10px $color-shadow-hover;
  }
}

.title {
  font-size: 0.8rem;
}

.count {
  font-size: 1.4rem;
  font-weight: bold;
}

.icon {
  font-size: 1.2rem;
}

.incidencia-list {
  background-color: rgba($color-white, 0.7) !important;
  border-radius: 5px;
  padding: 20px;
}

.incidencia-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid rgba(0, 0, 0, 0.1);
  padding: 0 10px 15px 10px;
  margin-bottom: 15px;
  font-size: 0.8rem;
  color: $color-dark-gray;
}

.priority-legend {
  display: flex;
  gap: 20px;
}

.priority-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  margin-right: 2px;
}

.alta {
  background-color: $color-high-priority;
}

.media {
  background-color: $color-medium-priority;
}

.baja {
  background-color: $color-low-priority;
}

.incidencia-item {
  display: flex;
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 8px;
  transition: all 0.2s ease;
  gap: 15px;
  align-items: flex-start;

    &:hover {
        background-color: rgba($color-white, 0.1);
    }
}

.incidencia-date {
  display: flex;
  flex-direction: column;
  font-size: 0.8rem;
  color: $color-dark-gray;
  min-width: 100px;
  flex-shrink: 0;

  span:first-child {
    font-weight: 500;
  }
}

.incidencia-content {
  display: flex;
  align-items: flex-start;
  gap: 20px;
  flex: 1;
    width: 100%;
}

.priority-marker {
  width: 5px;
  height: 40px;
  border-radius: 2px;
  flex-shrink: 0;
  margin-right: 5px;
}

.incidencia-text {
  display: flex;
  flex-direction: column;
  font-size: 0.9rem;
  line-height: 1.3;
  flex: 1;
  margin-left: 10px;
}

.incidencia-status-box {
  display: flex;
  align-items: center;
    justify-content: flex-end;
}

.incidencia-status {
  padding: 6px 10px;
  border-radius: 20px;
  font-size: 0.8rem;
  font-weight: 500;
}

.nueva {
  background-color: rgba($color-new, 0.17);
  color: $color-new;
}

.pendiente {
  background-color: rgba($color-pending, 0.17);
  color: $color-pending;
}

.mantenimiento {
  background-color: rgba($color-maintenance, 0.17);
  color: $color-maintenance;
}

.cerrada {
  background-color: rgba($color-closed, 0.17);
  color: $color-closed;
}

.en-curso {
  background-color: rgba($color-in-progress, 0.17);
  color: $color-in-progress;
}

.create-incidencia-card {
  cursor: pointer;
  transition: all 0.3s;

  &:hover {
    transform: scale(1.02);
    box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.7);
  }
}

.card-body-same-height {
  min-height: 100px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.glassmorphic-card {
  background: rgba($color-white, 0.7) !important;
  backdrop-filter: blur(8px);
  border: 1px solid rgba($color-white, 0.05) !important;
  box-shadow: 0 4px 24px 0 rgba(0, 0, 0, 0.3);
}

.pendiente-h2 {
  color: $color-pending;
  font-size: 70px;
}

.curso-h2 {
  color: $color-in-progress;
  font-size: 70px;
}

.cerrado-h2 {
  color: $color-maintenance;
  font-size: 70px;
}
.total-h2{
    color: $color-white;
    font-size: 70px;
}
.text-muted {
  color: $color-text-muted !important;
}
.h3 {
  color: rgba($color-white, 0.9);
}
table {
    background-color: transparent !important;
}

.table-responsive {
    max-height: calc(100vh - 300px);
    overflow-y: auto;
    background-color: transparent !important;
}

.table > :not(caption) > * > * {
    background-color: transparent !important;
}
canvas {
    max-height: 300px;
}
.status-dot {
    display: inline-block;
    width: 10px;
    height: 10px;
    border-radius: 50%;
}
.card.h-100 {
    height: calc(100vh - 200px) !important;
}

.card-body {
    padding: 1.5rem;
    display: flex;
    flex-direction: column;
}
.card-bodyTotal {
    padding: 1.8rem;
    display: flex;
    flex-direction: column;
    justify-content: center;
}
.incidencias-container {
    max-height: 600px;
    overflow-y: auto;
    padding: 0 10px;
}

/* Add smooth scrollbar for the tickets container */
.incidencias-container::-webkit-scrollbar {
    width: 8px;
}

.incidencias-container::-webkit-scrollbar-track {
    background: rgba($color-white, 0.1);
    border-radius: 4px;
}

.incidencias-container::-webkit-scrollbar-thumb {
    background: rgba(0, 0, 0, 0.2);
    border-radius: 4px;
}

.incidencias-container::-webkit-scrollbar-thumb:hover {
    background: rgba(0, 0, 0, 0.3);
}



.text-sm {
    font-size: 0.875rem;
    color: #000000;
}

.btn-ver-todos {
    background: transparent;
    border: none;
    color: #000000;
    font-size: 0.875rem;
    display: flex;
    align-items: center;
    padding: 0;
}

.incidencias-list {
    margin-top: 1rem;
}

.incidencia-row {
    display: flex;
    align-items: flex-start;
    margin-bottom: 0.5rem;
    min-height: 60px;
    width: 100%;
}

.incidencia-priority-bar {
    width: 5px;
    height: 40px;
    border-radius: 2px;
    flex-shrink: 0;
    margin-right: 5px;
    &.alta {
      background-color: $color-high-priority;
  }
  
  &.media {
    background-color: $color-medium-priority;
  }
  
  &.baja {
    background-color: $color-low-priority;
  }
}

.incidencia-content {
    display: flex;
    align-items: flex-start;
    gap: 20px;
    flex: 1;
        width: 100%;
}

.incidencia-date {
    display: flex;
    flex-direction: column;
    font-size: 0.8rem;
    color: $color-dark-gray;
    min-width: 100px;
    flex-shrink: 0;
}

.incidencia-date span:first-child {
    font-weight: 500;
}

.incidencia-description {
    display: flex;
    flex-direction: column;
    font-size: 0.9rem;
    line-height: 1.3;
    flex: 1;
    margin-left: 10px;
}

.incidencia-title {
    color: #000000;
    margin-bottom: 0.25rem;
}


.incidencia-subtitle {
    color: #747474;
    font-size: 0.875rem;
}
.incidencia-status-box {
    display: flex;
    align-items: center;
    justify-content: flex-end;
}

.incidencia-status {
    padding: 6px 10px;
    border-radius: 20px;
    font-size: 0.8rem;
    font-weight: 500;
}

@media (max-width: 768px) {
  .row {
    --bs-gutter-x: 0.5rem !important;
  }

  .col-sm-6 {
    flex: 0 0 auto;
    width: 50% !important;
    max-width: 50% !important;
  }

  .col-md-4,
  .col-md-8 {
    width: 100% !important;
    max-width: 100% !important;
    flex: 0 0 100% !important;
  }

  .card-body {
    padding: 1rem;
  }
  .card-bodyTotal {
    padding: 1rem;
  }
  .pendiente-h2,
  .curso-h2,
  .cerrado-h2,
    .total-h2{
    font-size: 40px;
  }

  .incidencia-list {
    padding: 10px;
  }

  .incidencia-list-header {
    font-size: 0.8rem;
    padding: 0 5px 10px 5px;
    margin-bottom: 10px;
  }

  .incidencia-item {
    padding: 10px;
    margin-bottom: 5px;
    gap: 10px;
  }

  .incidencia-date {
      min-width: 70px;
    font-size: 0.7rem;
  }

  .incidencia-content {
    gap: 10px;
  }

  .priority-marker {
    height: 30px;
    margin-right: 3px;
  }

  .incidencia-text {
    font-size: 0.7rem;
    margin-left: 5px;
  }

  .incidencia-status {
    padding: 4px 8px;
    font-size: 0.7rem;
  }
    .incidencias-container{
        padding: 0 5px;
    }

  .priority-legend {
    font-size: 0.7rem;
  }
  .card-body {
    padding: 10px;
  }
    .incidenciasabiertos {
        font-size: 0.8rem;
    }
    .abiertos-h3 {
        font-size: 1.3rem;
    }
}
</style>