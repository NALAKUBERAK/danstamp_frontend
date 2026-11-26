<script setup lang="ts">
interface TipoDocumento {
  id_tipo_documento: number
  nombre: string
}

interface RespuestaAuth {
  user: unknown
  token: string
}

const config = useRuntimeConfig()
const tiposDocumento = ref<TipoDocumento[]>([])
const cargandoTipos = ref(true)
const cargando = ref(false)
const errorTipos = ref('')
const mensaje = ref('')
const formulario = reactive({
  nombres: '',
  apellidos: '',
  email: '',
  password: '',
  id_tipo_documento: '',
  numero_documento: '',
  telefono: '',
  direccion: '',
})

onMounted(async () => {
  try {
    tiposDocumento.value = await $fetch<TipoDocumento[]>('/tipos-documento', {
      baseURL: config.public.apiBase,
    })
  } catch {
    errorTipos.value = 'No se pudieron cargar los tipos de documento. Recarga la página para intentarlo de nuevo.'
  } finally {
    cargandoTipos.value = false
  }
})

async function registrar() {
  if (cargandoTipos.value || !tiposDocumento.value.length) {
    mensaje.value = errorTipos.value || 'No hay tipos de documento disponibles para el registro.'
    return
  }

  cargando.value = true
  mensaje.value = ''

  try {
    await $fetch<RespuestaAuth>('/auth/register', {
      baseURL: config.public.apiBase,
      method: 'POST',
      body: {
        nombres: formulario.nombres.trim(),
        apellidos: formulario.apellidos.trim(),
        email: formulario.email.trim(),
        password: formulario.password,
        id_tipo_documento: Number(formulario.id_tipo_documento),
        numero_documento: Number(formulario.numero_documento),
        telefono: Number(formulario.telefono),
        direccion: formulario.direccion.trim(),
      },
    })

    await navigateTo('/login?registro=exitoso', { replace: true })
  } catch (error: unknown) {
    const fetchError = error as {
      statusCode?: number
      data?: { message?: string | string[] }
    }
    const backendMessage = fetchError.data?.message

    if (Array.isArray(backendMessage)) {
      mensaje.value = backendMessage.join('. ')
    } else if (backendMessage) {
      mensaje.value = backendMessage
    } else if (fetchError.statusCode === 429) {
      mensaje.value = 'Demasiados intentos. Espera unos minutos antes de volver a intentar.'
    } else {
      mensaje.value = 'No fue posible conectar con el servidor.'
    }
  } finally {
    cargando.value = false
  }
}
</script>

