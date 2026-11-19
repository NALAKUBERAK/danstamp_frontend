<script setup lang="ts">
interface Categoria {
  id_categoria: number
  nombre: string
  estado: boolean
}

interface Producto {
  id_producto: number
  codigo: string
  nombre: string
  descripcion: string | null
  material: string | null
  color: string | null
  talla: string | null
  precio_base: string
  stock: number
  stock_minimo: number
  imagen_url: string | null
  estado: boolean
  categoria: Categoria | null
}

interface UsuarioSesion {
  fullName?: string
  email?: string
  role?: string
}

const config = useRuntimeConfig()
const token = ref('')
const usuario = ref<UsuarioSesion | null>(null)
const productos = ref<Producto[]>([])
const categorias = ref<Categoria[]>([])
const busqueda = ref('')
const filtroEstado = ref('todos')
const orden = ref('nombre')
const cargando = ref(true)
const guardando = ref(false)
const errorCarga = ref('')
const mensaje = ref('')
const modalAbierto = ref(false)
const productoEditando = ref<number | null>(null)
const archivoImagen = ref<File | null>(null)

const formulario = reactive({
  codigo: '',
  nombre: '',
  descripcion: '',
  id_categoria: '',
  material: '',
  color: '',
  talla: '',
  precio_base: '',
  stock: '0',
  stock_minimo: '0',
  imagen_url: '',
  estado: true,
})

const productosFiltrados = computed(() => {
  const consulta = busqueda.value.trim().toLocaleLowerCase()
  const resultado = productos.value.filter((producto) => {
    const coincideBusqueda = !consulta
      || producto.nombre.toLocaleLowerCase().includes(consulta)
      || producto.codigo.toLocaleLowerCase().includes(consulta)
      || producto.categoria?.nombre.toLocaleLowerCase().includes(consulta)
    const coincideEstado = filtroEstado.value === 'todos'
      || (filtroEstado.value === 'activos' && producto.estado)
      || (filtroEstado.value === 'inactivos' && !producto.estado)
    return coincideBusqueda && coincideEstado
  })

  return resultado.sort((a, b) => {
    if (orden.value === 'precio-asc') return Number(a.precio_base) - Number(b.precio_base)
    if (orden.value === 'precio-desc') return Number(b.precio_base) - Number(a.precio_base)
    return a.nombre.localeCompare(b.nombre, 'es')
  })
})

const productosActivos = computed(() => productos.value.filter((producto) => producto.estado).length)
const productosInactivos = computed(() => productos.value.length - productosActivos.value)

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

  if (usuario.value?.role?.toLowerCase() !== 'admin') {
    await navigateTo('/principal', { replace: true })
    return
  }

  token.value = storedToken
  await cargarDatos()
})

async function cargarDatos() {
  cargando.value = true
  errorCarga.value = ''

  try {
    const headers = { Authorization: `Bearer ${token.value}` }
    const [listaProductos, listaCategorias] = await Promise.all([
      $fetch<Producto[]>('/productos/admin', { baseURL: config.public.apiBase, headers }),
      $fetch<Categoria[]>('/categorias/admin', { baseURL: config.public.apiBase, headers }),
    ])
    productos.value = listaProductos
    categorias.value = listaCategorias
  } catch (error: unknown) {
    const apiError = error as { statusCode?: number; data?: { message?: string | string[] } }
    if (apiError.statusCode === 401 || apiError.statusCode === 403) {
      localStorage.removeItem('authToken')
      localStorage.removeItem('authUser')
      await navigateTo('/login', { replace: true })
      return
    }
    errorCarga.value = obtenerMensaje(error, 'No se pudo cargar el catálogo. Inténtalo de nuevo.')
  } finally {
    cargando.value = false
  }
}

function obtenerMensaje(error: unknown, fallback: string): string {
  const apiError = error as { data?: { message?: string | string[] } }
  const apiMessage = apiError.data?.message
  if (Array.isArray(apiMessage)) return apiMessage.join('. ')
  return apiMessage || fallback
}

