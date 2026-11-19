```vue
<!-- filepath: c:\Users\Aprendiz.DESKTOP-VLGOKLF.001\Desktop\NUXT\proyectalgorit\app\pages\index.vue -->
<script setup>
import imagen from '~/assets/logo1.png'
</script>

<template>
  <main class="inicio">
    <div class="decoracion decoracion-1"></div>
    <div class="decoracion decoracion-2"></div>

    <section class="contenido">
      <div class="logo-contenedor">
        <img :src="imagen" alt="Logo de DAN'STAMP" class="imagen" />
      </div>

      <p class="etiqueta">CREATIVIDAD · CALIDAD · ESTILO</p>

      <h1>
        Bienvenidos a
        <span> DAN'STAMP </span>
      </h1>

      <p class="descripcion">
        Dale vida a tus ideas con diseños únicos y personalizados.
      </p>

      <NuxtLink to="/principal" class="boton">
        Descubre más
        <span>→</span>
      </NuxtLink>
    </section>
  </main>
</template>

<style scoped>
.inicio {
  position: relative;
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  padding: 2rem;
  color: #fff;
  background:
    radial-gradient(circle at top left, #55308f, transparent 35%),
    linear-gradient(135deg, #120d24, #281344 55%, #0d0918);
}

.contenido {
  position: relative;
  z-index: 2;
  max-width: 700px;
  text-align: center;
  animation: aparecer 1s ease-out;
}

.logo-contenedor {
  width: 190px;
  height: 190px;
  display: grid;
  place-items: center;
  margin: 0 auto 2rem;
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  box-shadow: 0 0 45px rgba(219, 120, 255, 0.45);
  animation: flotar 4s ease-in-out infinite;
  backdrop-filter: blur(10px);
}

.imagen {
  width: 78%;
  height: 78%;
  object-fit: contain;
  border-radius: 50%;
}

.etiqueta {
  margin-bottom: 1rem;
  color: #e8b9ff;
  font-size: 0.75rem;
  font-weight: bold;
  letter-spacing: 0.3rem;
}

h1 {
  margin: 0;
  font-size: clamp(2.5rem, 7vw, 5rem);
  line-height: 1.05;
  font-weight: 800;
}

h1 span {
  display: block;
  color: #e99aff;
  text-shadow: 0 0 25px rgba(233, 154, 255, 0.5);
}

.descripcion {
  max-width: 520px;
  margin: 1.5rem auto 2rem;
  color: #d8cee2;
  font-size: 1.1rem;
}

.boton {
  padding: 0.9rem 1.7rem;
  border: 0;
  border-radius: 999px;
  color: #0a0a0a;
  background: #e26efc;
  font-size: 1rem;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 8px 25px rgba(199, 12, 241, 0.3);
  transition: 0.3s ease;
}

.boton span {
  margin-left: 0.5rem;
  font-size: 1.3rem;
}

.boton:hover {
  transform: translateY(-4px) scale(1.04);
  background: #fff;
  box-shadow: 0 12px 30px rgba(241, 180, 255, 0.5);
}

.decoracion {
  position: absolute;
  border-radius: 50%;
  filter: blur(3px);
  opacity: 0.35;
  animation: pulso 5s ease-in-out infinite;
}

.decoracion-1 {
  top: 10%;
  left: 8%;
  width: 120px;
  height: 120px;
  background: #ff70d4;
}

.decoracion-2 {
  right: 8%;
  bottom: 12%;
  width: 180px;
  height: 180px;
  background: #714bff;
  animation-delay: 1.5s;
}

@keyframes aparecer {
  from {
    opacity: 0;
    transform: translateY(30px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes flotar {
  0%, 100% {
    transform: translateY(0) rotate(0deg);
  }

  50% {
    transform: translateY(-12px) rotate(2deg);
  }
}

@keyframes pulso {
  0%, 100% {
    transform: scale(1);
  }

  50% {
    transform: scale(1.2);
  }
}
</style>
```