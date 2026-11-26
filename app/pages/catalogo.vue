<script setup lang="ts">
interface ProductoCatalogo {
  id_producto: number
  codigo: string
  nombre: string
  descripcion: string | null
  precio_base: string
  imagen_url: string | null
  disponible: boolean
}

interface UsuarioSesion {
  id: number
  role: string
}

interface LineaCarrito {
  id_producto: number
  cantidad: number
}

interface LineaCarritoVista extends LineaCarrito {
  producto: ProductoCatalogo | undefined
}

const config = useRuntimeConfig()
const productos = ref<ProductoCatalogo[]>([])
const carrito = ref<LineaCarrito[]>([])
const usuario = ref<UsuarioSesion | null>(null)
const busqueda = ref('')
const direccionEntrega = ref('')
const observaciones = ref('')
const cargando = ref(true)
const procesandoPedido = ref(false)
const errorCarga = ref('')
const mensaje = ref('')
const pedidoCreado = ref('')
let temporizadorActualizacion: ReturnType<typeof setInterval> | undefined
let solicitudPendiente = false

const esCliente = computed(() => usuario.value?.role.trim().toLowerCase() === 'cliente')
const claveCarrito = computed(() => usuario.value ? `danstamp:carrito:${usuario.value.id}` : '')
const productosFiltrados = computed(() => {
  const consulta = busqueda.value.trim().toLocaleLowerCase()
  if (!consulta) return productos.value
  return productos.value.filter((producto) =>
    producto.nombre.toLocaleLowerCase().includes(consulta)
    || producto.codigo.toLocaleLowerCase().includes(consulta),
  )
})
const lineasCarrito = computed<LineaCarritoVista[]>(() =>
  carrito.value.map((linea) => ({
    ...linea,
    producto: productos.value.find((producto) => producto.id_producto === linea.id_producto),
  })),
)
const unidadesCarrito = computed(() =>
  carrito.value.reduce((total, linea) => total + linea.cantidad, 0),
)
const subtotal = computed(() =>
  lineasCarrito.value.reduce((total, linea) => {
    if (!linea.producto) return total
    return total + Number(linea.producto.precio_base) * linea.cantidad
  }, 0),
)
const checkoutDeshabilitado = computed(() =>
  !esCliente.value
  || !lineasCarrito.value.length
  || lineasCarrito.value.some((linea) => !linea.producto || !linea.producto.disponible)
  || !direccionEntrega.value.trim()
  || procesandoPedido.value,
)

function formatoPrecio(precio: string | number): string {
  const valor = Number(precio)
  if (!Number.isFinite(valor)) return 'Precio no disponible'
  return new Intl.NumberFormat('es-CO', {
    style: 'currency',
    currency: 'COP',
    maximumFractionDigits: 0,
  }).format(valor)
}

function mensajeDeError(error: unknown): string {
  const fetchError = error as {
    statusCode?: number
    data?: { message?: string | string[] }
  }
  const mensajeBackend = fetchError.data?.message
  if (Array.isArray(mensajeBackend)) return mensajeBackend.join('. ')
  if (mensajeBackend) return mensajeBackend
  if (fetchError.statusCode === 401) return 'Tu sesión venció. Inicia sesión nuevamente para continuar.'
  return 'No fue posible completar la solicitud. Intenta de nuevo.'
}

async function cargarProductos(silencioso = false): Promise<void> {
  if (solicitudPendiente) return
  solicitudPendiente = true
  if (!silencioso) {
    cargando.value = true
    errorCarga.value = ''
  }

  try {
    productos.value = await $fetch<ProductoCatalogo[]>('/productos', {
      baseURL: config.public.apiBase,
    })
    errorCarga.value = ''
  } catch (error: unknown) {
    errorCarga.value = mensajeDeError(error)
  } finally {
    solicitudPendiente = false
    cargando.value = false
  }
}

function guardarCarrito(): void {
  if (!claveCarrito.value) return
  try {
    localStorage.setItem(claveCarrito.value, JSON.stringify(carrito.value))
  } catch {
    mensaje.value = 'No se pudo guardar el carrito en este navegador. Puedes continuar con esta pestaña abierta.'
  }
}

