<script setup lang="ts">
interface Cliente {
  id_usuario: number
  nombres: string
  apellidos: string
  email: string
  isActive: boolean
  role: { nombre: string }
}

interface UsuarioSesion {
  fullName?: string
  role?: string
}

const config = useRuntimeConfig()
const usuario = ref<UsuarioSesion | null>(null)
const autorizado = ref(false)
const clientes = ref<Cliente[]>([])
const busqueda = ref('')
const pagina = ref(1)
const porPagina = 10
const cargando = ref(true)
const errorCarga = ref('')

const clientesFiltrados = computed(() => {
  const consulta = busqueda.value.trim().toLocaleLowerCase()
  return clientes.value.filter((cliente) => {
    const nombreCompleto = `${cliente.nombres} ${cliente.apellidos}`.toLocaleLowerCase()
    return !consulta
      || nombreCompleto.includes(consulta)
      || cliente.email.toLocaleLowerCase().includes(consulta)
  })
})
const totalPaginas = computed(() => Math.max(1, Math.ceil(clientesFiltrados.value.length / porPagina)))
const clientesPagina = computed(() => {
  const inicio = (pagina.value - 1) * porPagina
  return clientesFiltrados.value.slice(inicio, inicio + porPagina)
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
    const usuarios = await $fetch<Cliente[]>('/users', {
      baseURL: config.public.apiBase,
      headers: { Authorization: `Bearer ${storedToken}` },
    })
    clientes.value = usuarios.filter((item) => item.role?.nombre?.toLowerCase() === 'cliente')
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
      : apiMessage || 'No se pudo cargar la lista de clientes. Inténtalo de nuevo.'
  } finally {
    cargando.value = false
  }
})

function cerrarSesion() {
  localStorage.removeItem('authToken')
  localStorage.removeItem('authUser')
  navigateTo('/login', { replace: true })
}
</script>

<template>
  <main v-if="autorizado" class="clientes-page">
    <aside class="clientes-sidebar">
      <NuxtLink class="clientes-brand" to="/admin" aria-label="DAN’STAMP">
        <BrandLogo size="medium" />
      </NuxtLink>
      <div class="clientes-admin">
        <span class="admin-avatar">{{ usuario?.fullName?.charAt(0) || 'A' }}</span>
        <span><strong>{{ usuario?.fullName || 'Administrador' }}</strong><small>{{ usuario?.role === 'empleado' ? 'Empleado' : 'Administrador' }}</small></span>
      </div>
      <nav class="clientes-nav" aria-label="Navegación de administración">
        <NuxtLink class="nav-item" to="/admin"><span>▦</span> Dashboard</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/usuarios"><span>♙</span> Usuarios</NuxtLink>
        <NuxtLink class="nav-item active" to="/admin/clientes"><span>♧</span> Clientes</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/pedidos"><span>🛒</span> Pedidos</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/productos"><span>▣</span> Productos</NuxtLink>
      </nav>
      <button class="clientes-logout" type="button" @click="cerrarSesion"><span>↪</span> Cerrar sesión</button>
    </aside>

    <section class="clientes-main">
      <header class="clientes-topbar"><span>☰</span><span aria-label="Perfil administrador">♙</span></header>
      <div class="clientes-content">
        <div class="clientes-breadcrumb">
          <div><span aria-hidden="true">♧</span><strong>CLIENTES</strong><BrandLogo size="small" /></div>
          <p>⌂ <span>/</span> Clientes</p>
        </div>

        <header class="clientes-heading">
          <div>
            <p class="clientes-eyebrow">PANEL DE ADMINISTRACIÓN</p>
            <h1>Clientes registrados</h1>
            <p class="clientes-description">Personas registradas como clientes para comprar los productos.</p>
          </div>
          <article class="clientes-total" aria-label="Total de clientes registrados">
            <span>Total registrados</span>
            <strong>{{ clientes.length }}</strong>
          </article>
        </header>

        <p v-if="errorCarga" class="clientes-error" role="alert">{{ errorCarga }}</p>
        <div v-else-if="cargando" class="clientes-state">Cargando clientes...</div>
        <section v-else class="clientes-panel" aria-label="Lista de clientes">
          <div class="clientes-toolbar">
            <p>Mostrando <strong>{{ clientesFiltrados.length }}</strong> de <strong>{{ clientes.length }}</strong> clientes</p>
            <label class="clientes-search">
              <span aria-hidden="true">⌕</span>
              <input v-model="busqueda" type="search" placeholder="Buscar cliente" aria-label="Buscar cliente" />
            </label>
          </div>

          <div class="clientes-table-wrap">
            <table class="clientes-table">
              <thead>
                <tr><th>ID</th><th>Nombres</th><th>Apellidos</th><th>Correo electrónico</th><th>Estado</th></tr>
              </thead>
              <tbody>
                <tr v-for="cliente in clientesPagina" :key="cliente.id_usuario">
                  <td>{{ cliente.id_usuario }}</td>
                  <td>{{ cliente.nombres }}</td>
                  <td>{{ cliente.apellidos }}</td>
                  <td>{{ cliente.email }}</td>
                  <td><span class="estado-cliente" :class="{ inactivo: !cliente.isActive }">{{ cliente.isActive ? 'Activo' : 'Inactivo' }}</span></td>
                </tr>
                <tr v-if="!clientesFiltrados.length">
                  <td colspan="5" class="clientes-vacio">{{ clientes.length ? 'No hay clientes que coincidan con la búsqueda.' : 'Todavía no hay clientes registrados.' }}</td>
                </tr>
              </tbody>
            </table>
          </div>

          <footer class="clientes-paginacion">
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