<template>
  <main class="registro-page">
    <div class="registro-topbar" aria-hidden="true"></div>
    <NuxtLink to="/" class="registro-marca" aria-label="Volver al inicio">
      <BrandLogo size="large" />
    </NuxtLink>

    <section class="registro-layout" aria-labelledby="registro-title">
      <header class="registro-intro">
        <p class="registro-eyebrow">ACCESO DE CLIENTES</p>
        <h1>Tu cuenta,<br /><span>a tu manera.</span></h1>
        <p>Regístrate para empezar a crear y dar seguimiento a tus pedidos.</p>
        <div class="registro-sello" aria-hidden="true">✦</div>
      </header>

      <div class="registro-card">
        <div class="registro-card-heading">
          <div>
            <p class="registro-eyebrow">BIENVENIDO/A</p>
            <h2 id="registro-title">Crear cuenta</h2>
          </div>
          <span class="registro-icono" aria-hidden="true">＋</span>
        </div>

        <form class="registro-form" @submit.prevent="registrar">
          <div class="registro-campo-par">
            <div class="registro-campo">
              <label for="nombres">Nombres</label>
              <input id="nombres" v-model="formulario.nombres" autocomplete="given-name" required />
            </div>
            <div class="registro-campo">
              <label for="apellidos">Apellidos</label>
              <input id="apellidos" v-model="formulario.apellidos" autocomplete="family-name" required />
            </div>
          </div>

          <div class="registro-campo">
            <label for="email">Usuario / correo electrónico</label>
            <input id="email" v-model="formulario.email" type="email" autocomplete="email" placeholder="tu@correo.com" required />
          </div>

          <div class="registro-campo">
            <label for="password">Contraseña</label>
            <input
              id="password"
              v-model="formulario.password"
              type="password"
              autocomplete="new-password"
              minlength="8"
              maxlength="128"
              placeholder="Crea una contraseña segura"
              required
            />
            <small>Mínimo 8 caracteres, con mayúscula, minúscula, número y símbolo.</small>
          </div>

          <div class="registro-campo-par registro-documento">
            <div class="registro-campo">
              <label for="tipo-documento">Tipo de documento</label>
              <select id="tipo-documento" v-model="formulario.id_tipo_documento" :disabled="cargandoTipos || !tiposDocumento.length" required>
                <option value="" disabled>{{ cargandoTipos ? 'Cargando...' : 'Selecciona' }}</option>
                <option v-for="tipo in tiposDocumento" :key="tipo.id_tipo_documento" :value="String(tipo.id_tipo_documento)">
                  {{ tipo.nombre }}
                </option>
              </select>
            </div>
            <div class="registro-campo">
              <label for="numero-documento">Número de documento</label>
              <input id="numero-documento" v-model="formulario.numero_documento" type="number" min="1" step="1" inputmode="numeric" required />
            </div>
          </div>

          <div class="registro-campo-par">
            <div class="registro-campo">
              <label for="telefono">Teléfono</label>
              <input id="telefono" v-model="formulario.telefono" type="number" min="1" step="1" inputmode="tel" autocomplete="tel" required />
            </div>
            <div class="registro-campo">
              <label for="direccion">Dirección</label>
              <input id="direccion" v-model="formulario.direccion" autocomplete="street-address" required />
            </div>
          </div>

          <p v-if="errorTipos" class="registro-mensaje" role="alert">{{ errorTipos }}</p>
          <p v-if="mensaje" class="registro-mensaje" role="alert">{{ mensaje }}</p>

          <button class="registro-submit" type="submit" :disabled="cargando || cargandoTipos || !tiposDocumento.length">
            {{ cargando ? 'Creando cuenta...' : 'Crear mi cuenta' }}
            <span aria-hidden="true">↗</span>
          </button>
        </form>

        <p class="registro-login">¿Ya tienes una cuenta? <NuxtLink to="/login">Inicia sesión</NuxtLink></p>
      </div>
    </section>
  </main>
</template>

