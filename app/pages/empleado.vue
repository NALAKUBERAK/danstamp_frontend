<script setup lang="ts">
interface Pedido {
  id_pedido: number
  fecha_pedido: string
  fecha_entrega_estimada: string | null
  direccion_entrega: string
  total: string
  estado: string
  cliente: {
    nombres: string
    apellidos: string
  }
}

interface UsuarioSesion {
  fullName?: string
  role?: string
}

const config = useRuntimeConfig()
const usuario = ref<UsuarioSesion | null>(null)
const pedidos = ref<Pedido[]>([])
const cargando = ref(true)
const errorCarga = ref('')
const filtro = ref('todos')

const pedidosFiltrados = computed(() => {
  if (filtro.value === 'todos') return pedidos.value
  return pedidos.value.filter((pedido) => pedido.estado === filtro.value)
})

const pendientes = computed(() => pedidos.value.filter((pedido) => pedido.estado === 'pendiente').length)
const enProceso = computed(() => pedidos.value.filter((pedido) => pedido.estado === 'en_proceso').length)
const cotizaciones = computed(() => pedidos.value.filter((pedido) => pedido.estado.startsWith('cotizacion')).length)

const etiquetasEstado: Record<string, string> = {
  pendiente: 'Pendiente',
  en_proceso: 'En proceso',
  terminado: 'Terminado',
  entregado: 'Entregado',
  cancelado: 'Cancelado',
  cotizacion: 'Cotización',
  cotizacion_enviada: 'Cotización enviada',
}

onMounted(async () => {
  const token = localStorage.getItem('authToken')
  const storedUser = localStorage.getItem('authUser')

  if (!token || !storedUser) {
    await navigateTo('/login', { replace: true })
    return
  }

  try {
    usuario.value = JSON.parse(storedUser) as UsuarioSesion
  } catch {
    localStorage.removeItem('authToken')
    localStorage.removeItem('authUser')
    await navigateTo('/login', { replace: true })
    return
  }

  const role = usuario.value?.role?.toLowerCase()
  if (role !== 'empleado') {
    await navigateTo(role === 'admin' ? '/admin' : '/principal', { replace: true })
    return
  }

  try {
    pedidos.value = await $fetch<Pedido[]>('/pedidos', {
      baseURL: config.public.apiBase,
      headers: { Authorization: `Bearer ${token}` },
    })
  } catch (error: unknown) {
    const apiError = error as { statusCode?: number; data?: { message?: string | string[] } }
    if (apiError.statusCode === 401 || apiError.statusCode === 403) {
      localStorage.removeItem('authToken')
      localStorage.removeItem('authUser')
      await navigateTo('/login', { replace: true })
      return
    }
    const apiMessage = apiError.data?.message
    errorCarga.value = Array.isArray(apiMessage)
      ? apiMessage.join('. ')
      : apiMessage || 'No se pudieron cargar tus pedidos.'
  } finally {
    cargando.value = false
  }
})

function nombreEstado(estado: string): string {
  return etiquetasEstado[estado] || estado.replaceAll('_', ' ')
}

function formatoMoneda(valor: string): string {
  return Number(valor).toLocaleString('es-CO', {
    style: 'currency',
    currency: 'COP',
    maximumFractionDigits: 0,
  })
}

function formatoFecha(fecha: string | null): string {
  if (!fecha) return 'Sin fecha'
  return new Date(`${fecha.slice(0, 10)}T00:00:00`).toLocaleDateString('es-CO', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  })
}

function cerrarSesion() {
  localStorage.removeItem('authToken')
  localStorage.removeItem('authUser')
  navigateTo('/login', { replace: true })
}
</script>

