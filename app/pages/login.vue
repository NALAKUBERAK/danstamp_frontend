<script setup lang="ts">
const email = ref('')
const password = ref('')
const route = useRoute()
const mensaje = ref(route.query.registro === 'exitoso' ? 'Cuenta creada correctamente. Inicia sesión para continuar.' : '')
const cargando = ref(false)
const config = useRuntimeConfig()

async function iniciarSesion() {
  if (!email.value || !password.value) {
    mensaje.value = 'Completa tu correo y contraseña para continuar.'
    return
  }

  cargando.value = true
  mensaje.value = ''

  try {
    const respuesta = await $fetch<{
      user: { id: number; email: string; fullName: string; role: string }
      token: string
    }>(
      '/auth/login',
      {
        baseURL: config.public.apiBase,
        method: 'POST',
        body: {
          email: email.value.trim(),
          password: password.value,
        },
      },
    )

    const rutasPorRol: Record<string, string> = {
      admin: '/admin',
      cliente: '/principal',
      empleado: '/empleado',
    }
    const rol = respuesta.user.role.trim().toLowerCase()
    const ruta = rutasPorRol[rol]

    if (!ruta) {
      mensaje.value = 'El tipo de usuario de esta cuenta no tiene una vista asignada.'
      return
    }

    localStorage.setItem('authToken', respuesta.token)
    localStorage.setItem('authUser', JSON.stringify(respuesta.user))
    await navigateTo(ruta, { replace: true })
  } catch (error: unknown) {
    const fetchError = error as {
      statusCode?: number
      statusMessage?: string
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
  <main class="login-page">
    <section class="login-copy" aria-label="Mensaje de bienvenida">
      <span>DAN’STAMP</span>
      <h2>Acceso Seguro</h2>
      <p>Manten el control.</p>
    </section>
    <NuxtLink to="/principal" class="boton-inicio" aria-label="Regresar al inicio">Regresar al inicio ←</NuxtLink>
    <section class="login-panel" aria-labelledby="login-title">
      <div class="login-cabecera">
        <h1 id="login-title">BIENVENIDOS</h1>
      </div>

      <form class="login-formulario" @submit.prevent="iniciarSesion">
        <label for="email">Usuario</label>
        <input id="email" v-model="email" type="email" autocomplete="email" placeholder="Ingrese su usuario" required />

        <label for="password">Contraseña</label>
        <input id="password" v-model="password" type="password" autocomplete="current-password" placeholder="Ingrese su contraseña" required />

        <div class="acciones-login">
          <button type="submit" :disabled="cargando">{{ cargando ? 'Ingresando...' : 'Ingresar' }}</button>
          <NuxtLink to="/registro" class="boton-secundario">Registrarse</NuxtLink>
        </div>
        <p v-if="mensaje" class="login-mensaje" role="status">{{ mensaje }}</p>
      </form>

      <div class="continuar"><span>continuar con</span></div>
      <div class="redes-login" aria-label="Acceso con redes sociales">
        <button type="button" class="social-google" aria-label="Continuar con Google">G</button>
        <button type="button" class="social-facebook" aria-label="Continuar con Facebook">f</button>
        <button type="button" class="social-github" aria-label="Continuar con GitHub"><i class="mdi mdi-github" aria-hidden="true"></i></button>
      </div>
    </section>

  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Space+Grotesk:wght@500;600;700&display=swap');
@import url('@mdi/font/css/materialdesignicons.css');

.login-page { --registro-rosa: #e56ddd; --registro-rosa-claro: #f3c7ef; --registro-lila: #dfc5f5; --registro-tinta: #28212d; min-height: 100vh; display: flex; align-items: center; justify-content: flex-end; position: relative; color: var(--registro-tinta); background: linear-gradient(125deg, #f8d7f0 0%, #e8d6f7 48%, #fff4f8 100%); font-family: 'DM Sans', sans-serif; }
.boton-inicio { position: absolute; z-index: 2; top: 28px; left: 34px; padding: 10px 15px; border: 1px solid rgba(255,255,255,.8); border-radius: 20px; color: white; background: #a33ca6; font-family: 'DM Sans', sans-serif; font-size: 12px; font-weight: 700; transition: background .2s, transform .2s; }
.boton-inicio:hover { background: #8e328f; transform: translateY(-2px); }
.login-copy { position: absolute; z-index: 1; top: 24%; left: 5.5%; color: var(--registro-tinta); font-family: 'DM Sans', sans-serif; }
.login-copy > span { display: block; margin-bottom: 18px; font-size: 17px; }
.login-copy h2, .login-copy p { margin: 0; font-weight: 700; line-height: .98; }
.login-copy h2 { font-size: clamp(2.7rem, 4.8vw, 4.4rem); letter-spacing: -.06em; }
.login-copy p { font-size: clamp(2.5rem, 4.4vw, 4rem); font-weight: 400; letter-spacing: -.06em; }
.login-panel { position: absolute; left: 75%; display: grid; grid-template-rows: auto 1fr auto auto; width: min(46vw, 560px); min-height: 82vh; padding: 70px 62px; border: 1px solid rgba(255,255,255,.8); border-radius: 34px; background: rgba(255,255,255,.8); box-shadow: 0 24px 70px rgba(106,61,110,.13); backdrop-filter: blur(12px); transform: translateX(-50%); }
.login-cabecera { display: flex; justify-content: center; width: 100%; }
.login-cabecera h1 { margin: 0 0 24px; color: #b941b9; text-align: center; font-size: clamp(2rem, 3.5vw, 3.5rem); letter-spacing: -.04em; line-height: 1; }
.login-formulario { display: grid; align-self: center; gap: 13px; width: 100%; margin: 0; }
.login-formulario label { margin-left: 16px; color: #46384a; font-size: 13px; font-weight: 700; }
.login-formulario input { width: 100%; height: 54px; margin-bottom: 7px; padding: 0 21px; border: 2px solid #e6d9e8; border-radius: 27px; outline: none; color: var(--registro-tinta); background: white; box-shadow: 0 6px 16px rgba(106,61,110,.1); font: inherit; font-size: 14px; transition: border-color .2s, box-shadow .2s, transform .2s; }
.login-formulario input::placeholder { color: #a89caa; }
.login-formulario input:focus { border-color: #cb65d0; box-shadow: 0 0 0 4px rgba(203,101,208,.15), 0 7px 16px rgba(106,61,110,.12); transform: translateY(-1px); }
.acciones-login { display: grid; grid-template-columns: 1fr 1fr; gap: 18px; margin-top: 20px; }
.login-formulario button, .login-formulario .boton-secundario, .redes-login button { border: 0; border-radius: 20px; color: var(--registro-tinta); background: white; cursor: pointer; font: inherit; font-weight: 700; }
.login-formulario button { width: 100%; height: 44px; padding: 0 12px; font-size: 13px; }
.login-formulario .boton-secundario { display: grid; place-items: center; height: 44px; padding: 0 12px; font-size: 13px; text-decoration: none; }
.login-formulario button { color: white; background: #bd4dc1; }
.login-formulario button:hover:not(:disabled), .redes-login button:hover { color: white; background: #a83caf; }
.login-formulario button:disabled { cursor: wait; opacity: .7; }
.login-formulario .boton-secundario { border: 1px solid #dfc5f5; color: #8e328f; background: white; }
.login-formulario .boton-secundario:hover { color: white; background: #a83caf; }
.login-mensaje { margin: 10px 0 0; color: #8e328f; text-align: center; font-size: 11px; }
.continuar { display: flex; align-items: center; gap: 7px; margin: 32px 0 22px; color: #685b6b; font-size: 11px; font-weight: 700; white-space: nowrap; }
.continuar::before, .continuar::after { content: ''; height: 1px; flex: 1; border-top: 1px dashed #bdaabe; }
.redes-login { display: flex; justify-content: space-between; padding: 0 14px; }
.redes-login button { width: 41px; height: 31px; font-family: Arial, sans-serif; font-size: 20px; }
.social-google { color: #4285f4 !important; }
.social-facebook { color: #1877f2 !important; }
.social-github { color: #181717 !important; }
@media (max-width: 800px) { .login-page { display: block; min-height: 100vh; padding: 32vh 28px 40px 50%; background-size: 100% 100%; }.boton-inicio { top: 20px; left: 20px; }.login-copy { top: 18%; left: 6%; }.login-copy h2 { font-size: 2.1rem; }.login-copy p { font-size: 1.9rem; }.login-panel { position: relative; left: auto; display: grid; width: min(100%, 350px); min-height: 78vh; margin: 0 auto; padding: 50px 30px; transform: none; }.login-cabecera h1 { margin-bottom: 20px; } }
</style>