function cargarCarritoGuardado(): void {
  if (!claveCarrito.value) return
  const guardado = localStorage.getItem(claveCarrito.value)
  if (!guardado) return

  try {
    const lineas = JSON.parse(guardado) as unknown
    if (!Array.isArray(lineas)) throw new Error('Formato de carrito inválido')
    carrito.value = lineas.filter((linea): linea is LineaCarrito =>
      Number.isInteger(linea?.id_producto)
      && linea.id_producto > 0
      && Number.isInteger(linea?.cantidad)
      && linea.cantidad > 0,
    )
  } catch {
    localStorage.removeItem(claveCarrito.value)
    mensaje.value = 'El carrito guardado no era válido y se limpió. Puedes volver a agregar productos.'
  }
}

function agregarAlCarrito(producto: ProductoCatalogo): void {
  mensaje.value = ''
  pedidoCreado.value = ''

  if (!usuario.value) {
    void navigateTo('/login?redirect=/catalogo')
    return
  }
  if (!esCliente.value) {
    mensaje.value = 'Solo las cuentas de cliente pueden agregar productos al carrito.'
    return
  }
  if (!producto.disponible) return

  const existente = carrito.value.find((linea) => linea.id_producto === producto.id_producto)
  if (existente) existente.cantidad += 1
  else carrito.value.push({ id_producto: producto.id_producto, cantidad: 1 })
  guardarCarrito()
}

function cambiarCantidad(idProducto: number, cambio: number): void {
  const linea = carrito.value.find((item) => item.id_producto === idProducto)
  if (!linea) return
  linea.cantidad += cambio
  if (linea.cantidad < 1) {
    carrito.value = carrito.value.filter((item) => item.id_producto !== idProducto)
  }
  guardarCarrito()
}

function quitarDelCarrito(idProducto: number): void {
  carrito.value = carrito.value.filter((linea) => linea.id_producto !== idProducto)
  guardarCarrito()
}

async function crearPedido(): Promise<void> {
  mensaje.value = ''
  pedidoCreado.value = ''

  if (!esCliente.value) {
    mensaje.value = 'Inicia sesión con una cuenta de cliente para realizar el pedido.'
    return
  }
  const token = localStorage.getItem('authToken')
  if (!token) {
    mensaje.value = 'Tu sesión venció. Inicia sesión nuevamente para continuar.'
    return
  }

  procesandoPedido.value = true
  try {
    const pedido = await $fetch<{ id_pedido: number; total: string }>('/pedidos/checkout', {
      baseURL: config.public.apiBase,
      method: 'POST',
      headers: { Authorization: `Bearer ${token}` },
      body: {
        direccion_entrega: direccionEntrega.value.trim(),
        ...(observaciones.value.trim() ? { observaciones: observaciones.value.trim() } : {}),
        items: carrito.value.map(({ id_producto, cantidad }) => ({ id_producto, cantidad })),
      },
    })
    pedidoCreado.value = `Pedido #${pedido.id_pedido} recibido. Total: ${formatoPrecio(pedido.total)}.`
    carrito.value = []
    guardarCarrito()
    observaciones.value = ''
    await cargarProductos(true)
  } catch (error: unknown) {
    mensaje.value = mensajeDeError(error)
  } finally {
    procesandoPedido.value = false
  }
}

function actualizarAlVolver(): void {
  void cargarProductos(true)
}

onMounted(() => {
  try {
    const storedUser = localStorage.getItem('authUser')
    if (storedUser) {
      const parsed = JSON.parse(storedUser) as Partial<UsuarioSesion>
      if (typeof parsed.id === 'number' && typeof parsed.role === 'string') {
        usuario.value = { id: parsed.id, role: parsed.role }
      } else {
        mensaje.value = 'No se pudo identificar tu sesión. Inicia sesión para comprar.'
      }
    }
    cargarCarritoGuardado()
  } catch {
    mensaje.value = 'No se pudo leer la sesión guardada. Inicia sesión para comprar.'
  }

  void cargarProductos()
  temporizadorActualizacion = setInterval(() => {
    void cargarProductos(true)
  }, 30_000)
  window.addEventListener('focus', actualizarAlVolver)
})