<template>
  <main class="empleado-page">
    <header class="empleado-topbar">
      <NuxtLink class="empleado-marca" to="/principal" aria-label="Ir al inicio">
        <BrandLogo size="medium" />
      </NuxtLink>
      <div class="empleado-perfil">
        <span>{{ usuario?.fullName || 'Trabajador' }}</span>
        <span class="rol-badge">TRABAJADOR</span>
        <button type="button" @click="cerrarSesion">Cerrar sesión</button>
      </div>
    </header>

    <section class="empleado-contenido">
      <div class="empleado-intro">
        <p class="empleado-eyebrow">ESPACIO DE TRABAJO</p>
        <h1>Hola, {{ usuario?.fullName?.split(' ')[0] || 'equipo' }}.</h1>
        <p>Consulta tus pedidos asignados y solicitudes de cotización.</p>
      </div>

      <p v-if="errorCarga" class="empleado-error" role="alert">{{ errorCarga }}</p>
      <div v-if="cargando" class="empleado-cargando">Cargando tus pedidos...</div>

      <template v-else-if="!errorCarga">
        <section class="empleado-resumen" aria-label="Resumen de pedidos">
          <article><span class="resumen-icono">▤</span><div><small>PEDIDOS ASIGNADOS</small><strong>{{ pedidos.length }}</strong></div></article>
          <article><span class="resumen-icono">◷</span><div><small>PENDIENTES</small><strong>{{ pendientes }}</strong></div></article>
          <article><span class="resumen-icono">↻</span><div><small>EN PROCESO</small><strong>{{ enProceso }}</strong></div></article>
          <article><span class="resumen-icono">✦</span><div><small>COTIZACIONES</small><strong>{{ cotizaciones }}</strong></div></article>
        </section>

        <section class="empleado-pedidos">
          <header class="pedidos-heading">
            <div><p class="empleado-eyebrow">SEGUIMIENTO</p><h2>Mis pedidos</h2></div>
            <label>Estado
              <select v-model="filtro">
                <option value="todos">Todos</option>
                <option value="pendiente">Pendientes</option>
                <option value="en_proceso">En proceso</option>
                <option value="terminado">Terminados</option>
                <option value="cotizacion">Cotizaciones</option>
              </select>
            </label>
          </header>

          <div v-if="!pedidosFiltrados.length" class="empleado-vacio">
            No hay pedidos para mostrar con este filtro.
          </div>
          <div v-else class="tabla-scroll">
            <table class="tabla-pedidos">
              <thead>
                <tr><th>Pedido</th><th>Cliente</th><th>Fecha</th><th>Entrega estimada</th><th>Total</th><th>Estado</th></tr>
              </thead>
              <tbody>
                <tr v-for="pedido in pedidosFiltrados" :key="pedido.id_pedido">
                  <td>#{{ pedido.id_pedido }}</td>
                  <td>{{ pedido.cliente.nombres }} {{ pedido.cliente.apellidos }}</td>
                  <td>{{ formatoFecha(pedido.fecha_pedido) }}</td>
                  <td>{{ formatoFecha(pedido.fecha_entrega_estimada) }}</td>
                  <td>{{ formatoMoneda(pedido.total) }}</td>
                  <td><span class="estado-pedido" :class="`estado-${pedido.estado}`">{{ nombreEstado(pedido.estado) }}</span></td>
                </tr>
              </tbody>
            </table>
          </div>
        </section>
      </template>
    </section>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Space+Grotesk:wght@500;600;700&display=swap');

