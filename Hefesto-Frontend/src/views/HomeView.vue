<template>
  <!-- Fondo animado con múltiples elementos span -->
  <div class="background">
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
    <span></span>
  </div>

  <!-- Contenedor principal del dashboard -->
  <div class="dashboard min-vh-100">
    <!-- Sidebar -->
    <div :class="['sidebar p-0 d-flex flex-column', { 'collapsed': isSidebarCollapsed }]">
      <!-- Logo del dashboard -->
      <div class="logo mb-5 p-4">
        <img :src="logoImg" alt="Logo" class="img-fluid mb-4" style="max-width: 210px;">
      </div>

      <!-- Navegación del sidebar -->
      <nav class="nav flex-column">
        <!-- Iteración sobre los elementos del menú filtrados por rol -->
        <a
          v-for="item in filteredMenuItems"
          :key="item.name"
          href="#"
          :class="['nav-link d-flex align-items-center', { active: activeItem === item.name }]"
          @click.prevent="setActiveItem(item.name)"
        >
          <img :src="item.icon" class="me-3" alt="" aria-hidden="true" />
          <span>{{ item.name }}</span>
        </a>
      </nav>

      <!-- Botón para cerrar sesión -->
      <button @click="logout" class="nav-link mt-auto d-flex align-items-center">
        <img :src="cerrarImg" class="me-3" alt="" aria-hidden="true" />
        <span>Cerrar sesión</span>
      </button>
    </div>

    <!-- Contenido principal -->
    <div class="main-content d-flex flex-column">
      <!-- Header del dashboard -->
      <header class="header px-4 py-3 d-flex justify-content-between align-items-center">
        <!-- Botón para colapsar el sidebar en dispositivos móviles -->
        <button @click="toggleSidebar" class="btn btn-link d-md-none text-white">
          <i class="bi bi-list fs-4"></i>
        </button>
        <h1 class="h4 mb-0"></h1>
        <!-- Información del usuario -->
        <div class="d-flex align-items-center gap-3">
          <div class="text-end">
            <span class="user-name">{{ userName }} {{ userLastName }}</span>
            <span class="user-role">{{ userRole }}</span>
          </div>
          <!-- Avatar del usuario -->
          <div class="user-avatar">
            <img :src="userImagePath" alt="User avatar" class="rounded-circle" v-if="userImagePath" />
            <div
              v-else
              class="rounded-circle bg-secondary d-flex justify-content-center align-items-center"
              style="width: 40px; height: 40px;"
            >
              <i class="bi bi-person-circle text-white" style="font-size: 1.5rem;"></i>
            </div>
          </div>
        </div>
      </header>

      <!-- Área de contenido dinámico -->
      <div class="content flex-grow-1 d-flex flex-column">
        <div class="content-panel p-4 d-flex flex-column">
          <!-- Componente dinámico basado en el ítem activo -->
          <component :is="componenteActual" v-if="componenteActual" />
        </div>
      </div>
    </div>

    <!-- Overlay para móviles -->
    <div
      class="sidebar-overlay d-md-none"
      :class="{ 'active': isSidebarCollapsed }"
      @click="toggleSidebar"
    ></div>
  </div>
</template>

<script setup>
import { ref, shallowRef, defineAsyncComponent, onMounted, computed, watch } from 'vue'; // Import shallowRef
import { useRouter } from 'vue-router';
import axios from 'axios';

// Importa las imágenes
import logoImg from '../assets/images/icons/logos.png';
import ticketsIcon from '../assets/images/icons/tickets.svg';  // Ruta corregida: ../assets/...
import mantenimientoIcon from '../assets/images/icons/mantenimiento.svg';
import ajustesIcon from '../assets/images/icons/ajustes.svg';
import administracionIcon from '../assets/images/icons/administracion.svg';
import panelIcon from '../assets/images/icons/panel.svg';
import cerrarImg from '../assets/images/icons/cerrar.svg'


// Uso del router de Vue
const router = useRouter();

