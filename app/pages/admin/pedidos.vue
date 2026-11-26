<script setup lang="ts">
interface Pedido {
  id_pedido: number
  fecha_pedido: string
  cliente: { nombres: string; apellidos: string; email: string }
  total: string
  estado: string
}

interface UsuarioSesion {
  fullName?: string
  role?: string
}

const config = useRuntimeConfig()
const usuario = ref<UsuarioSesion | null>(null)
const autorizado = ref(false)
const pedidos = ref<Pedido[]>([])
const busqueda = ref('')
const pagina = ref(1)
const porPagina = 10
const cargando = ref(true)
const errorCarga = ref('')

const pedidosFiltrados = computed(() => {
  const consulta = busqueda.value.trim().toLocaleLowerCase()
  return pedidos.value.filter((pedido) => {
    const cliente = `${pedido.cliente.nombres} ${pedido.cliente.apellidos}`.toLocaleLowerCase()
    return !consulta
      || String(pedido.id_pedido).includes(consulta)
      || cliente.includes(consulta)
      || pedido.cliente.email.toLocaleLowerCase().includes(consulta)
      || pedido.estado.toLocaleLowerCase().includes(consulta)
  })
})
const totalPaginas = computed(() => Math.max(1, Math.ceil(pedidosFiltrados.value.length / porPagina)))
const pedidosPagina = computed(() => {
  const inicio = (pagina.value - 1) * porPagina
  return pedidosFiltrados.value.slice(inicio, inicio + porPagina)
})

watch(busqueda, () => {
  pagina.value = 1
})