<style scoped>
.registro-page {
  --registro-rosa: #e56ddd;
  --registro-rosa-claro: #f3c7ef;
  --registro-lila: #dfc5f5;
  --registro-tinta: #28212d;
  min-height: 100vh;
  padding: 54px 7vw 64px;
  overflow: hidden;
  position: relative;
  color: var(--registro-tinta);
  background: linear-gradient(125deg, #f8d7f0 0%, #e8d6f7 48%, #fff4f8 100%);
  font-family: 'DM Sans', Arial, sans-serif;
}

.registro-topbar {
  position: absolute;
  inset: 0 0 auto;
  height: 10px;
  background: var(--registro-rosa);
}

.registro-marca {
  position: absolute;
  top: 28px;
  left: 7vw;
  display: block;
  text-decoration: none;
}

.registro-layout {
  display: grid;
  grid-template-columns: minmax(240px, .75fr) minmax(420px, 1fr);
  align-items: center;
  gap: clamp(36px, 8vw, 120px);
  max-width: 1080px;
  min-height: calc(100vh - 118px);
  margin: 0 auto;
}

.registro-intro {
  position: relative;
  padding-bottom: 48px;
}

.registro-eyebrow {
  margin: 0 0 18px;
  color: #765879;
  font-size: 10px;
  font-weight: 700;
  letter-spacing: .16em;
}

.registro-intro h1 {
  margin: 0;
  font-family: 'Space Grotesk', Arial, sans-serif;
  font-size: clamp(2.8rem, 5vw, 4.7rem);
  letter-spacing: -.07em;
  line-height: .96;
}

.registro-intro h1 span {
  color: #b941b9;
}

.registro-intro > p:last-of-type {
  max-width: 330px;
  margin: 24px 0 0;
  color: #685b6b;
  font-size: 14px;
  line-height: 1.65;
}

.registro-sello {
  display: grid;
  place-items: center;
  width: 70px;
  height: 70px;
  margin-top: 40px;
  border: 1px solid rgba(126, 67, 133, .25);
  border-radius: 50%;
  color: #a33ca6;
  font-size: 28px;
}

.registro-card {
  padding: clamp(28px, 4vw, 48px);
  border: 1px solid rgba(255, 255, 255, .8);
  border-radius: 22px;
  background: rgba(255, 255, 255, .8);
  box-shadow: 0 24px 70px rgba(106, 61, 110, .13);
  backdrop-filter: blur(12px);
}

.registro-card-heading {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 26px;
}

.registro-card-heading .registro-eyebrow {
  margin-bottom: 8px;
}

.registro-card-heading h2 {
  margin: 0;
  font-family: 'Space Grotesk', Arial, sans-serif;
  font-size: 30px;
  letter-spacing: -.05em;
}

.registro-icono {
  display: grid;
  place-items: center;
  width: 42px;
  height: 42px;
  border-radius: 50%;
  color: white;
  background: var(--registro-rosa);
  font-size: 25px;
}

.registro-form {
  display: grid;
  gap: 16px;
}

.registro-campo-par {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 14px;
}

.registro-campo {
  display: grid;
  gap: 7px;
  min-width: 0;
}

.registro-campo label {
  color: #46384a;
  font-size: 12px;
  font-weight: 700;
}

.registro-campo input,
.registro-campo select {
  width: 100%;
  height: 44px;
  padding: 0 13px;
  border: 1px solid #e6d9e8;
  border-radius: 10px;
  outline: none;
  color: var(--registro-tinta);
  background: #fff;
  font: inherit;
  font-size: 13px;
  transition: border-color .2s, box-shadow .2s;
}

.registro-campo input:focus,
.registro-campo select:focus {
  border-color: #cb65d0;
  box-shadow: 0 0 0 3px rgba(203, 101, 208, .15);
}

.registro-campo input::placeholder {
  color: #a89caa;
}

.registro-campo select:disabled {
  color: #877c88;
  background: #f7f3f8;
}

.registro-campo small {
  color: #766b78;
  font-size: 10px;
  line-height: 1.4;
}

.registro-mensaje {
  margin: 0;
  color: #9c2a52;
  font-size: 12px;
  line-height: 1.5;
}

.registro-submit {
  display: flex;
  justify-content: space-between;
  align-items: center;
  min-height: 48px;
  margin-top: 4px;
  padding: 0 18px;
  border: 0;
  border-radius: 12px;
  color: white;
  background: #bd4dc1;
  cursor: pointer;
  font: inherit;
  font-size: 13px;
  font-weight: 700;
  transition: background .2s, transform .2s;
}

.registro-submit:hover:not(:disabled) {
  background: #a83caf;
  transform: translateY(-1px);
}

.registro-submit:disabled {
  cursor: wait;
  opacity: .65;
}

.registro-submit span {
  font-size: 19px;
}

.registro-login {
  margin: 22px 0 0;
  color: #766b78;
  text-align: center;
  font-size: 12px;
}

.registro-login a {
  color: #9f38a8;
  font-weight: 700;
  text-decoration: none;
}

@media (max-width: 760px) {
  .registro-page {
    padding: 76px 20px 36px;
  }

  .registro-marca {
    left: 20px;
  }

  .registro-layout {
    grid-template-columns: 1fr;
    gap: 24px;
    max-width: 520px;
    min-height: auto;
  }

  .registro-intro {
    padding: 0;
  }

  .registro-intro h1 {
    font-size: clamp(2.6rem, 11vw, 3.8rem);
  }

  .registro-intro > p:last-of-type {
    margin-top: 14px;
  }

  .registro-sello {
    display: none;
  }
}

@media (max-width: 480px) {
  .registro-card {
    padding: 24px 18px;
  }

  .registro-campo-par {
    grid-template-columns: 1fr;
  }
}
</style>