function abrirNuevoProducto() {
  productoEditando.value = null
  Object.assign(formulario, {
    codigo: '',
    nombre: '',
    descripcion: '',
    id_categoria: '',
    material: '',
    color: '',
    talla: '',
    precio_base: '',
    stock: '0',
    stock_minimo: '0',
    imagen_url: '',
    estado: true,
  })
  archivoImagen.value = null
  mensaje.value = ''
  modalAbierto.value = true
}

function abrirEdicion(producto: Producto) {
  productoEditando.value = producto.id_producto
  Object.assign(formulario, {
    codigo: producto.codigo,
    nombre: producto.nombre,
    descripcion: producto.descripcion ?? '',
    id_categoria: producto.categoria ? String(producto.categoria.id_categoria) : '',
    material: producto.material ?? '',
    color: producto.color ?? '',
    talla: producto.talla ?? '',
    precio_base: producto.precio_base,
    stock: String(producto.stock),
    stock_minimo: String(producto.stock_minimo),
    imagen_url: producto.imagen_url ?? '',
    estado: producto.estado,
  })
  archivoImagen.value = null
  mensaje.value = ''
  modalAbierto.value = true
}

function seleccionarImagen(event: Event) {
  const input = event.target as HTMLInputElement
  archivoImagen.value = input.files?.[0] ?? null
}

async function guardarProducto() {
  guardando.value = true
  mensaje.value = ''
  const datos = new FormData()
  datos.append('codigo', formulario.codigo.trim())
  datos.append('nombre', formulario.nombre.trim())
  datos.append('descripcion', formulario.descripcion.trim())
  datos.append('material', formulario.material.trim())
  datos.append('color', formulario.color.trim())
  datos.append('talla', formulario.talla.trim())
  datos.append('precio_base', formulario.precio_base)
  datos.append('stock', formulario.stock)
  datos.append('stock_minimo', formulario.stock_minimo)
  datos.append('estado', String(formulario.estado))
  if (formulario.id_categoria) datos.append('id_categoria', formulario.id_categoria)
  if (archivoImagen.value) datos.append('imagen', archivoImagen.value)
  else if (formulario.imagen_url.trim()) datos.append('imagen_url', formulario.imagen_url.trim())

  try {
    await $fetch(productoEditando.value ? `/productos/${productoEditando.value}` : '/productos', {
      baseURL: config.public.apiBase,
      method: productoEditando.value ? 'PATCH' : 'POST',
      headers: { Authorization: `Bearer ${token.value}` },
      body: datos,
    })
    modalAbierto.value = false
    await cargarDatos()
  } catch (error: unknown) {
    mensaje.value = obtenerMensaje(error, 'No se pudo guardar el producto.')
  } finally {
    guardando.value = false
  }
}

async function cambiarEstado(producto: Producto) {
  const accion = producto.estado ? 'desactivar' : 'activar'
  if (!window.confirm(`¿Quieres ${accion} «${producto.nombre}»?`)) return

  try {
    await $fetch(`/productos/${producto.id_producto}`, {
      baseURL: config.public.apiBase,
      method: 'PATCH',
      headers: { Authorization: `Bearer ${token.value}` },
      body: { estado: !producto.estado },
    })
    await cargarDatos()
  } catch (error: unknown) {
    errorCarga.value = obtenerMensaje(error, `No se pudo ${accion} el producto.`)
  }
}

function exportarCsv() {
  const encabezados = ['Código', 'Producto', 'Categoría', 'Precio', 'Stock', 'Estado']
  const filas = productosFiltrados.value.map((producto) => [
    producto.codigo,
    producto.nombre,
    producto.categoria?.nombre ?? '',
    producto.precio_base,
    String(producto.stock),
    producto.estado ? 'Activo' : 'Inactivo',
  ])
  const csv = [encabezados, ...filas]
    .map((fila) => fila.map((valor) => `"${valor.replaceAll('"', '""')}"`).join(','))
    .join('\r\n')
  const enlace = document.createElement('a')
  enlace.href = URL.createObjectURL(new Blob(['\uFEFF', csv], { type: 'text/csv;charset=utf-8' }))
  enlace.download = 'catalogo-danstamp.csv'
  enlace.click()
  URL.revokeObjectURL(enlace.href)
}

