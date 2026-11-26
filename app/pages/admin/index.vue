<script setup lang="ts">
interface Rol {
  nombre: string
}

interface Usuario {
  id_usuario: number
  nombres: string
  apellidos: string
  role: Rol
}

interface Producto {
  id_producto: number
}

interface Pedido {
  id_pedido: number
  fecha_pedido: string
  cliente: { nombres: string; apellidos: string }
  total: string
  estado: string
}

interface ReporteVentas {
  grossSales: string
  ordersDelivered: number
  validatedPayments: number
  paymentsReceived: string
}

interface MesVentas {
  etiqueta: string
  grossSales: number
  ordersDelivered: number
  paymentsReceived: number
  validatedPayments: number
}

interface EstadoPedido {
  estado: string
  cantidad: string | number
}

interface ProductoMasPedido {
  id_producto: number
  nombre: string
  unidades: string | number
  ventas: string
}

interface ProductoStockBajo {
  id_producto: number
  codigo: string
  nombre: string
  stock: number
  stock_minimo: number
}

const config = useRuntimeConfig()
const token = ref('')
const usuario = ref<{ fullName?: string; role?: string } | null>(null)
const autorizado = ref(false)
const usuarios = ref<Usuario[]>([])
const productos = ref<Producto[]>([])
const pedidos = ref<Pedido[]>([])
const ventasMensuales = ref<MesVentas[]>([])
const pedidosPorEstado = ref<EstadoPedido[]>([])
const productosMasPedidos = ref<ProductoMasPedido[]>([])
const productosStockBajo = ref<ProductoStockBajo[]>([])
const cargando = ref(true)
const errorCarga = ref('')

const clientesCount = computed(() => usuarios.value.filter((user) => user.role?.nombre === 'cliente').length)
const ventasTotales = computed(() => ventasMensuales.value.reduce((sum, month) => sum + month.grossSales, 0))
const pagosTotales = computed(() => ventasMensuales.value.reduce((sum, month) => sum + month.paymentsReceived, 0))
const pagosValidados = computed(() => ventasMensuales.value.reduce((sum, month) => sum + month.validatedPayments, 0))
const pedidosEntregados = computed(() => ventasMensuales.value.reduce((sum, month) => sum + month.ordersDelivered, 0))
const mayorVenta = computed(() => Math.max(...ventasMensuales.value.map((month) => month.grossSales), 0))
const maxPedidosEstado = computed(() => Math.max(...pedidosPorEstado.value.map((item) => Number(item.cantidad)), 0))
const maxProductosMasPedidos = computed(() => Math.max(...productosMasPedidos.value.map((item) => Number(item.unidades)), 0))
const pedidosRecientes = computed(() => pedidos.value.slice(0, 8))