// Referencias reactivas para los datos del usuario
const userRole = ref(null);
const userName = ref(null);
const userLastName = ref(null);
const userPicture = ref(null);
const isLoggingOut = ref(false);
const activeItem = ref('Panel');
const componenteActual = shallowRef(null); // Use shallowRef here

// URLs de la API
const API_AUTH_URL = import.meta.env.VITE_API_AUTH_URL;
const ME_URL = `${API_AUTH_URL}/auth/me`;
const LOGOUT_URL = `${API_AUTH_URL}/auth/logout`;
const IMAGE_URL = import.meta.env.VITE_IMAGE_URL;

// Componentes asíncronos para el contenido dinámico
const Panel = defineAsyncComponent(() => import('../components/Panel.vue'));
const Incidencias = defineAsyncComponent(() => import('../components/Incidencias.vue'));
const Mantenimiento = defineAsyncComponent(() => import('../components/Mantenimiento.vue'));
const Ajustes = defineAsyncComponent(() => import('../components/Ajustes.vue'));
const Administracion = defineAsyncComponent(() => import('../components/Administracion.vue'));

// Elementos base del menú
const baseMenuItems = [
  { name: 'Incidencias', icon: ticketsIcon, to: '/incidencias' },
  { name: 'Mantenimiento', icon: mantenimientoIcon, to: '/mantenimiento' },
  { name: 'Ajustes', icon: ajustesIcon, to: '/ajustes' },
];

// Elementos del menú filtrados por rol
const allMenuItems = {
  administrador: [
    { name: 'Panel', icon: panelIcon, to: '/dashboard' },
    ...baseMenuItems,
    { name: 'Administracion', icon: administracionIcon, to: '/administracion' },
  ],
  operario: [
    { name: 'Panel', icon: panelIcon, to: '/dashboard' },
    { name: 'Incidencias', icon: ticketsIcon, to: '/incidencias' },
    { name: 'Ajustes', icon: ajustesIcon, to: '/ajustes' },
  ],
  tecnico: [
    { name: 'Panel', icon: panelIcon, to: '/dashboard' },
    { name: 'Incidencias', icon: ticketsIcon, to: '/incidencias' },
    { name: 'Mantenimiento', icon: mantenimientoIcon, to: '/mantenimiento' },
    { name: 'Ajustes', icon: ajustesIcon, to: '/ajustes' },
  ],
};

// Computed property para filtrar los ítems del menú según el rol del usuario
const filteredMenuItems = computed(() => {
  return allMenuItems[userRole.value] || [];
});

// Watch activeItem and update componenteActual
watch(activeItem, (newItem) => {
  switch (newItem) {
    case 'Panel':
      componenteActual.value = Panel;
      break;
    case 'Incidencias':
      componenteActual.value = Incidencias;
      break;
    case 'Mantenimiento':
      componenteActual.value = Mantenimiento;
      break;
    case 'Ajustes':
      componenteActual.value = Ajustes;
      break;
    case 'Administracion':
      componenteActual.value = Administracion;
      break;
    default:
      componenteActual.value = Panel; // Default to Panel
      break;
  }
});


// Función para establecer el ítem activo
const setActiveItem = (itemName) => {
  activeItem.value = itemName;
  console.log('Active item:', itemName);
};

// Computed property para construir la ruta de la imagen del usuario
const userImagePath = computed(() => {
  return userPicture.value ? `${IMAGE_URL}${userPicture.value}` : null;
});

// Hook de ciclo de vida: cuando el componente es montado
onMounted(async () => {
  const token = localStorage.getItem('token');
  if (token) {
    try {
      const response = await axios.get(ME_URL, {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      });
      userRole.value = response.data.rol;
      userName.value = response.data.name;
      userLastName.value = response.data.primer_apellido;
      userPicture.value = response.data.foto_perfil;

      // Set initial component after user role is loaded
      if (userRole.value === 'operario') {
        activeItem.value = 'Incidencias';
        componenteActual.value = Incidencias;
      } else {
        activeItem.value = 'Panel'; // Default to 'Panel' for other roles
        componenteActual.value = Panel;
      }

    } catch (error) {
      console.error('Error fetching user data:', error);
      router.push('/');
    }
  } else {
    router.push('/');
  }
});