onUnmounted(() => {
  if (temporizadorActualizacion) clearInterval(temporizadorActualizacion)
  window.removeEventListener('focus', actualizarAlVolver)
})
</script>

<template>
  <main class="catalogo-cliente">
    <header class="catalogo-header">
      <NuxtLink to="/principal" class="marca" aria-label="Ir al inicio"><BrandLogo size="large" /></NuxtLink>
      <nav aria-label="Navegación principal">
        <NuxtLink to="/principal">Inicio</NuxtLink>
        <a href="#productos" aria-current="page">Catálogo</a>
        <a href="#carrito">Carrito <span class="contador-carrito">{{ unidadesCarrito }}</span></a>
      </nav>
      <NuxtLink v-if="!esCliente" to="/login?redirect=/catalogo" class="boton boton-oscuro">Iniciar sesión</NuxtLink>
      <span v-else class="saludo-cliente">Compra segura · DAN'STAMP</span>
    </header>

    <section class="catalogo-portada">
      <p class="etiqueta">CATÁLOGO DAN'STAMP</p>
      <h1>Encuentra tu<br /><em>próxima favorita.</em></h1>
      <p>Productos disponibles del inventario, listos para pedir.</p>
      <span class="actualizacion">El catálogo se actualiza automáticamente cada 30 segundos.</span>
    </section>

    <div class="catalogo-contenido">
      <section id="productos" class="productos-seccion" aria-labelledby="productos-titulo">
        <div class="encabezado-productos">
          <div>
            <p class="etiqueta">HECHOS PARA TI</p>
            <h2 id="productos-titulo">Productos</h2>
          </div>
          <label class="busqueda">
            <span class="solo-lectores">Buscar productos</span>
            <input v-model="busqueda" type="search" placeholder="Buscar por nombre o código" />
          </label>
        </div>

        <p v-if="errorCarga" class="aviso error" role="alert">{{ errorCarga }}</p>
        <p v-if="mensaje" class="aviso" role="status">{{ mensaje }}</p>
        <div v-if="cargando" class="estado-catalogo">Cargando productos...</div>
        <div v-else-if="!productosFiltrados.length" class="estado-catalogo">
          {{ productos.length ? 'No encontramos productos con esa búsqueda.' : 'Aún no hay productos disponibles.' }}
        </div>
        <div v-else class="rejilla-productos">
          <article v-for="producto in productosFiltrados" :key="producto.id_producto" class="tarjeta-producto">
            <div class="imagen-producto">
              <img v-if="producto.imagen_url" :src="producto.imagen_url" :alt="producto.nombre" loading="lazy" />
              <span v-else class="imagen-vacia" aria-hidden="true">D</span>
              <span class="insignia-stock" :class="{ agotado: !producto.disponible }">
                {{ producto.disponible ? 'Disponible' : 'Agotado' }}
              </span>
            </div>
            <div class="informacion-producto">
              <p class="codigo-producto">{{ producto.codigo }}</p>
              <h3>{{ producto.nombre }}</h3>
              <p v-if="producto.descripcion" class="descripcion-producto">{{ producto.descripcion }}</p>
              <div class="pie-producto">
                <strong>{{ formatoPrecio(producto.precio_base) }}</strong>
                <button
                  type="button"
                  class="boton boton-agregar"
                  :disabled="!producto.disponible || (usuario && !esCliente)"
                  @click="agregarAlCarrito(producto)"
                >
                  {{ esCliente ? 'Agregar al carrito' : usuario ? 'Solo clientes' : 'Inicia sesión para comprar' }}
                </button>
              </div>
            </div>
          </article>
        </div>
      </section>

      <aside id="carrito" class="panel-carrito" aria-labelledby="carrito-titulo">
        <div class="titulo-carrito">
          <div>
            <p class="etiqueta">TU COMPRA</p>
            <h2 id="carrito-titulo">Carrito</h2>
          </div>
          <span class="contador-carrito contador-panel">{{ unidadesCarrito }}</span>
        </div>

        <p v-if="!esCliente" class="nota-carrito">Inicia sesión como cliente para guardar productos y realizar un pedido.</p>
        <div v-if="!lineasCarrito.length" class="carrito-vacio">Tu carrito está vacío.</div>
        <div v-else class="lineas-carrito">
          <article v-for="linea in lineasCarrito" :key="linea.id_producto" class="linea-carrito">
            <img v-if="linea.producto?.imagen_url" :src="linea.producto.imagen_url" :alt="linea.producto.nombre" />
            <div class="detalle-linea">
              <strong>{{ linea.producto?.nombre ?? 'Producto ya no disponible' }}</strong>
              <span v-if="linea.producto">{{ formatoPrecio(linea.producto.precio_base) }}</span>
              <span v-else class="texto-agotado">Retíralo para continuar.</span>
              <span v-if="linea.producto && !linea.producto.disponible" class="texto-agotado">Agotado temporalmente.</span>
              <div class="controles-cantidad" :aria-label="`Cantidad: ${linea.cantidad}`">
                <button type="button" :aria-label="`Reducir cantidad de ${linea.producto?.nombre ?? 'producto'}`" @click="cambiarCantidad(linea.id_producto, -1)">−</button>
                <span>{{ linea.cantidad }}</span>
                <button type="button" :aria-label="`Aumentar cantidad de ${linea.producto?.nombre ?? 'producto'}`" @click="cambiarCantidad(linea.id_producto, 1)">+</button>
                <button type="button" class="quitar-linea" @click="quitarDelCarrito(linea.id_producto)">Quitar</button>
              </div>
            </div>
          </article>
        </div>

        <div class="total-carrito">
          <span>Subtotal</span>
          <strong>{{ formatoPrecio(subtotal) }}</strong>
        </div>
        <p class="nota-envio">El costo de envío se confirmará al coordinar la entrega.</p>

        <form v-if="esCliente" class="formulario-checkout" @submit.prevent="crearPedido">
          <label for="direccion-entrega">Dirección de entrega</label>
          <input
            id="direccion-entrega"
            v-model="direccionEntrega"
            type="text"
            maxlength="200"
            autocomplete="street-address"
            placeholder="Dirección completa"
            required
          />
          <label for="observaciones">Observaciones <span>(opcional)</span></label>
          <textarea id="observaciones" v-model="observaciones" rows="3" placeholder="Indicaciones para tu pedido"></textarea>
          <p v-if="mensaje" class="aviso error" role="alert">{{ mensaje }}</p>
          <p v-if="pedidoCreado" class="aviso exito" role="status">{{ pedidoCreado }}</p>
          <button type="submit" class="boton boton-pedido" :disabled="checkoutDeshabilitado">
            {{ procesandoPedido ? 'Enviando pedido...' : 'Realizar pedido' }}
          </button>
        </form>
        <NuxtLink v-else to="/login?redirect=/catalogo" class="boton boton-pedido boton-login">
          Iniciar sesión para comprar
        </NuxtLink>
      </aside>
    </div>
  </main>