onMounted(async () => {
  const storedToken = localStorage.getItem('authToken')
  const storedUser = localStorage.getItem('authUser')

  if (!storedToken || !storedUser) {
    await navigateTo('/login', { replace: true })
    return
  }

  try {
    usuario.value = JSON.parse(storedUser)
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
  token.value = storedToken
  await cargarDashboard()
})

function fechaLocal(fecha: Date): string {
  const mes = String(fecha.getMonth() + 1).padStart(2, '0')
  const dia = String(fecha.getDate()).padStart(2, '0')
  return `${fecha.getFullYear()}-${mes}-${dia}`
}

async function cargarDashboard() {
  cargando.value = true
  errorCarga.value = ''
  const headers = { Authorization: `Bearer ${token.value}` }
  const year = new Date().getFullYear()
  const desde = fechaLocal(new Date(year, 0, 1))
  const hasta = fechaLocal(new Date(year, 11, 31))

  try {
    const reportesPorMes = Array.from({ length: 12 }, async (_, index) => {
      const start = new Date(year, index, 1)
      const end = new Date(year, index + 1, 0)
      const reporte = await $fetch<ReporteVentas>('/admin/reportes/ventas', {
        baseURL: config.public.apiBase,
        headers,
        query: { desde: fechaLocal(start), hasta: fechaLocal(end) },
      })
      return {
        etiqueta: start.toLocaleDateString('es-CO', { month: 'short' }).replace('.', ''),
        grossSales: Number(reporte.grossSales),
        ordersDelivered: reporte.ordersDelivered,
        paymentsReceived: Number(reporte.paymentsReceived),
        validatedPayments: reporte.validatedPayments,
      }
    })

    const [listaUsuarios, listaProductos, listaPedidos, listaVentas, listaEstados, listaTopProductos, listaStockBajo] = await Promise.all([
      $fetch<Usuario[]>('/users', { baseURL: config.public.apiBase, headers }),
      $fetch<Producto[]>('/productos/admin', { baseURL: config.public.apiBase, headers }),
      $fetch<Pedido[]>('/pedidos/admin', { baseURL: config.public.apiBase, headers }),
      Promise.all(reportesPorMes),
      $fetch<EstadoPedido[]>('/admin/reportes/pedidos-por-estado', {
        baseURL: config.public.apiBase,
        headers,
        query: { desde, hasta },
      }),
      $fetch<ProductoMasPedido[]>('/admin/reportes/productos-mas-vendidos', {
        baseURL: config.public.apiBase,
        headers,
        query: { desde, hasta, limite: 5 },
      }),
      $fetch<ProductoStockBajo[]>('/admin/reportes/stock-bajo', {
        baseURL: config.public.apiBase,
        headers,
      }),
    ])
    usuarios.value = listaUsuarios
    productos.value = listaProductos
    pedidos.value = listaPedidos
    ventasMensuales.value = listaVentas
    pedidosPorEstado.value = listaEstados
    productosMasPedidos.value = listaTopProductos
    productosStockBajo.value = listaStockBajo
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
      : apiMessage || 'No se pudo cargar el panel. Inténtalo de nuevo.'
  } finally {
    cargando.value = false
  }
}

function formatoMoneda(valor: string | number): string {
  return Number(valor).toLocaleString('es-CO', {
    style: 'currency',
    currency: 'COP',
    maximumFractionDigits: 0,
  })
}

function formatoFecha(fecha: string): string {
  return new Date(fecha).toLocaleDateString('es-CO', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  })
}

function nombreEstado(estado: string): string {
  const nombres: Record<string, string> = {
    pendiente: 'Pendientes',
    en_proceso: 'En proceso',
    terminado: 'Terminados',
    entregado: 'Entregados',
    cancelado: 'Cancelados',
    cotizacion: 'Cotizaciones',
    cotizacion_enviada: 'Cotizaciones enviadas',
  }
  return nombres[estado] ?? estado
}

function porcentaje(valor: number, maximo: number): string {
  return maximo > 0 ? `${(valor / maximo) * 100}%` : '0%'
}

function cerrarSesion() {
  localStorage.removeItem('authToken')
  localStorage.removeItem('authUser')
  navigateTo('/login', { replace: true })
}
</script>