// Función para cerrar sesión
const logout = async () => {
  isLoggingOut.value = true;
  const token = localStorage.getItem('token');
  if (token) {
    try {
      await axios.get(LOGOUT_URL, {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      });
      localStorage.removeItem('token');
      localStorage.removeItem('token_type');
      router.push('/');
      console.log('Sesión cerrada correctamente');
    } catch (error) {
      console.error('Error al cerrar sesión:', error);
    } finally {
      isLoggingOut.value = false;
    }
  } else {
    console.warn('No token found in localStorage.');
    router.push('/');
    isLoggingOut.value = false;
  }
};

// Referencia reactiva para el estado colapsado del sidebar
const isSidebarCollapsed = ref(false);

// Función para alternar el estado colapsado del sidebar
const toggleSidebar = () => {
  isSidebarCollapsed.value = !isSidebarCollapsed.value;
};
</script>

<style lang="scss" scoped>
$white-01: rgba(255, 255, 255, 0.1);
$white-03: rgba(255, 255, 255, 0.3);
$white-05: rgba(255, 255, 255, 0.5);
$white-06: rgba(255, 255, 255, 0.6);
$white-07: rgba(255, 255, 255, 0.7);
$white-08: rgba(255, 255, 255, 0.8);
$white-09: rgba(255, 255, 255, 0.9);

.dashboard {
  display: grid;
  grid-template-columns: 280px 1fr;
  position: relative;
  overflow-x: hidden;

  @media (max-width: 767px) {
    grid-template-columns: 1fr;
  }
}

/* Estilos del sidebar con efecto glassmorphism */
.sidebar {
  background: $white-05;
  backdrop-filter: blur(10px);
  border-right: 1px solid $white-01;
  height: 100%;
  position: sticky;
  top: 0;
  z-index: 1000;
  width: 280px;
  transition: transform 0.3s ease;

  &.collapsed {
    @media (min-width: 768px) {
      width: 80px;

      .nav-link {
        justify-content: center;
        padding: 0.8rem 0;
      }

      .logo img {
        max-width: 50px;
      }
    }

    @media (max-width: 767px) {
      transform: translateX(0);
    }
  }

  @media (max-width: 767px) {
    position: fixed;
    transform: translateX(-100%);
    width: 280px;
  }
}

.nav-link {
  color: $white-07;
  padding: 0.8rem 1rem;
  margin-bottom: 0.5rem;
  border-radius: 8px;
  transition: all 0.3s;
  font-size: 0.95rem;
  text-decoration: none;

  &:hover,
  &.active {
    background: $white-01;
    color: white;
  }

  img {
    width: 24px;
    height: 24px;
    margin-right: 10px;
    vertical-align: middle;
  }
}

/* Estilos del header */
.header {
  background: $white-05;
  backdrop-filter: blur(10px);
  color: white;
}

/* Estilos del contenido principal */
.main-content {
  min-height: 100vh;
  width: 100%;

  @media (max-width: 767px) {
    margin-left: 0;
    width: 100vw;
    overflow-x: hidden;
  }
}

/* Estilos del panel de contenido */
.content-panel {
  background: $white-03;
  backdrop-filter: blur(10px);
  border: 1px solid $white-01;
  flex-grow: 1;
}

/* Estilos del overlay para móviles */
.sidebar-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 999;
  opacity: 0;
  visibility: hidden;
  transition: all 0.3s ease;

  &.active {
    opacity: 1;
    visibility: visible;
  }
}

.logo img {
  max-width: 100px;
  transition: all 0.3s ease;
}

.text-end {
  display: flex;
  flex-direction: column;
}

.user-name {
  font-size: 0.9rem;
  color: $white-09;
}

.user-role {
  font-size: 0.8rem;
  color: $white-06;
}

.user-avatar {
  width: 40px;
  height: 40px;
  background: $white-01;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  color: $white-08;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    border-radius: 50%;
  }
}
</style>