function cerrarSesion() {
  localStorage.removeItem('authToken')
  localStorage.removeItem('authUser')
  navigateTo('/login', { replace: true })
}
</script>

<template>
  <main class="admin-page">
    <aside class="admin-sidebar">
      <NuxtLink class="admin-brand" to="/admin"><span class="brand-mark">D</span><span>DAN’STAMP</span></NuxtLink>
      <div class="admin-user">
        <span class="user-avatar">{{ usuario?.fullName?.charAt(0) || 'A' }}</span>
        <span><strong>{{ usuario?.fullName || 'Administrador' }}</strong><small>Administrador</small></span>
      </div>
      <nav class="admin-nav" aria-label="Navegación de administración">
        <NuxtLink class="nav-item active" to="/admin/productos"><span>▦</span> Catálogo de productos</NuxtLink>
      </nav>
      <button class="logout-button" type="button" @click="cerrarSesion"><span>↪</span> Cerrar sesión</button>
    </aside>

    <section class="admin-content">
      <header class="admin-topbar">
        <span class="topbar-section"><i></i> PRODUCTOS</span>
        <div class="topbar-actions">
          <button type="button" class="button button-dark" @click="abrirNuevoProducto"><span>＋</span> Agregar producto</button>
          <button type="button" class="button button-light" :disabled="!productosFiltrados.length" @click="exportarCsv">⇩ Exportar CSV</button>
        </div>
      </header>

      <div class="admin-heading">
        <div>
          <p class="eyebrow">PANEL DE ADMINISTRACIÓN</p>
          <h1>Catálogo Dan Stamp</h1>
          <p class="heading-copy">Gestiona productos, precios, inventario e imágenes de tu catálogo.</p>
        </div>
        <div class="admin-summary">
          <span><strong>{{ productos.length }}</strong> productos</span>
          <span><strong>{{ productosActivos }}</strong> activos</span>
          <span><strong>{{ productosInactivos }}</strong> inactivos</span>
        </div>
      </div>

      <section class="catalog-toolbar" aria-label="Filtros del catálogo">
        <label class="search-box">
          <span aria-hidden="true">⌕</span>
          <input v-model="busqueda" type="search" placeholder="Buscar por nombre, código o categoría" />
        </label>
        <label class="select-filter">
          <span>Estado</span>
          <select v-model="filtroEstado">
            <option value="todos">Todos</option>
            <option value="activos">Activos</option>
            <option value="inactivos">Inactivos</option>
          </select>
        </label>
        <label class="select-filter">
          <span>Ordenar</span>
          <select v-model="orden">
            <option value="nombre">Nombre A-Z</option>
            <option value="precio-asc">Precio menor</option>
            <option value="precio-desc">Precio mayor</option>
          </select>
        </label>
      </section>

      <p v-if="errorCarga" class="state-message error-message" role="alert">{{ errorCarga }}</p>
      <div v-if="cargando" class="state-message">Cargando catálogo...</div>
      <div v-else-if="!errorCarga && !productosFiltrados.length" class="empty-state">
        <span>✦</span>
        <h2>{{ productos.length ? 'No encontramos productos' : 'Tu catálogo está listo para empezar' }}</h2>
        <p>{{ productos.length ? 'Prueba con otros filtros o términos de búsqueda.' : 'Agrega el primer producto para mostrarlo aquí.' }}</p>
        <button v-if="!productos.length" type="button" class="button button-dark" @click="abrirNuevoProducto">Agregar producto</button>
      </div>
      <section v-else class="product-grid" aria-label="Productos">
        <article v-for="producto in productosFiltrados" :key="producto.id_producto" class="product-card">
          <div class="product-image">
            <img v-if="producto.imagen_url" :src="producto.imagen_url" :alt="producto.nombre" />
            <span v-else class="image-placeholder">D</span>
            <span class="status-badge" :class="{ inactive: !producto.estado }">{{ producto.estado ? 'Activo' : 'Inactivo' }}</span>
          </div>
          <div class="product-details">
            <div class="product-category">{{ producto.categoria?.nombre || 'Sin categoría' }}<span>·</span>{{ producto.codigo }}</div>
            <h2>{{ producto.nombre }}</h2>
            <p class="product-meta">{{ [producto.color, producto.material, producto.talla].filter(Boolean).join(' · ') || 'Producto Dan Stamp' }}</p>
            <div class="product-bottom">
              <strong class="product-price">{{ Number(producto.precio_base).toLocaleString('es-CO', { style: 'currency', currency: 'COP', maximumFractionDigits: 0 }) }}</strong>
              <span class="product-stock" :class="{ low: producto.stock <= producto.stock_minimo }">Stock {{ producto.stock }}</span>
            </div>
            <div class="product-actions">
              <button type="button" class="action-edit" @click="abrirEdicion(producto)">Editar</button>
              <button type="button" class="action-status" @click="cambiarEstado(producto)">{{ producto.estado ? 'Desactivar' : 'Activar' }}</button>
            </div>
          </div>
        </article>
      </section>
    </section>

    <div v-if="modalAbierto" class="modal-backdrop" @click.self="modalAbierto = false">
      <section class="product-modal" role="dialog" aria-modal="true" aria-labelledby="modal-title">
        <header class="modal-header">
          <div><p class="eyebrow">CATÁLOGO</p><h2 id="modal-title">{{ productoEditando ? 'Editar producto' : 'Agregar producto' }}</h2></div>
          <button type="button" class="modal-close" aria-label="Cerrar" @click="modalAbierto = false">×</button>
        </header>
        <form class="product-form" @submit.prevent="guardarProducto">
          <div class="form-pair">
            <label>Código<input v-model="formulario.codigo" maxlength="30" required /></label>
            <label>Nombre<input v-model="formulario.nombre" maxlength="100" required /></label>
          </div>
          <div class="form-pair">
            <label>Categoría
              <select v-model="formulario.id_categoria">
                <option value="">Sin categoría</option>
                <option v-for="categoria in categorias.filter((item) => item.estado)" :key="categoria.id_categoria" :value="String(categoria.id_categoria)">{{ categoria.nombre }}</option>
              </select>
            </label>
            <label>Precio (COP)<input v-model="formulario.precio_base" type="number" min="0" step="0.01" required /></label>
          </div>
          <div class="form-pair form-triple">
            <label>Stock<input v-model="formulario.stock" type="number" min="0" step="1" required /></label>
            <label>Stock mínimo<input v-model="formulario.stock_minimo" type="number" min="0" step="1" required /></label>
            <label>Talla<input v-model="formulario.talla" maxlength="30" /></label>
          </div>
          <div class="form-pair">
            <label>Material<input v-model="formulario.material" maxlength="50" /></label>
            <label>Color<input v-model="formulario.color" maxlength="30" /></label>
          </div>
          <label>Descripción<textarea v-model="formulario.descripcion" rows="3"></textarea></label>
          <label>Imagen del producto<input type="file" accept="image/jpeg,image/png,image/webp" @change="seleccionarImagen" /><small>JPG, PNG o WEBP; máximo 5 MB.</small></label>
          <label v-if="!archivoImagen">O URL de imagen<input v-model="formulario.imagen_url" type="url" placeholder="https://..." /></label>
          <label class="checkbox-label"><input v-model="formulario.estado" type="checkbox" /> Producto activo</label>
          <p v-if="mensaje" class="state-message error-message" role="alert">{{ mensaje }}</p>
          <footer class="modal-actions">
            <button type="button" class="button button-light" @click="modalAbierto = false">Cancelar</button>
            <button type="submit" class="button button-dark" :disabled="guardando">{{ guardando ? 'Guardando...' : 'Guardar producto' }}</button>
          </footer>
        </form>
      </section>
    </div>
  </main>
