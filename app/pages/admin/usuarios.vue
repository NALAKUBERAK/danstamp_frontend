<script setup lang="ts">
interface Usuario {
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
const usuarios = ref<Usuario[]>([])
const busqueda = ref('')
const pagina = ref(1)
const porPagina = 10
const cargando = ref(true)
const errorCarga = ref('')

const usuariosFiltrados = computed(() => {
  const consulta = busqueda.value.trim().toLocaleLowerCase()
  return usuarios.value.filter((item) => {
    const nombreCompleto = `${item.nombres} ${item.apellidos}`.toLocaleLowerCase()
    return !consulta
      || nombreCompleto.includes(consulta)
      || item.email.toLocaleLowerCase().includes(consulta)
      || item.role.nombre.toLocaleLowerCase().includes(consulta)
  })
})
const totalPaginas = computed(() => Math.max(1, Math.ceil(usuariosFiltrados.value.length / porPagina)))
const usuariosPagina = computed(() => {
  const inicio = (pagina.value - 1) * porPagina
  return usuariosFiltrados.value.slice(inicio, inicio + porPagina)
})
const usuariosActivos = computed(() => usuarios.value.filter((item) => item.isActive).length)

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
    usuarios.value = await $fetch<Usuario[]>('/users', {
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
      : apiMessage || 'No se pudo cargar la lista de usuarios. Inténtalo de nuevo.'
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
  <main v-if="autorizado" class="usuarios-page">
    <aside class="usuarios-sidebar">
      <NuxtLink class="usuarios-brand" to="/admin" aria-label="DAN’STAMP">
        <BrandLogo size="medium" />
      </NuxtLink>
      <div class="usuarios-admin">
        <span class="admin-avatar">{{ usuario?.fullName?.charAt(0) || 'A' }}</span>
        <span><strong>{{ usuario?.fullName || 'Administrador' }}</strong><small>{{ usuario?.role === 'empleado' ? 'Empleado' : 'Administrador' }}</small></span>
      </div>
      <nav class="usuarios-nav" aria-label="Navegación de administración">
        <NuxtLink class="nav-item" to="/admin"><span>▦</span> Dashboard</NuxtLink>
        <NuxtLink class="nav-item active" to="/admin/usuarios"><span>♙</span> Usuarios</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/clientes"><span>♧</span> Clientes</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/pedidos"><span>🛒</span> Pedidos</NuxtLink>
        <NuxtLink class="nav-item" to="/admin/productos"><span>▣</span> Productos</NuxtLink>
      </nav>
      <button class="usuarios-logout" type="button" @click="cerrarSesion"><span>↪</span> Cerrar sesión</button>
    </aside>

    <section class="usuarios-main">
      <header class="usuarios-topbar"><span>☰</span><span aria-label="Perfil administrador">♙</span></header>
      <div class="usuarios-content">
        <div class="usuarios-breadcrumb">
          <div><span aria-hidden="true">♙</span><strong>USUARIOS</strong><BrandLogo size="small" /></div>
          <p>⌂ <span>/</span> Usuarios</p>
        </div>

        <header class="usuarios-heading">
          <div>
            <p class="usuarios-eyebrow">PANEL DE ADMINISTRACIÓN</p>
            <h1>Usuarios registrados</h1>
            <p class="usuarios-description">Consulta las cuentas registradas en el sistema.</p>
          </div>
          <div class="usuarios-summary" aria-label="Resumen de usuarios">
            <article><span>Total registrados</span><strong>{{ usuarios.length }}</strong></article>
            <article><span>Activos</span><strong>{{ usuariosActivos }}</strong></article>
          </div>
        </header>

        <p v-if="errorCarga" class="usuarios-error" role="alert">{{ errorCarga }}</p>
        <div v-else-if="cargando" class="usuarios-state">Cargando usuarios...</div>
        <section v-else class="usuarios-panel" aria-label="Lista de usuarios registrados">
          <div class="usuarios-toolbar">
            <p><strong>{{ usuariosFiltrados.length }}</strong> usuario{{ usuariosFiltrados.length === 1 ? '' : 's' }}</p>
            <label class="usuarios-search">
              <span aria-hidden="true">⌕</span>
              <input v-model="busqueda" type="search" placeholder="Buscar usuario" aria-label="Buscar usuario" />
            </label>
          </div>

          <div class="usuarios-table-wrap">
            <table class="usuarios-table">
              <thead>
                <tr><th>ID</th><th>Nombre</th><th>Apellidos</th><th>Correo electrónico</th><th>Rol</th><th>Estado</th></tr>
              </thead>
              <tbody>
                <tr v-for="item in usuariosPagina" :key="item.id_usuario">
                  <td>{{ item.id_usuario }}</td>
                  <td>{{ item.nombres }}</td>
                  <td>{{ item.apellidos }}</td>
                  <td>{{ item.email }}</td>
                  <td>{{ item.role.nombre }}</td>
                  <td><span class="estado-usuario" :class="{ inactivo: !item.isActive }">{{ item.isActive ? 'Activo' : 'Inactivo' }}</span></td>
                </tr>
                <tr v-if="!usuariosFiltrados.length">
                  <td colspan="6" class="usuarios-vacio">{{ usuarios.length ? 'No hay usuarios que coincidan con la búsqueda.' : 'Todavía no hay usuarios registrados.' }}</td>
                </tr>
              </tbody>
            </table>
          </div>

          <footer class="usuarios-paginacion">
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

.usuarios-page { --usuarios-rosa: #e56ddd; --usuarios-tinta: #28212d; min-height: 100vh; display: grid; grid-template-columns: 220px minmax(0, 1fr); color: var(--usuarios-tinta); background: linear-gradient(115deg, #f8e6f5 0%, #efdef8 52%, #fff4f8 100%); font-family: 'DM Sans', Arial, sans-serif; }
.usuarios-sidebar { position: sticky; top: 0; display: flex; flex-direction: column; min-height: 100vh; padding: 20px 14px; background: #f1c8ed; }
.usuarios-brand { display: flex; align-items: center; padding: 0 7px; text-decoration: none; }
.usuarios-admin { display: flex; align-items: center; gap: 9px; margin: 28px 0; padding: 8px 7px; }
.admin-avatar { display: grid; place-items: center; flex: 0 0 36px; height: 36px; border-radius: 50%; color: white; background: #a83caf; font-weight: 700; }
.usuarios-admin strong, .usuarios-admin small { display: block; max-width: 140px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.usuarios-admin strong { font-size: 10px; }
.usuarios-admin small { margin-top: 3px; color: #705873; font-size: 9px; }
.usuarios-nav { display: grid; gap: 6px; }
.nav-item, .usuarios-logout { display: flex; align-items: center; gap: 10px; min-height: 37px; padding: 0 9px; border: 0; border-radius: 8px; color: #46384a; background: transparent; font: inherit; font-size: 10px; text-align: left; text-decoration: none; }
.nav-item span, .usuarios-logout span { width: 16px; font-size: 15px; text-align: center; }
.nav-item:hover, .nav-item.active { color: #7d2f83; background: rgba(255,255,255,.62); font-weight: 700; }
.usuarios-logout { margin-top: auto; cursor: pointer; }
.usuarios-logout:hover { color: #7d2f83; background: rgba(255,255,255,.62); }
.usuarios-main { min-width: 0; }
.usuarios-topbar { display: flex; justify-content: space-between; align-items: center; height: 28px; padding: 0 15px; color: #342537; background: #e47ae3; font-size: 13px; }
.usuarios-content { padding: 0 28px 36px; }
.usuarios-breadcrumb { display: flex; justify-content: space-between; align-items: center; min-height: 48px; margin: 0 -28px 20px; padding: 0 28px; background: rgba(255,255,255,.94); }
.usuarios-breadcrumb > div { display: flex; align-items: center; gap: 9px; font-size: 10px; }
.usuarios-breadcrumb > div > span { font-size: 17px; }
.usuarios-breadcrumb p { color: #786b7b; font-size: 9px; }
.usuarios-breadcrumb p { margin: 0; }
.usuarios-breadcrumb p span { padding: 0 5px; }
.usuarios-heading { display: flex; justify-content: space-between; align-items: end; gap: 18px; margin: 28px 0 22px; }
.usuarios-eyebrow { margin: 0 0 7px; color: #96789b; font-size: 9px; font-weight: 700; letter-spacing: .14em; }
.usuarios-heading h1 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: clamp(24px, 3vw, 34px); letter-spacing: -.05em; }
.usuarios-description { margin: 7px 0 0; color: #887b8b; font-size: 12px; }
.usuarios-summary { display: flex; gap: 10px; }
.usuarios-summary article { display: grid; gap: 6px; min-width: 110px; padding: 11px 14px; border: 1px solid #f0e5f1; border-radius: 10px; background: rgba(255,255,255,.8); }
.usuarios-summary span { color: #786b7b; font-size: 9px; }
.usuarios-summary strong { color: #8e328f; font-size: 19px; }
.usuarios-panel { overflow: hidden; border: 1px solid #eee4ef; border-radius: 12px; background: rgba(255,255,255,.96); box-shadow: 0 8px 24px rgba(106,61,110,.06); }
.usuarios-toolbar { display: flex; justify-content: space-between; align-items: center; gap: 15px; padding: 14px 16px; }
.usuarios-toolbar p { margin: 0; color: #887b8b; font-size: 10px; }
.usuarios-toolbar p strong { color: var(--usuarios-tinta); }
.usuarios-search { display: flex; align-items: center; gap: 6px; width: min(250px, 100%); height: 34px; padding: 0 10px; border: 1px solid #e8dce9; border-radius: 8px; color: #927d95; }
.usuarios-search span { font-size: 20px; }
.usuarios-search input { width: 100%; border: 0; outline: 0; color: var(--usuarios-tinta); background: transparent; font: inherit; font-size: 10px; }
.usuarios-table-wrap { overflow-x: auto; }
.usuarios-table { width: 100%; border-collapse: collapse; text-align: left; }
.usuarios-table th, .usuarios-table td { height: 43px; padding: 8px 12px; border-top: 1px solid #e9e1e9; font-size: 10px; white-space: nowrap; }
.usuarios-table th { color: #493d4d; background: #fcf9fc; font-size: 9px; font-weight: 700; }
.estado-usuario { display: inline-block; min-width: 58px; padding: 4px 8px; border-radius: 999px; color: #237b57; background: #e2f5e9; font-size: 9px; text-align: center; }
.estado-usuario.inactivo { color: #9c2a52; background: #fde7eb; }
.usuarios-vacio { height: 100px !important; color: #887b8b; text-align: center; }
.usuarios-paginacion { display: flex; justify-content: space-between; align-items: center; gap: 12px; padding: 12px 16px; border-top: 1px solid #e9e1e9; color: #887b8b; font-size: 9px; }
.usuarios-paginacion div { display: flex; gap: 6px; }
.usuarios-paginacion button { min-height: 29px; padding: 0 11px; border: 1px solid #e6d9e8; border-radius: 6px; color: #55485b; background: white; cursor: pointer; font: inherit; }
.usuarios-paginacion button:disabled { cursor: not-allowed; opacity: .45; }
.usuarios-state, .usuarios-error { margin: 18px 0; padding: 15px; border-radius: 9px; color: #765879; background: white; font-size: 11px; }
.usuarios-error { color: #9c2a52; background: #fff0f3; }
@media (max-width: 760px) { .usuarios-page { display: block; }.usuarios-sidebar { position: static; min-height: auto; padding: 10px 14px; }.usuarios-admin { display: none; }.usuarios-nav { display: flex; overflow-x: auto; margin-top: 8px; }.nav-item { flex: 0 0 auto; }.usuarios-logout { position: absolute; top: 9px; right: 12px; min-height: 24px; }.usuarios-content { padding: 0 13px 24px; }.usuarios-breadcrumb { margin: 0 -13px 16px; padding: 0 13px; }.usuarios-heading { align-items: start; flex-direction: column; }.usuarios-summary { width: 100%; }.usuarios-summary article { flex: 1; } }
@media (max-width: 480px) { .usuarios-toolbar { align-items: stretch; flex-direction: column; }.usuarios-search { width: 100%; }.usuarios-table th, .usuarios-table td { padding-right: 9px; padding-left: 9px; }.usuarios-paginacion { align-items: flex-start; flex-direction: column; } }
</style>