onMounted(async () => {
  const storedToken = localStorage.getItem('authToken')
  const storedUser = localStorage.getItem('authUser')

  if (!storedToken || !storedUser) {
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
  if (role !== 'admin' && role !== 'empleado') {
    await navigateTo('/principal', { replace: true })
    return
  }

  autorizado.value = true

  try {
    pedidos.value = await $fetch<Pedido[]>('/pedidos/admin', {
      baseURL: config.public.apiBase,
      headers: { Authorization: `Bearer ${storedToken}` },
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
      : apiMessage || 'No se pudo cargar la lista de pedidos. Inténtalo de nuevo.'
  } finally {
    cargando.value = false
  }
})

function formatoFecha(fecha: string): string {
  return new Date(fecha).toLocaleDateString('es-CO', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  })
}

function formatoMoneda(valor: string): string {
  return Number(valor).toLocaleString('es-CO', {
    style: 'currency',
    currency: 'COP',
    maximumFractionDigits: 0,
  })
}

function etiquetaEstado(estado: string): string {
  const etiquetas: Record<string, string> = {
    pendiente: 'Pendiente',
    en_proceso: 'En proceso',
    terminado: 'Terminado',
    entregado: 'Entregado',
    cancelado: 'Cancelado',
    cotizacion: 'Cotización',
    cotizacion_enviada: 'Cotización enviada',
  }
  return etiquetas[estado] || estado
}

function cerrarSesion() {
  localStorage.removeItem('authToken')
  localStorage.removeItem('authUser')
  navigateTo('/login', { replace: true })
}
</script>

<template>
  <main v-if="autorizado" class="pedidos-page">
    <aside class="pedidos-sidebar">
      <NuxtLink class="pedidos-brand" to="/admin" aria-label="DAN’STAMP">
        <BrandLogo size="medium" />
      </NuxtLink>
      <div class="pedidos-admin">
        <span class="admin-avatar">{{ usuario?.fullName?.charAt(0) || 'A' }}</span>
        <span><strong>{{ usuario?.fullName || 'Administrador' }}</strong><small>{{ usuario?.role === 'empleado' ? 'Empleado' : 'Administrador' }}</small></span>
      </div>
      <nav class="pedidos-nav" aria-label="Navegación de administración">
        <NuxtLink class="nav-item" to="/admin"><span>▦</span> Dashboard</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/usuarios"><span>♙</span> Usuarios</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/clientes"><span>♧</span> Clientes</NuxtLink>
        <NuxtLink class="nav-item active" to="/admin/pedidos"><span>🛒</span> Pedidos</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/productos"><span>▣</span> Productos</NuxtLink>
      </nav>
      <button class="pedidos-logout" type="button" @click="cerrarSesion"><span>↪</span> Cerrar sesión</button>
    </aside>

    <section class="pedidos-main">
      <header class="pedidos-topbar"><span>☰</span><span aria-label="Perfil administrador">♙</span></header>
      <div class="pedidos-content">
        <div class="pedidos-breadcrumb">
          <div><span aria-hidden="true">🛒</span><strong>PEDIDOS</strong><BrandLogo size="small" /></div>
          <p>⌂ <span>/</span> Pedidos</p>
        </div>

        <header class="pedidos-heading">
          <div>
            <p class="pedidos-eyebrow">PANEL DE ADMINISTRACIÓN</p>
            <h1>Pedidos del sistema</h1>
            <p class="pedidos-description">Consulta los pedidos realizados por los clientes.</p>
          </div>
          <article class="pedidos-total" aria-label="Total de pedidos registrados">
            <span>Total de pedidos</span>
            <strong>{{ pedidos.length }}</strong>
          </article>
        </header>

        <p v-if="errorCarga" class="pedidos-error" role="alert">{{ errorCarga }}</p>
        <div v-else-if="cargando" class="pedidos-state">Cargando pedidos...</div>
        <section v-else class="pedidos-panel" aria-label="Lista de pedidos realizados">
          <div class="pedidos-toolbar">
            <p>Mostrando <strong>{{ pedidosFiltrados.length }}</strong> de <strong>{{ pedidos.length }}</strong> pedidos</p>
            <label class="pedidos-search">
              <span aria-hidden="true">⌕</span>
              <input v-model="busqueda" type="search" placeholder="Buscar pedido o cliente" aria-label="Buscar pedido o cliente" />
            </label>
          </div>

          <div class="pedidos-table-wrap">
            <table class="pedidos-table">
              <thead>
                <tr><th>ID</th><th>Cliente</th><th>Fecha</th><th>Total</th><th>Estado</th></tr>
              </thead>
              <tbody>
                <tr v-for="pedido in pedidosPagina" :key="pedido.id_pedido">
                  <td>{{ pedido.id_pedido }}</td>
                  <td>
                    <strong>{{ pedido.cliente.nombres }} {{ pedido.cliente.apellidos }}</strong>
                    <small>{{ pedido.cliente.email }}</small>
                  </td>
                  <td>{{ formatoFecha(pedido.fecha_pedido) }}</td>
                  <td>{{ formatoMoneda(pedido.total) }}</td>
                  <td><span class="estado-pedido" :class="`estado-${pedido.estado}`">{{ etiquetaEstado(pedido.estado) }}</span></td>
                </tr>
                <tr v-if="!pedidosFiltrados.length">
                  <td colspan="5" class="pedidos-vacio">{{ pedidos.length ? 'No hay pedidos que coincidan con la búsqueda.' : 'Todavía no hay pedidos registrados.' }}</td>
                </tr>
              </tbody>
            </table>
          </div>

          <footer class="pedidos-paginacion">
            <span>Página {{ pagina }} de {{ totalPaginas }}</span>
            <div>
              <button type="button" :disabled="pagina <= 1" @click="pagina--">Anterior</button>
              <button type="button" :disabled="pagina >= totalPaginas" @click="pagina++">Siguiente</button>
            </div>
          </footer>
        </section>
      </div>
    </section>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Space+Grotesk:wght@500;600;700&display=swap');

.pedidos-page { --pedidos-tinta: #28212d; min-height: 100vh; display: grid; grid-template-columns: 220px minmax(0, 1fr); color: var(--pedidos-tinta); background: linear-gradient(115deg, #f8e6f5 0%, #efdef8 52%, #fff4f8 100%); font-family: 'DM Sans', Arial, sans-serif; }
.pedidos-sidebar { position: sticky; top: 0; display: flex; flex-direction: column; min-height: 100vh; padding: 20px 14px; background: #f1c8ed; }
.pedidos-brand { display: flex; align-items: center; padding: 0 7px; text-decoration: none; }
.pedidos-admin { display: flex; align-items: center; gap: 9px; margin: 28px 0; padding: 8px 7px; }
.admin-avatar { display: grid; place-items: center; flex: 0 0 36px; height: 36px; border-radius: 50%; color: white; background: #a83caf; font-weight: 700; }
.pedidos-admin strong, .pedidos-admin small { display: block; max-width: 140px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.pedidos-admin strong { font-size: 10px; }
.pedidos-admin small { margin-top: 3px; color: #705873; font-size: 9px; }
.pedidos-nav { display: grid; gap: 6px; }
.nav-item, .pedidos-logout { display: flex; align-items: center; gap: 10px; min-height: 37px; padding: 0 9px; border: 0; border-radius: 8px; color: #46384a; background: transparent; font: inherit; font-size: 10px; text-align: left; text-decoration: none; }
.nav-item span, .pedidos-logout span { width: 16px; font-size: 15px; text-align: center; }
.nav-item:hover, .nav-item.active { color: #7d2f83; background: rgba(255,255,255,.62); font-weight: 700; }
.pedidos-logout { margin-top: auto; cursor: pointer; }
.pedidos-logout:hover { color: #7d2f83; background: rgba(255,255,255,.62); }
.pedidos-main { min-width: 0; }
.pedidos-topbar { display: flex; justify-content: space-between; align-items: center; height: 28px; padding: 0 15px; color: #342537; background: #e47ae3; font-size: 13px; }
.pedidos-content { padding: 0 28px 36px; }
.pedidos-breadcrumb { display: flex; justify-content: space-between; align-items: center; min-height: 48px; margin: 0 -28px 20px; padding: 0 28px; background: rgba(255,255,255,.94); }
.pedidos-breadcrumb > div { display: flex; align-items: center; gap: 9px; font-size: 10px; }
.pedidos-breadcrumb > div > span { font-size: 17px; }
.pedidos-breadcrumb p { color: #786b7b; font-size: 9px; }
.pedidos-breadcrumb p { margin: 0; }
.pedidos-breadcrumb p span { padding: 0 5px; }
.pedidos-heading { display: flex; justify-content: space-between; align-items: end; gap: 18px; margin: 28px 0 22px; }
.pedidos-eyebrow { margin: 0 0 7px; color: #96789b; font-size: 9px; font-weight: 700; letter-spacing: .14em; }
.pedidos-heading h1 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: clamp(24px, 3vw, 34px); letter-spacing: -.05em; }
.pedidos-description { margin: 7px 0 0; color: #887b8b; font-size: 12px; }
.pedidos-total { display: grid; gap: 6px; min-width: 125px; padding: 11px 14px; border: 1px solid #f0e5f1; border-radius: 10px; background: rgba(255,255,255,.8); }
.pedidos-total span { color: #786b7b; font-size: 9px; }
.pedidos-total strong { color: #8e328f; font-size: 19px; }
.pedidos-panel { overflow: hidden; border: 1px solid #eee4ef; border-radius: 12px; background: rgba(255,255,255,.96); box-shadow: 0 8px 24px rgba(106,61,110,.06); }
.pedidos-toolbar { display: flex; justify-content: space-between; align-items: center; gap: 15px; padding: 14px 16px; }
.pedidos-toolbar p { margin: 0; color: #887b8b; font-size: 10px; }
.pedidos-toolbar p strong { color: var(--pedidos-tinta); }
.pedidos-search { display: flex; align-items: center; gap: 6px; width: min(250px, 100%); height: 34px; padding: 0 10px; border: 1px solid #e8dce9; border-radius: 8px; color: #927d95; }
.pedidos-search span { font-size: 20px; }
.pedidos-search input { width: 100%; border: 0; outline: 0; color: var(--pedidos-tinta); background: transparent; font: inherit; font-size: 10px; }
.pedidos-table-wrap { overflow-x: auto; }
.pedidos-table { width: 100%; border-collapse: collapse; text-align: left; }
.pedidos-table th, .pedidos-table td { height: 46px; padding: 8px 12px; border-top: 1px solid #e9e1e9; font-size: 10px; white-space: nowrap; }
.pedidos-table th { color: #493d4d; background: #fcf9fc; font-size: 9px; font-weight: 700; }
.pedidos-table td:nth-child(2) strong, .pedidos-table td:nth-child(2) small { display: block; }
.pedidos-table td:nth-child(2) small { margin-top: 3px; color: #887b8b; font-size: 9px; }
.estado-pedido { display: inline-block; padding: 4px 8px; border-radius: 999px; color: #765879; background: #f1e8f2; font-size: 9px; }
.estado-pedido.estado-pendiente, .estado-pedido.estado-cotizacion { color: #88611e; background: #fff1d6; }
.estado-pedido.estado-en_proceso, .estado-pedido.estado-cotizacion_enviada { color: #236896; background: #e2f2fc; }
.estado-pedido.estado-terminado { color: #644993; background: #eee7fa; }
.estado-pedido.estado-entregado { color: #237b57; background: #e2f5e9; }
.estado-pedido.estado-cancelado { color: #9c2a52; background: #fde7eb; }
.pedidos-vacio { height: 100px !important; color: #887b8b; text-align: center; }
.pedidos-paginacion { display: flex; justify-content: space-between; align-items: center; gap: 12px; padding: 12px 16px; border-top: 1px solid #e9e1e9; color: #887b8b; font-size: 9px; }
.pedidos-paginacion div { display: flex; gap: 6px; }
.pedidos-paginacion button { min-height: 29px; padding: 0 11px; border: 1px solid #e6d9e8; border-radius: 6px; color: #55485b; background: white; cursor: pointer; font: inherit; }
.pedidos-paginacion button:disabled { cursor: not-allowed; opacity: .45; }
.pedidos-state, .pedidos-error { margin: 18px 0; padding: 15px; border-radius: 9px; color: #765879; background: white; font-size: 11px; }
.pedidos-error { color: #9c2a52; background: #fff0f3; }
@media (max-width: 760px) { .pedidos-page { display: block; }.pedidos-sidebar { position: static; min-height: auto; padding: 10px 14px; }.pedidos-admin { display: none; }.pedidos-nav { display: flex; overflow-x: auto; margin-top: 8px; }.nav-item { flex: 0 0 auto; }.pedidos-logout { position: absolute; top: 9px; right: 12px; min-height: 24px; }.pedidos-content { padding: 0 13px 24px; }.pedidos-breadcrumb { margin: 0 -13px 16px; padding: 0 13px; }.pedidos-heading { align-items: start; flex-direction: column; }.pedidos-total { width: 100%; } }
@media (max-width: 480px) { .pedidos-toolbar { align-items: stretch; flex-direction: column; }.pedidos-search { width: 100%; }.pedidos-table th, .pedidos-table td { padding-right: 9px; padding-left: 9px; }.pedidos-paginacion { align-items: flex-start; flex-direction: column; } }
</style>