</template>

<style>
.admin-page {
  --admin-pink: #e56ddd;
  --admin-lilac: #f3c7ef;
  --admin-ink: #28212d;
  min-height: 100vh;
  display: grid;
  grid-template-columns: 248px minmax(0, 1fr);
  color: var(--admin-ink);
  background: #fcf8fb;
  font-family: 'DM Sans', Arial, sans-serif;
}

.admin-sidebar {
  position: sticky;
  top: 0;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  padding: 26px 18px 22px;
  background: linear-gradient(180deg, #f3c7ef, #efdaf4);
}

.admin-brand { display: flex; align-items: center; gap: 10px; padding: 0 8px; color: var(--admin-ink); font-family: 'Space Grotesk', Arial, sans-serif; font-size: 13px; font-weight: 700; letter-spacing: .05em; text-decoration: none; }
.brand-mark { display: grid; place-items: center; width: 30px; height: 30px; border-radius: 50%; color: white; background: #bd4dc1; font-size: 16px; }
.admin-user { display: flex; align-items: center; gap: 11px; margin: 38px 0 34px; padding: 14px 10px; border-radius: 12px; background: rgba(255,255,255,.52); }
.user-avatar { display: grid; place-items: center; flex: 0 0 38px; height: 38px; border-radius: 50%; color: white; background: #a83caf; font-weight: 700; }
.admin-user strong, .admin-user small { display: block; max-width: 150px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.admin-user strong { font-size: 12px; }
.admin-user small { margin-top: 4px; color: #765879; font-size: 10px; }
.admin-nav { display: grid; gap: 7px; }
.nav-item, .logout-button { display: flex; align-items: center; gap: 12px; min-height: 42px; padding: 0 12px; border: 0; border-radius: 10px; color: #55485b; background: transparent; font: inherit; font-size: 12px; text-align: left; text-decoration: none; }
.nav-item span, .logout-button span { width: 18px; font-size: 17px; text-align: center; }
.nav-item.active { color: #8e328f; background: rgba(255,255,255,.72); font-weight: 700; }
.logout-button { margin-top: auto; cursor: pointer; }
.logout-button:hover { color: #8e328f; background: rgba(255,255,255,.55); }

.admin-content { min-width: 0; padding: 0 clamp(22px, 4vw, 58px) 48px; }
.admin-topbar { display: flex; justify-content: space-between; align-items: center; min-height: 74px; border-bottom: 1px solid #eee4ef; }
.topbar-section { display: flex; align-items: center; gap: 9px; font-size: 11px; font-weight: 700; letter-spacing: .08em; }
.topbar-section i { width: 10px; height: 10px; border-radius: 50%; background: var(--admin-pink); }
.topbar-actions { display: flex; gap: 9px; }
.button { display: inline-flex; align-items: center; justify-content: center; gap: 7px; min-height: 36px; padding: 0 13px; border: 1px solid transparent; border-radius: 999px; cursor: pointer; font: inherit; font-size: 11px; font-weight: 700; transition: background .2s, transform .2s; }
.button:hover:not(:disabled) { transform: translateY(-1px); }
.button:disabled { cursor: not-allowed; opacity: .5; }
.button-dark { color: white; background: #302833; }
.button-dark:hover:not(:disabled) { background: #8e328f; }
.button-light { border-color: #e5d9e7; color: #55485b; background: white; }
.button-light:hover:not(:disabled) { background: #fbf2fc; }

.admin-heading { display: flex; justify-content: space-between; align-items: end; gap: 20px; padding: 30px 0 24px; }
.eyebrow { margin: 0 0 8px; color: #96789b; font-size: 9px; font-weight: 700; letter-spacing: .14em; }
.admin-heading h1, .modal-header h2 { margin: 0; font-family: 'Space Grotesk', Arial, sans-serif; font-size: clamp(24px, 3vw, 34px); letter-spacing: -.05em; }
.heading-copy { margin: 8px 0 0; color: #887b8b; font-size: 12px; }
.admin-summary { display: flex; gap: 8px; }
.admin-summary span { padding: 8px 10px; border: 1px solid #f0e5f1; border-radius: 10px; color: #786b7b; background: rgba(255,255,255,.7); font-size: 10px; white-space: nowrap; }
.admin-summary strong { color: #8e328f; }

.catalog-toolbar { display: flex; align-items: end; gap: 11px; margin-bottom: 20px; }
.search-box { display: flex; align-items: center; gap: 8px; flex: 1; min-height: 42px; padding: 0 13px; border: 1px solid #ebdfec; border-radius: 11px; color: #927d95; background: white; }
.search-box span { font-size: 22px; line-height: 1; }
.search-box input { width: 100%; border: 0; outline: 0; color: var(--admin-ink); background: transparent; font: inherit; font-size: 12px; }
.search-box input::placeholder { color: #ad9eae; }
.select-filter { display: grid; gap: 5px; color: #8b7c8e; font-size: 9px; font-weight: 700; }
.select-filter select { min-width: 120px; min-height: 42px; padding: 0 10px; border: 1px solid #ebdfec; border-radius: 10px; outline: 0; color: #55485b; background: white; font: inherit; font-size: 11px; }

.product-grid { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 17px; }
.product-card { min-width: 0; overflow: hidden; border: 1px solid #f0e8ef; border-radius: 15px; background: white; box-shadow: 0 5px 16px rgba(106,61,110,.06); transition: transform .2s, box-shadow .2s; }
.product-card:hover { transform: translateY(-3px); box-shadow: 0 12px 26px rgba(106,61,110,.12); }
.product-image { position: relative; display: grid; place-items: center; aspect-ratio: 1.12; overflow: hidden; background: linear-gradient(135deg, #f8e6f4, #eadcf6); }
.product-image img { width: 100%; height: 100%; object-fit: cover; }
.image-placeholder { color: rgba(142,50,143,.55); font-family: 'Space Grotesk', Arial, sans-serif; font-size: 70px; font-weight: 700; }
.status-badge { position: absolute; top: 10px; left: 10px; padding: 5px 8px; border-radius: 999px; color: #237b57; background: #e2f5e9; font-size: 9px; font-weight: 700; }
.status-badge.inactive { color: #85617f; background: #f1e8f2; }
.product-details { padding: 13px 14px 14px; }
.product-category { display: flex; gap: 6px; overflow: hidden; color: #9c8b9e; font-size: 9px; text-overflow: ellipsis; white-space: nowrap; }
.product-category span { color: #d3bfd4; }
.product-details h2 { min-height: 36px; margin: 6px 0 3px; overflow: hidden; font-family: 'Space Grotesk', Arial, sans-serif; font-size: 15px; letter-spacing: -.02em; text-overflow: ellipsis; }
.product-meta { min-height: 14px; margin: 0; overflow: hidden; color: #9c8b9e; font-size: 9px; text-overflow: ellipsis; white-space: nowrap; }
.product-bottom { display: flex; justify-content: space-between; align-items: center; gap: 5px; margin-top: 13px; }
.product-price { color: #392d3e; font-size: 12px; }
.product-stock { color: #6b6170; font-size: 9px; white-space: nowrap; }
.product-stock.low { color: #b04f5b; }
.product-actions { display: flex; gap: 7px; margin-top: 12px; }
.product-actions button { min-height: 27px; padding: 0 10px; border: 1px solid #eaddec; border-radius: 999px; color: #765879; background: white; cursor: pointer; font: inherit; font-size: 9px; font-weight: 700; }
.product-actions button:hover { border-color: #cb65d0; color: #8e328f; background: #fcf3fc; }
.product-actions .action-status { color: #577660; }

.state-message { padding: 18px; border-radius: 12px; color: #756477; background: #f5edf6; font-size: 12px; text-align: center; }
.error-message { color: #9c2a52; background: #fff0f3; }
.empty-state { display: grid; justify-items: center; gap: 8px; padding: 60px 20px; border: 1px dashed #dfc5e1; border-radius: 16px; color: #786b7b; text-align: center; }
.empty-state > span { color: #bd4dc1; font-size: 32px; }
.empty-state h2 { margin: 0; color: #392d3e; font-family: 'Space Grotesk', Arial, sans-serif; font-size: 19px; }
.empty-state p { margin: 0 0 8px; font-size: 12px; }

.modal-backdrop { position: fixed; z-index: 20; inset: 0; display: grid; place-items: center; overflow-y: auto; padding: 24px; background: rgba(40,33,45,.48); }
.product-modal { width: min(100%, 620px); max-height: calc(100vh - 48px); overflow-y: auto; padding: 26px; border: 1px solid rgba(255,255,255,.8); border-radius: 20px; background: #fffafd; box-shadow: 0 24px 80px rgba(40,33,45,.25); }
.modal-header { display: flex; justify-content: space-between; align-items: start; margin-bottom: 20px; }
.modal-header h2 { font-size: 25px; }
.modal-close { width: 32px; height: 32px; border: 0; border-radius: 50%; color: #765879; background: #f3e7f4; cursor: pointer; font-size: 23px; line-height: 1; }
.product-form { display: grid; gap: 13px; }
.product-form label { display: grid; gap: 6px; color: #5b4b60; font-size: 10px; font-weight: 700; }
.product-form input:not([type="checkbox"]):not([type="file"]), .product-form select, .product-form textarea { width: 100%; min-height: 38px; padding: 8px 10px; border: 1px solid #e6d9e8; border-radius: 8px; outline: 0; color: var(--admin-ink); background: white; font: inherit; font-size: 12px; }
.product-form textarea { resize: vertical; }
.product-form input:focus, .product-form select:focus, .product-form textarea:focus { border-color: #cb65d0; box-shadow: 0 0 0 3px rgba(203,101,208,.12); }
.product-form input[type="file"] { color: #726477; font: inherit; font-size: 11px; }
.product-form small { color: #998a9c; font-size: 9px; font-weight: 400; }
.form-pair { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.form-triple { grid-template-columns: repeat(3, 1fr); }
.checkbox-label { display: flex !important; align-items: center; gap: 8px !important; }
.checkbox-label input { accent-color: #bd4dc1; }
.modal-actions { display: flex; justify-content: end; gap: 9px; margin-top: 5px; }

@media (max-width: 1250px) { .product-grid { grid-template-columns: repeat(3, minmax(0, 1fr)); } }
@media (max-width: 950px) {
  .admin-page { grid-template-columns: 205px minmax(0, 1fr); }
  .admin-content { padding-right: 22px; padding-left: 22px; }
  .admin-heading { align-items: start; flex-direction: column; }
}
@media (max-width: 700px) {
  .admin-page { display: block; }
  .admin-sidebar { position: static; min-height: auto; padding: 12px 16px; }
  .admin-user, .admin-nav { display: none; }
  .logout-button { position: absolute; top: 8px; right: 12px; margin: 0; }
  .admin-content { padding: 0 16px 30px; }
  .admin-topbar { align-items: start; flex-direction: column; gap: 12px; padding: 15px 0; }
  .admin-heading { padding-top: 23px; }
  .admin-summary { flex-wrap: wrap; }
  .catalog-toolbar { align-items: stretch; flex-wrap: wrap; }
  .search-box { flex-basis: 100%; }
  .select-filter { flex: 1; }
  .select-filter select { width: 100%; }
  .product-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 11px; }
}
@media (max-width: 440px) {
  .product-grid { grid-template-columns: 1fr; }
  .product-image { aspect-ratio: 1.35; }
  .topbar-actions { width: 100%; }
  .topbar-actions .button { flex: 1; }
  .form-pair, .form-triple { grid-template-columns: 1fr; }
  .product-modal { padding: 20px 16px; }
}
</style>