</template>

<style scoped>
.catalogo-cliente {
  --tinta: #211b20;
  --rosa: #ec4d9d;
  --rosa-claro: #f7d8e9;
  --crema: #fbf8f3;
  min-height: 100vh;
  color: var(--tinta);
  background: var(--crema);
  font-family: 'DM Sans', sans-serif;
}

.catalogo-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 78px;
  padding: 12px 5vw;
  color: #fff;
  background: var(--tinta);
}

.marca { display: inline-flex; align-items: center; }
.catalogo-header nav { display: flex; align-items: center; gap: 30px; font-size: 13px; font-weight: 700; }
.catalogo-header nav a:hover { color: #ff9fce; }
.contador-carrito { display: inline-grid; min-width: 22px; height: 22px; place-items: center; margin-left: 4px; padding: 0 5px; border-radius: 50%; color: var(--tinta); background: #ff9fce; font-size: 11px; }
.saludo-cliente { color: #f4c7df; font-size: 12px; font-weight: 700; }
.boton { display: inline-flex; justify-content: center; align-items: center; border: 0; border-radius: 999px; padding: 12px 17px; font: inherit; font-size: 12px; font-weight: 700; text-decoration: none; cursor: pointer; transition: transform .2s, background .2s; }
.boton:hover:not(:disabled) { transform: translateY(-2px); }
.boton-oscuro { color: var(--tinta); background: #fff; }

.catalogo-portada { padding: 72px 8vw 68px; background: var(--rosa-claro); }
.etiqueta { margin: 0 0 13px; color: #76646f; font-size: 10px; font-weight: 700; letter-spacing: .16em; }
.catalogo-portada h1 { margin: 0; font-family: 'Space Grotesk', sans-serif; font-size: clamp(2.7rem, 6vw, 5.2rem); letter-spacing: -.07em; line-height: .96; }
.catalogo-portada h1 em { color: var(--rosa); font-style: normal; }
.catalogo-portada > p:not(.etiqueta) { margin: 20px 0 8px; color: #564851; }
.actualizacion { color: #76646f; font-size: 11px; }

.catalogo-contenido { display: grid; grid-template-columns: minmax(0, 1fr) 340px; align-items: start; gap: 32px; max-width: 1440px; margin: auto; padding: 54px 5vw 90px; }
.encabezado-productos, .titulo-carrito { display: flex; align-items: end; justify-content: space-between; gap: 20px; margin-bottom: 24px; }
.encabezado-productos h2, .titulo-carrito h2 { margin: 0; font-family: 'Space Grotesk', sans-serif; font-size: 32px; letter-spacing: -.06em; }
.busqueda input { width: min(300px, 42vw); min-height: 42px; border: 1px solid #ded5db; border-radius: 22px; padding: 0 16px; color: var(--tinta); background: #fff; font: inherit; font-size: 12px; }
.rejilla-productos { display: grid; grid-template-columns: repeat(auto-fill, minmax(225px, 1fr)); gap: 18px; }
.tarjeta-producto { overflow: hidden; border: 1px solid #eee6eb; border-radius: 8px; background: #fff; box-shadow: 0 8px 24px rgba(48, 29, 42, .04); }
.imagen-producto { position: relative; aspect-ratio: .86; overflow: hidden; background: #f2e7ed; }
.imagen-producto img { width: 100%; height: 100%; object-fit: cover; transition: transform .35s; }
.tarjeta-producto:hover .imagen-producto img { transform: scale(1.035); }
.imagen-vacia { display: grid; width: 100%; height: 100%; place-items: center; color: #c65b91; font: 700 64px 'Space Grotesk', sans-serif; }
.insignia-stock { position: absolute; top: 12px; left: 12px; border-radius: 20px; padding: 7px 10px; color: #235a3d; background: #d7f0df; font-size: 10px; font-weight: 700; }
.insignia-stock.agotado { color: #8a3333; background: #f9dcdc; }
.informacion-producto { padding: 16px; }
.codigo-producto { margin: 0 0 7px; color: #8a7b84; font-size: 10px; letter-spacing: .08em; }
.informacion-producto h3 { min-height: 42px; margin: 0; font-family: 'Space Grotesk', sans-serif; font-size: 17px; line-height: 1.2; }
.descripcion-producto { min-height: 32px; margin: 9px 0 0; color: #746870; font-size: 11px; line-height: 1.45; }
.pie-producto { display: grid; gap: 13px; margin-top: 16px; }
.pie-producto strong { font-family: 'Space Grotesk', sans-serif; font-size: 20px; }
.boton-agregar { min-height: 41px; color: #fff; background: var(--tinta); }
.boton-agregar:hover:not(:disabled), .boton-pedido:hover:not(:disabled) { background: #a93171; }
.boton-agregar:disabled, .boton-pedido:disabled { opacity: .48; cursor: not-allowed; }
.estado-catalogo { padding: 35px 20px; border: 1px dashed #d4c6cf; border-radius: 8px; color: #6f6069; text-align: center; }
.aviso { margin: 12px 0; border-radius: 7px; padding: 11px 13px; color: #553b4a; background: #f7e6ef; font-size: 12px; line-height: 1.45; }
.aviso.error { color: #842e38; background: #fbe4e4; }
.aviso.exito { color: #245c3e; background: #dff3e5; }

.panel-carrito { position: sticky; top: 18px; border: 1px solid #eee6eb; border-radius: 9px; padding: 22px; background: #fff; box-shadow: 0 8px 24px rgba(48, 29, 42, .05); }
.titulo-carrito { align-items: center; margin-bottom: 12px; }
.contador-panel { width: 30px; height: 30px; font-size: 12px; }
.nota-carrito, .nota-envio { color: #766870; font-size: 11px; line-height: 1.5; }
.carrito-vacio { padding: 23px 0; color: #766870; font-size: 12px; }
.lineas-carrito { max-height: 350px; overflow-y: auto; }
.linea-carrito { display: grid; grid-template-columns: 58px minmax(0, 1fr); gap: 12px; padding: 14px 0; border-top: 1px solid #f0e9ed; }
.linea-carrito > img { width: 58px; height: 68px; border-radius: 4px; object-fit: cover; }
.detalle-linea { display: grid; align-content: start; gap: 5px; min-width: 0; font-size: 11px; }
.detalle-linea strong { overflow: hidden; font-size: 12px; text-overflow: ellipsis; white-space: nowrap; }
.texto-agotado { color: #9c3d3d; font-size: 10px; }
.controles-cantidad { display: flex; align-items: center; gap: 9px; margin-top: 5px; }
.controles-cantidad button { width: 25px; height: 25px; border: 1px solid #e8dce3; border-radius: 50%; color: var(--tinta); background: #fff; cursor: pointer; }
.controles-cantidad .quitar-linea { width: auto; height: auto; margin-left: auto; border: 0; border-radius: 0; color: #923d66; font-size: 10px; text-decoration: underline; }
.total-carrito { display: flex; justify-content: space-between; margin-top: 13px; border-top: 1px solid #eee6eb; padding-top: 15px; font-size: 13px; }
.total-carrito strong { font-size: 17px; }
.nota-envio { margin: 8px 0 20px; }
.formulario-checkout { display: grid; gap: 9px; }
.formulario-checkout label { margin-top: 7px; font-size: 11px; font-weight: 700; }
.formulario-checkout label span { color: #8a7b84; font-weight: 400; }
.formulario-checkout input, .formulario-checkout textarea { width: 100%; border: 1px solid #ded5db; border-radius: 6px; padding: 11px; color: var(--tinta); background: #fff; font: inherit; font-size: 12px; }
.formulario-checkout textarea { resize: vertical; }
.boton-pedido { width: 100%; min-height: 44px; margin-top: 9px; color: #fff; background: var(--rosa); }
.boton-login { margin-top: 12px; }
.solo-lectores { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; }

@media (max-width: 900px) {
  .catalogo-contenido { grid-template-columns: minmax(0, 1fr) 300px; gap: 20px; padding-right: 4vw; padding-left: 4vw; }
  .rejilla-productos { grid-template-columns: repeat(auto-fill, minmax(190px, 1fr)); }
}

@media (max-width: 700px) {
  .catalogo-header { min-height: 68px; padding: 10px 18px; }
  .catalogo-header nav { gap: 12px; font-size: 11px; }
  .catalogo-header nav a:first-child, .saludo-cliente { display: none; }
  .catalogo-portada { padding: 54px 24px; }
  .catalogo-contenido { display: flex; flex-direction: column; padding: 36px 18px 60px; }
  .productos-seccion, .panel-carrito { width: 100%; }
  .panel-carrito { position: static; order: -1; }
  .encabezado-productos { align-items: flex-start; flex-direction: column; }
  .busqueda, .busqueda input { width: 100%; }
  .rejilla-productos { grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; }
  .informacion-producto { padding: 12px; }
  .informacion-producto h3 { font-size: 14px; }
  .pie-producto strong { font-size: 17px; }
  .boton-agregar { padding: 10px 8px; font-size: 10px; }
}
</style>