.empleado-page { --empleado-rosa: #e56ddd; --empleado-lila: #f3c7ef; --empleado-ink: #28212d; min-height: 100vh; color: var(--empleado-ink); background: linear-gradient(125deg, #f8d7f0 0%, #e8d6f7 48%, #fff4f8 100%); font-family: 'DM Sans', Arial, sans-serif; }
.empleado-topbar { display: flex; justify-content: space-between; align-items: center; min-height: 64px; padding: 0 6vw; border-bottom: 1px solid rgba(255,255,255,.7); background: rgba(255,255,255,.65); }
.empleado-marca { display: flex; align-items: center; text-decoration: none; }
.empleado-perfil { display: flex; align-items: center; gap: 13px; font-size: 11px; }
.rol-badge { padding: 5px 8px; border-radius: 99px; color: #8e328f; background: #f4e3f5; font-size: 8px; font-weight: 700; letter-spacing: .06em; }
.empleado-perfil button { padding: 8px 12px; border: 1px solid #e6d9e8; border-radius: 99px; color: #55485b; background: white; cursor: pointer; font: inherit; font-size: 10px; }
.empleado-contenido { max-width: 1180px; margin: auto; padding: 48px 28px 60px; }
.empleado-intro { margin-bottom: 28px; }
.empleado-eyebrow { margin: 0 0 8px; color: #946c9a; font-size: 9px; font-weight: 700; letter-spacing: .14em; }
.empleado-intro h1 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: clamp(30px, 5vw, 46px); letter-spacing: -.06em; }
.empleado-intro > p:last-child { margin: 9px 0 0; color: #716575; font-size: 13px; }
.empleado-resumen { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 14px; margin-bottom: 26px; }
.empleado-resumen article { display: flex; align-items: center; gap: 14px; min-height: 84px; padding: 15px; border: 1px solid rgba(255,255,255,.8); border-radius: 14px; background: rgba(255,255,255,.78); box-shadow: 0 8px 25px rgba(106,61,110,.07); }
.resumen-icono { display: grid; place-items: center; flex: 0 0 43px; height: 43px; border-radius: 12px; color: #8e328f; background: #f3d9f3; font-size: 20px; }
.empleado-resumen small, .empleado-resumen strong { display: block; }
.empleado-resumen small { color: #887b8b; font-size: 8px; font-weight: 700; letter-spacing: .06em; }
.empleado-resumen strong { margin-top: 6px; font-family: 'Space Grotesk', Arial, sans-serif; font-size: 20px; }
.empleado-pedidos { padding: 22px; border: 1px solid rgba(255,255,255,.85); border-radius: 16px; background: rgba(255,255,255,.84); box-shadow: 0 12px 34px rgba(106,61,110,.08); }
.pedidos-heading { display: flex; justify-content: space-between; align-items: end; gap: 14px; margin-bottom: 18px; }
.pedidos-heading h2 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: 22px; letter-spacing: -.04em; }
.pedidos-heading label { display: grid; gap: 5px; color: #766b78; font-size: 9px; }
.pedidos-heading select { min-width: 145px; height: 34px; padding: 0 10px; border: 1px solid #e6d9e8; border-radius: 8px; color: var(--empleado-ink); background: white; font: inherit; font-size: 10px; }
.tabla-scroll { overflow-x: auto; }
.tabla-pedidos { width: 100%; border-collapse: collapse; text-align: left; }
.tabla-pedidos th, .tabla-pedidos td { padding: 12px 9px; border-top: 1px solid #eee5ef; font-size: 10px; white-space: nowrap; }
.tabla-pedidos th { color: #8b7c8e; font-size: 8px; letter-spacing: .05em; }
.tabla-pedidos td { color: #514653; }
.estado-pedido { display: inline-block; padding: 5px 8px; border-radius: 99px; color: #8e328f; background: #f4e5f5; font-size: 8px; font-weight: 700; }
.estado-entregado { color: #317b58; background: #e4f4e9; }
.estado-cancelado { color: #a44c5d; background: #fae9ec; }
.estado-en_proceso { color: #4a70a1; background: #e8f1fb; }
.empleado-vacio, .empleado-cargando, .empleado-error { padding: 32px 16px; border-radius: 10px; color: #796c7d; background: rgba(255,255,255,.75); font-size: 11px; text-align: center; }
.empleado-error { color: #9c2a52; background: #fff0f3; }

@media (max-width: 760px) { .empleado-resumen { grid-template-columns: repeat(2, minmax(0, 1fr)); }.empleado-contenido { padding: 34px 18px 44px; }.empleado-pedidos { padding: 17px 13px; } }
@media (max-width: 480px) { .empleado-topbar { align-items: start; flex-direction: column; gap: 10px; padding: 12px 16px; }.empleado-perfil { width: 100%; justify-content: space-between; }.empleado-resumen article { gap: 9px; padding: 11px; }.resumen-icono { flex-basis: 35px; height: 35px; }.empleado-resumen small { font-size: 7px; }.empleado-resumen strong { font-size: 17px; }.pedidos-heading { align-items: start; flex-direction: column; }.pedidos-heading select { width: 100%; } }
</style>