<template>
  <main v-if="autorizado" class="dashboard-page">
    <aside class="dashboard-sidebar">
      <NuxtLink class="dashboard-brand" to="/admin" aria-label="DAN’STAMP">
        <BrandLogo size="medium" />
      </NuxtLink>

      <div class="dashboard-user">
        <span class="user-avatar">{{ usuario?.fullName?.charAt(0) || 'A' }}</span>
        <span><strong>{{ usuario?.fullName || 'Administrador' }}</strong><small>{{ usuario?.role === 'empleado' ? 'Empleado' : 'Administrador' }}</small></span>
      </div>

      <nav class="dashboard-nav" aria-label="Navegación de administrador">
        <NuxtLink class="nav-link current" to="/admin"><span>▦</span> Dashboard</NuxtLink>
        <NuxtLink class="nav-link" to="/admin/usuarios"><span>♙</span> Usuarios <b>›</b></NuxtLink>
        <NuxtLink class="nav-link" to="/admin/clientes"><span>♙</span> Clientes <b>›</b></NuxtLink>
        <NuxtLink class="nav-link" to="/admin/productos"><span>▣</span> Tienda <b>›</b></NuxtLink>
        <NuxtLink class="nav-link" to="/admin/pedidos"><span>🛒</span> Pedidos <b>›</b></NuxtLink>
      </nav>

      <button class="dashboard-logout" type="button" @click="cerrarSesion"><span>↪</span> Cerrar sesión</button>
    </aside>

    <section class="dashboard-main">
      <header class="dashboard-topbar">
        <span class="menu-mark">☰</span>
        <span class="topbar-profile" aria-label="Administrador">♙</span>
      </header>

      <div class="dashboard-content">
        <div class="dashboard-breadcrumb">
          <div class="breadcrumb-title"><span>▦</span><strong>Dashboard</strong><BrandLogo size="small" /></div>
          <div class="breadcrumb-path">⌂ <span>/</span> Dashboard</div>
        </div>

        <p v-if="errorCarga" class="dashboard-error" role="alert">{{ errorCarga }}</p>
        <div v-if="cargando" class="dashboard-loading">Cargando datos del panel...</div>

        <template v-else-if="!errorCarga">
          <section class="stat-grid" aria-label="Resumen del negocio">
            <NuxtLink id="usuarios" class="stat-card stat-link" to="/admin/usuarios">
              <span class="stat-icon">♧</span>
              <div><h2>USUARIOS</h2><strong>{{ usuarios.length }}</strong></div>
            </NuxtLink>
            <NuxtLink id="clientes" class="stat-card stat-link" to="/admin/clientes">
              <span class="stat-icon">♙</span>
              <div><h2>CLIENTES</h2><strong>{{ clientesCount }}</strong></div>
            </NuxtLink>
            <NuxtLink class="stat-card stat-link" to="/admin/productos">
              <span class="stat-icon">▣</span>
              <div><h2>TIENDA</h2><strong>{{ productos.length }}</strong></div>
            </NuxtLink>
            <NuxtLink class="stat-card stat-link" to="/admin/pedidos">
              <span class="stat-icon">🛒</span>
              <div><h2>PEDIDOS</h2><strong>{{ pedidos.length }}</strong></div>
            </NuxtLink>
          </section>

          <section class="dashboard-panels">
            <article id="pedidos" class="panel orders-panel">
              <header class="panel-heading">
                <h2>Últimos pedidos</h2>
                <NuxtLink to="/admin/pedidos">{{ pedidos.length }} en total · Ver todos</NuxtLink>
              </header>
              <div v-if="!pedidosRecientes.length" class="panel-empty">Todavía no hay pedidos registrados.</div>
              <div v-else class="orders-table-wrap">
                <table class="orders-table">
                  <thead><tr><th>#</th><th>Nombres y apellidos</th><th>Fecha</th><th>Monto</th></tr></thead>
                  <tbody>
                    <tr v-for="pedido in pedidosRecientes" :key="pedido.id_pedido">
                      <td>{{ pedido.id_pedido }}</td>
                      <td>{{ pedido.cliente.nombres }} {{ pedido.cliente.apellidos }}</td>
                      <td>{{ formatoFecha(pedido.fecha_pedido) }}</td>
                      <td>{{ formatoMoneda(pedido.total) }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </article>

            <article class="panel sales-panel">
              <header class="panel-heading">
                <div><h2>Ventas del año</h2><p>{{ formatoMoneda(ventasTotales) }} · {{ pedidosEntregados }} pedidos entregados</p></div>
              </header>
              <div class="chart">
                <div class="chart-y-labels"><span>{{ formatoMoneda(mayorVenta) }}</span><span>{{ formatoMoneda(mayorVenta / 2) }}</span><span>$0</span></div>
                <div class="chart-bars">
                  <div v-for="mes in ventasMensuales" :key="mes.etiqueta" class="chart-column" :title="`${mes.etiqueta}: ${formatoMoneda(mes.grossSales)}`">
                    <span class="bar-value" v-if="mes.grossSales > 0">{{ (mes.grossSales / 1000000).toFixed(1) }}</span>
                    <div class="chart-bar" :style="{ height: porcentaje(mes.grossSales, mayorVenta) }"></div>
                    <small>{{ mes.etiqueta }}</small>
                  </div>
                </div>
              </div>
              <p v-if="ventasTotales === 0" class="chart-empty">Aún no hay pedidos entregados este año.</p>
              <p class="payment-summary">Pagos validados: <strong>{{ formatoMoneda(pagosTotales) }}</strong> · {{ pagosValidados }} pagos</p>
              <div class="chart-legend"><span><i></i> Ventas entregadas</span></div>
            </article>
          </section>

          <section class="dashboard-panels report-panels" aria-label="Análisis del negocio">
            <article class="panel report-panel">
              <header class="panel-heading">
                <div><h2>Pedidos por estado</h2><p>Distribución de pedidos del año</p></div>
              </header>
              <div v-if="!pedidosPorEstado.length" class="panel-empty">Todavía no hay pedidos registrados.</div>
              <ul v-else class="report-list">
                <li v-for="item in pedidosPorEstado" :key="item.estado">
                  <div class="report-row-title"><span>{{ nombreEstado(item.estado) }}</span><strong>{{ Number(item.cantidad) }}</strong></div>
                  <div class="report-track"><span :style="{ width: porcentaje(Number(item.cantidad), maxPedidosEstado) }"></span></div>
                </li>
              </ul>
            </article>

            <article class="panel report-panel">
              <header class="panel-heading">
                <div><h2>Productos más pedidos</h2><p>Unidades en pedidos activos y entregados</p></div>
              </header>
              <div v-if="!productosMasPedidos.length" class="panel-empty">Aún no hay productos en pedidos.</div>
              <ul v-else class="report-list">
                <li v-for="item in productosMasPedidos" :key="item.id_producto">
                  <div class="report-row-title"><span>{{ item.nombre }}</span><strong>{{ Number(item.unidades) }} uds.</strong></div>
                  <div class="report-track product-track"><span :style="{ width: porcentaje(Number(item.unidades), maxProductosMasPedidos) }"></span></div>
                  <small class="product-sales">{{ formatoMoneda(item.ventas) }} en pedidos</small>
                </li>
              </ul>
            </article>

            <article class="panel report-panel stock-panel">
              <header class="panel-heading">
                <div><h2>Inventario por reponer</h2><p>Productos que alcanzaron el stock mínimo</p></div>
                <NuxtLink to="/admin/productos">Ver tienda</NuxtLink>
              </header>
              <div v-if="!productosStockBajo.length" class="panel-empty">No hay productos activos bajo su mínimo de inventario.</div>
              <ul v-else class="stock-list">
                <li v-for="item in productosStockBajo" :key="item.id_producto" class="stock-item">
                  <div class="report-row-title"><strong>{{ item.nombre }}</strong><span>{{ item.codigo }}</span></div>
                  <div class="stock-level"><span>Stock: {{ item.stock }} / mínimo: {{ item.stock_minimo }}</span><strong>{{ item.stock_minimo ? Math.round((item.stock / item.stock_minimo) * 100) : 0 }}%</strong></div>
                  <div class="report-track stock-track"><span :style="{ width: porcentaje(item.stock, item.stock_minimo) }"></span></div>
                </li>
              </ul>
            </article>
          </section>
        </template>
      </div>
    </section>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Space+Grotesk:wght@500;600;700&display=swap');

.dashboard-page { --dash-pink: #e56ddd; --dash-lilac: #f3c7ef; --dash-ink: #25202a; min-height: 100vh; display: grid; grid-template-columns: 220px minmax(0, 1fr); color: var(--dash-ink); background: linear-gradient(115deg, #f8e6f5 0%, #efdef8 52%, #fff4f8 100%); font-family: 'DM Sans', Arial, sans-serif; }
.dashboard-sidebar { position: sticky; top: 0; display: flex; flex-direction: column; min-height: 100vh; padding: 18px 14px; background: #f1c8ed; }
.dashboard-brand { display: flex; align-items: center; padding: 0 7px; text-decoration: none; }
.dashboard-user { display: flex; align-items: center; gap: 9px; margin: 25px 0 27px; padding: 8px 7px; }
.user-avatar { display: grid; place-items: center; flex: 0 0 34px; height: 34px; border-radius: 50%; color: white; background: #a83caf; font-weight: 700; }
.dashboard-user strong, .dashboard-user small { display: block; max-width: 130px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dashboard-user strong { font-size: 10px; }
.dashboard-user small { margin-top: 3px; color: #705873; font-size: 9px; }
.dashboard-nav { display: grid; gap: 7px; }
.nav-link, .dashboard-logout { display: flex; align-items: center; gap: 11px; min-height: 36px; padding: 0 9px; border: 0; border-radius: 8px; color: #46384a; background: transparent; font: inherit; font-size: 10px; text-decoration: none; }
.nav-link > span, .dashboard-logout > span { width: 15px; font-size: 15px; text-align: center; }
.nav-link b { margin-left: auto; font-size: 14px; }
.nav-link:hover, .nav-link.current { color: #7d2f83; background: rgba(255,255,255,.62); font-weight: 700; }
.dashboard-logout { margin-top: auto; cursor: pointer; }
.dashboard-logout:hover { color: #7d2f83; background: rgba(255,255,255,.62); }
.dashboard-main { min-width: 0; }
.dashboard-topbar { display: flex; justify-content: space-between; align-items: center; height: 27px; padding: 0 14px; color: #342537; background: #e47ae3; }
.menu-mark { font-size: 12px; }
.topbar-profile { font-size: 15px; }
.dashboard-content { padding: 0 22px 34px; }
.dashboard-breadcrumb { display: flex; justify-content: space-between; align-items: center; min-height: 45px; margin: 0 -22px 9px; padding: 0 22px; background: rgba(255,255,255,.92); }
.breadcrumb-title { display: flex; align-items: center; gap: 7px; font-size: 10px; }
.breadcrumb-title > span { color: #5a4a60; font-size: 16px; }
.breadcrumb-path { color: #685b6b; font-size: 9px; }
.breadcrumb-path span { padding: 0 4px; color: #b6a8b8; }
.dashboard-error, .dashboard-loading { margin: 14px 0; padding: 14px; border-radius: 8px; color: #8e328f; background: #fff; font-size: 11px; }
.dashboard-error { color: #9c2a52; background: #fff0f3; }
.stat-grid { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; margin: 10px 0 26px; }
.stat-card { display: flex; align-items: center; min-height: 61px; overflow: hidden; border: 1px solid rgba(217,196,222,.55); border-radius: 3px; color: var(--dash-ink); background: rgba(255,255,255,.94); text-decoration: none; }
.stat-icon { display: grid; place-items: center; align-self: stretch; flex: 0 0 58px; color: #35273a; background: #e978e5; font-size: 22px; }
.stat-card > div { padding: 7px 16px; }
.stat-card h2 { margin: 0 0 7px; font-size: 9px; font-weight: 700; }
.stat-card strong { font-size: 11px; }
.stat-link:hover { outline: 2px solid rgba(189,77,193,.42); }
.dashboard-panels { display: grid; grid-template-columns: 1fr 1fr; gap: 22px; align-items: stretch; }
.panel { min-width: 0; min-height: 238px; padding: 15px; background: rgba(255,255,255,.96); }
.panel-heading { display: flex; justify-content: space-between; align-items: baseline; gap: 8px; margin-bottom: 12px; }
.panel-heading h2 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: 14px; letter-spacing: -.03em; }
.panel-heading > span, .panel-heading > a { color: #98899a; font-size: 8px; }
.panel-heading > a { text-decoration: none; }
.panel-heading p { margin: 5px 0 0; color: #897a8c; font-size: 9px; }
.orders-table-wrap { overflow-x: auto; }
.orders-table { width: 100%; border-collapse: collapse; text-align: left; }
.orders-table th, .orders-table td { height: 25px; padding: 4px 5px; border-top: 1px solid #d5ced5; font-size: 9px; white-space: nowrap; }
.orders-table th { font-size: 8px; font-weight: 700; }
.orders-table td:nth-child(2) { max-width: 145px; overflow: hidden; text-overflow: ellipsis; }
.orders-table td:last-child, .orders-table th:last-child { text-align: right; }
.panel-empty { padding: 55px 8px; color: #948698; font-size: 10px; text-align: center; }
.sales-panel { display: flex; flex-direction: column; }
.chart { display: flex; flex: 1; min-height: 148px; padding: 8px 6px 0 0; }
.chart-y-labels { display: flex; flex-direction: column; justify-content: space-between; padding: 2px 6px 21px 0; color: #93869a; font-size: 7px; writing-mode: horizontal-tb; }
.chart-bars { display: flex; flex: 1; align-items: end; justify-content: space-around; gap: 8px; border-bottom: 1px solid #ddd6df; background: repeating-linear-gradient(to top, transparent 0, transparent 32%, rgba(216,207,219,.48) 33%, transparent 34%); }
.chart-column { position: relative; display: flex; flex: 1; flex-direction: column; align-items: center; justify-content: end; height: 100%; }
.chart-bar { width: min(58%, 28px); min-height: 3px; border-radius: 4px 4px 0 0; background: linear-gradient(180deg, #39a7f5, #9cd6fa); transition: height .3s ease; }
.bar-value { position: absolute; bottom: calc(100% + 3px); color: #398ddb; font-size: 7px; }
.chart-column small { position: absolute; bottom: -16px; color: #8a7f8e; font-size: 7px; }
.chart-legend { display: flex; justify-content: center; margin-top: 21px; color: #887a8c; font-size: 8px; }
.chart-legend span { display: flex; align-items: center; gap: 5px; }
.chart-legend i { width: 6px; height: 6px; border-radius: 50%; background: #39a7f5; }
.chart-empty { margin: 8px 0 0; color: #948698; font-size: 9px; text-align: center; }
.payment-summary { margin: 8px 0 0; color: #887a8c; font-size: 9px; text-align: center; }
.payment-summary strong { color: #423549; }
.report-panels { margin-top: 18px; }
.report-panel .panel-heading { align-items: flex-start; }
.report-panel .panel-heading p { max-width: 250px; line-height: 1.4; }
.report-panel .panel-heading a { color: #8e328f; font-size: 9px; text-decoration: none; }
.report-list, .stock-list { display: grid; gap: 13px; margin: 0; padding: 0; list-style: none; }
.report-row-title { display: flex; justify-content: space-between; gap: 12px; margin-bottom: 5px; color: #514653; font-size: 9px; }
.report-row-title strong { color: #332639; white-space: nowrap; }
.report-track { height: 7px; overflow: hidden; border-radius: 8px; background: #f0e9f1; }
.report-track span { display: block; height: 100%; border-radius: inherit; background: linear-gradient(90deg, #ce66d1, #ed9be9); transition: width .3s ease; }
.product-track span { background: linear-gradient(90deg, #38a5ed, #99d8fa); }
.product-sales { display: block; margin-top: 4px; color: #8a7f8e; font-size: 8px; }
.stock-panel { grid-column: 1 / -1; }
.stock-list { grid-template-columns: repeat(auto-fit, minmax(210px, 1fr)); gap: 10px; }
.stock-item { padding: 12px; border: 1px solid #eee6ef; border-radius: 6px; background: #fffafd; }
.stock-item .report-row-title { align-items: flex-start; }
.stock-item .report-row-title strong { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.stock-item .report-row-title span { color: #8a7f8e; font-size: 8px; }
.stock-level { display: flex; justify-content: space-between; margin: 11px 0 5px; color: #766b7b; font-size: 8px; }
.stock-level strong { color: #9c2a52; }
.stock-track span { background: linear-gradient(90deg, #e97878, #f3b2a1); }

@media (max-width: 900px) { .dashboard-page { grid-template-columns: 185px minmax(0, 1fr); }.dashboard-content { padding-right: 16px; padding-left: 16px; }.dashboard-breadcrumb { margin-right: -16px; margin-left: -16px; padding-right: 16px; padding-left: 16px; }.stat-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }.dashboard-panels { gap: 14px; } }
@media (max-width: 650px) { .dashboard-page { display: block; }.dashboard-sidebar { position: static; min-height: auto; padding: 9px 14px; }.dashboard-user { display: none; }.dashboard-nav { display: flex; gap: 4px; margin-top: 8px; overflow-x: auto; }.nav-link { flex: 0 0 auto; padding: 0 8px; }.dashboard-logout { position: absolute; top: 6px; right: 12px; min-height: 26px; }.dashboard-content { padding: 0 12px 24px; }.dashboard-breadcrumb { margin-right: -12px; margin-left: -12px; padding: 0 12px; }.dashboard-panels { grid-template-columns: 1fr; }.stat-icon { flex-basis: 44px; }.stat-card > div { padding: 7px 10px; } }
@media (max-width: 390px) { .stat-grid { gap: 7px; }.stat-card h2 { font-size: 8px; }.stat-icon { flex-basis: 38px; font-size: 18px; }.stat-card > div { padding: 6px 7px; }.stat-card strong { font-size: 10px; }.dashboard-content { padding-right: 8px; padding-left: 8px; } }
</style>