.clientes-page { --clientes-tinta: #28212d; min-height: 100vh; display: grid; grid-template-columns: 220px minmax(0, 1fr); color: var(--clientes-tinta); background: linear-gradient(115deg, #f8e6f5 0%, #efdef8 52%, #fff4f8 100%); font-family: 'DM Sans', Arial, sans-serif; }
.clientes-sidebar { position: sticky; top: 0; display: flex; flex-direction: column; min-height: 100vh; padding: 20px 14px; background: #f1c8ed; }
.clientes-brand { display: flex; align-items: center; padding: 0 7px; text-decoration: none; }
.clientes-admin { display: flex; align-items: center; gap: 9px; margin: 28px 0; padding: 8px 7px; }
.admin-avatar { display: grid; place-items: center; flex: 0 0 36px; height: 36px; border-radius: 50%; color: white; background: #a83caf; font-weight: 700; }
.clientes-admin strong, .clientes-admin small { display: block; max-width: 140px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.clientes-admin strong { font-size: 10px; }
.clientes-admin small { margin-top: 3px; color: #705873; font-size: 9px; }
.clientes-nav { display: grid; gap: 6px; }
.nav-item, .clientes-logout { display: flex; align-items: center; gap: 10px; min-height: 37px; padding: 0 9px; border: 0; border-radius: 8px; color: #46384a; background: transparent; font: inherit; font-size: 10px; text-align: left; text-decoration: none; }
.nav-item span, .clientes-logout span { width: 16px; font-size: 15px; text-align: center; }
.nav-item:hover, .nav-item.active { color: #7d2f83; background: rgba(255,255,255,.62); font-weight: 700; }
.clientes-logout { margin-top: auto; cursor: pointer; }
.clientes-logout:hover { color: #7d2f83; background: rgba(255,255,255,.62); }
.clientes-main { min-width: 0; }
.clientes-topbar { display: flex; justify-content: space-between; align-items: center; height: 28px; padding: 0 15px; color: #342537; background: #e47ae3; font-size: 13px; }
.clientes-content { padding: 0 28px 36px; }
.clientes-breadcrumb { display: flex; justify-content: space-between; align-items: center; min-height: 48px; margin: 0 -28px 20px; padding: 0 28px; background: rgba(255,255,255,.94); }
.clientes-breadcrumb > div { display: flex; align-items: center; gap: 9px; font-size: 10px; }
.clientes-breadcrumb > div > span { font-size: 17px; }
.clientes-breadcrumb p { color: #786b7b; font-size: 9px; }
.clientes-breadcrumb p { margin: 0; }
.clientes-breadcrumb p span { padding: 0 5px; }
.clientes-heading { display: flex; justify-content: space-between; align-items: end; gap: 18px; margin: 28px 0 22px; }
.clientes-eyebrow { margin: 0 0 7px; color: #96789b; font-size: 9px; font-weight: 700; letter-spacing: .14em; }
.clientes-heading h1 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: clamp(24px, 3vw, 34px); letter-spacing: -.05em; }
.clientes-description { margin: 7px 0 0; color: #887b8b; font-size: 12px; }
.clientes-total { display: grid; gap: 6px; min-width: 125px; padding: 11px 14px; border: 1px solid #f0e5f1; border-radius: 10px; background: rgba(255,255,255,.8); }
.clientes-total span { color: #786b7b; font-size: 9px; }
.clientes-total strong { color: #8e328f; font-size: 19px; }
.clientes-panel { overflow: hidden; border: 1px solid #eee4ef; border-radius: 12px; background: rgba(255,255,255,.96); box-shadow: 0 8px 24px rgba(106,61,110,.06); }
.clientes-toolbar { display: flex; justify-content: space-between; align-items: center; gap: 15px; padding: 14px 16px; }
.clientes-toolbar p { margin: 0; color: #887b8b; font-size: 10px; }
.clientes-toolbar p strong { color: var(--clientes-tinta); }
.clientes-search { display: flex; align-items: center; gap: 6px; width: min(250px, 100%); height: 34px; padding: 0 10px; border: 1px solid #e8dce9; border-radius: 8px; color: #927d95; }
.clientes-search span { font-size: 20px; }
.clientes-search input { width: 100%; border: 0; outline: 0; color: var(--clientes-tinta); background: transparent; font: inherit; font-size: 10px; }
.clientes-table-wrap { overflow-x: auto; }
.clientes-table { width: 100%; border-collapse: collapse; text-align: left; }
.clientes-table th, .clientes-table td { height: 43px; padding: 8px 12px; border-top: 1px solid #e9e1e9; font-size: 10px; white-space: nowrap; }
.clientes-table th { color: #493d4d; background: #fcf9fc; font-size: 9px; font-weight: 700; }
.estado-cliente { display: inline-block; min-width: 58px; padding: 4px 8px; border-radius: 999px; color: #237b57; background: #e2f5e9; font-size: 9px; text-align: center; }
.estado-cliente.inactivo { color: #9c2a52; background: #fde7eb; }
.clientes-vacio { height: 100px !important; color: #887b8b; text-align: center; }
.clientes-paginacion { display: flex; justify-content: space-between; align-items: center; gap: 12px; padding: 12px 16px; border-top: 1px solid #e9e1e9; color: #887b8b; font-size: 9px; }
.clientes-paginacion div { display: flex; gap: 6px; }
.clientes-paginacion button { min-height: 29px; padding: 0 11px; border: 1px solid #e6d9e8; border-radius: 6px; color: #55485b; background: white; cursor: pointer; font: inherit; }
.clientes-paginacion button:disabled { cursor: not-allowed; opacity: .45; }
.clientes-state, .clientes-error { margin: 18px 0; padding: 15px; border-radius: 9px; color: #765879; background: white; font-size: 11px; }
.clientes-error { color: #9c2a52; background: #fff0f3; }
@media (max-width: 760px) { .clientes-page { display: block; }.clientes-sidebar { position: static; min-height: auto; padding: 10px 14px; }.clientes-admin { display: none; }.clientes-nav { display: flex; overflow-x: auto; margin-top: 8px; }.nav-item { flex: 0 0 auto; }.clientes-logout { position: absolute; top: 9px; right: 12px; min-height: 24px; }.clientes-content { padding: 0 13px 24px; }.clientes-breadcrumb { margin: 0 -13px 16px; padding: 0 13px; }.clientes-heading { align-items: start; flex-direction: column; }.clientes-total { width: 100%; } }
@media (max-width: 480px) { .clientes-toolbar { align-items: stretch; flex-direction: column; }.clientes-search { width: 100%; }.clientes-table th, .clientes-table td { padding-right: 9px; padding-left: 9px; }.clientes-paginacion { align-items: flex-start; flex-direction: column; } }
</style>
