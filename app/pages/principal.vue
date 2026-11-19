<script setup>
import imagenSuperior from '~/assets/paginaprincipal/1.png'
import imagenProducto1 from '~/assets/paginaprincipal/2.png'
import imagenProducto2 from '~/assets/paginaprincipal/3.png'
import imagenProducto3 from '~/assets/paginaprincipal/4.png'
import imagenProducto4 from '~/assets/paginaprincipal/5.png'
import cafe from '~/assets/paginaprincipal/cafenube.png'
import logo from '~/assets/logo1.png'

const productos = [
  { nombre: 'Camisetas', imagen: imagenProducto1, tono: 'rosa' },
  { nombre: 'Sudaderas', imagen: imagenProducto2, tono: 'azul' },
  { nombre: 'Tote bags', imagen: imagenProducto3, tono: 'verde' },
  { nombre: 'Tazas', imagen: imagenProducto4, tono: 'amarillo' },
]

const pasos = [
  { numero: '01', titulo: 'Cuéntanos tu idea', texto: 'Escríbenos qué quieres crear y para quién.' },
  { numero: '02', titulo: 'Damos forma al diseño', texto: 'Preparamos una propuesta que encaje contigo.' },
  { numero: '03', titulo: 'Lo hacemos realidad', texto: 'Recibes una pieza cuidada y lista para disfrutar.' },
]

const preguntas = [
  '¿Puedo pedir una sola unidad?',
  '¿Qué técnicas de impresión utilizáis?',
  '¿Cuánto tarda mi pedido?',
  '¿Puedo llevar mi propio diseño?',
]
</script>

<template>
  <main class="pagina-principal">
    <header class="barra-navegacion">
      <NuxtLink to="/" class="marca" aria-label="Volver al inicio">
        <img :src="logo" alt="DAN'STAMP" class="logo-imagen" />
        <span>DAN'STAMP</span>
      </NuxtLink>
      <nav class="navegacion" aria-label="Navegación principal">
        <a href="#productos">Productos</a>
        <a href="#proceso">Cómo funciona</a>
        <a href="#preguntas">Preguntas</a>
      </nav>
      <NuxtLink class="boton boton-oscuro" to="/login">Iniciar sesión <span>↗</span></NuxtLink>
    </header>

    <section class="hero seccion-ancha">
      <div class="hero-copia">
        <p class="ceja">PERSONALIZACIÓN CON CARÁCTER</p>
        <h2>Atrevete a<br /><em>brillar</em> con lo<br />que hablas por ti.</h2>
        <p class="hero-descripcion">Creamos prendas y objetos que convierten tus ideas en algo que puedes llevar, regalar y recordar.</p>
        <a class="boton boton-oscuro" href="#productos">Ver lo que hacemos <span>↗</span></a>
      </div>
      <div class="hero-imagen">
        <img :src="imagenSuperior" alt="Prenda personalizada de DAN'STAMP" />
        <span class="nota-foto">Hecho para ser tú <strong>✦</strong></span>
      </div>
    </section>

    <section id="productos" class="productos seccion-contenido">
      <div class="cabecera-seccion">
        <div>
          <p class="ceja">01 / PERSONALIZA TU PRODUCTO</p>
          <h2>Ideas que se<br /><em>llevan puestas.</em></h2>
        </div>
        <p class="texto-seccion">Desde una pieza para ti hasta un pedido para todo tu equipo. Tu imaginación marca el límite.</p>
      </div>
      <div class="rejilla-productos">
        <article v-for="producto in productos" :key="producto.nombre" class="producto" :class="`producto-${producto.tono}`">
          <div class="producto-imagen"><img :src="producto.imagen" :alt="producto.nombre" /></div>
          <div class="producto-info"><h3>{{ producto.nombre }}</h3><span class="flecha">↗</span></div>
        </article>
      </div>
    </section>

    <section id="proceso" class="proceso seccion-ancha">
      <div class="cabecera-seccion cabecera-proceso">
        <div><p class="ceja">02 / MENOS DUDA, MÁS ESTILO</p><h2>Así de<br /><em>fácil.</em></h2></div>
        <p class="texto-seccion">Tu tiempo importa. Hemos hecho el proceso claro para que solo tengas que preocuparte de tener una buena idea.</p>
      </div>
      <div class="rejilla-pasos">
        <article v-for="paso in pasos" :key="paso.numero" class="paso">
          <span class="paso-numero">{{ paso.numero }}</span>
          <h3>{{ paso.titulo }}</h3>
          <p>{{ paso.texto }}</p>
        </article>
      </div>
    </section>

    <section class="trabajos seccion-ancha">
      <div class="trabajos-copia"><p class="ceja">03 / TRABAJOS QUE HABLAN</p><h2>Hecho con<br /><em>intención.</em></h2><p>Cada encargo es una oportunidad para hacer algo diferente. Mira algunas de nuestras piezas favoritas.</p><a href="#contacto" class="enlace">Quiero crear algo <span>↗</span></a></div>
      <div class="trabajo-imagen"><img :src="cafe" alt="Pack de café personalizado" /><span>PACK · CAFÉ NUBE</span></div>
    </section>

    <section class="razones seccion-contenido">
      <p class="ceja">04 / BUENAS IDEAS. MEJOR IMPRESAS.</p>
      <div class="rejilla-razones"><article><strong>01</strong><h3>Diseño con intención</h3><p>No hacemos copias. Hacemos piezas que tienen algo que decir.</p></article><article><strong>02</strong><h3>Cuidamos el detalle</h3><p>Materiales, color y acabado importan tanto como la idea.</p></article><article><strong>03</strong><h3>Trato cercano</h3><p>Siempre sabrás en qué punto está tu pedido.</p></article></div>
    </section>

    <section class="confianza seccion-contenido"><div><p class="ceja">05 / GENTE FELIZ, PRENDAS FELICES</p><h2>Lo que dicen<br /><em>de nosotros.</em></h2></div><div class="testimonio"><p>“DAN'STAMP entendió exactamente lo que tenía en la cabeza y lo convirtió en mi camiseta favorita.”</p><strong>— Marta G. · Diseñadora</strong></div><div class="testimonio testimonio-claro"><p>“El resultado fue precioso y el proceso muy fácil. Repetiremos seguro.”</p><strong>— Equipo Nube</strong></div></section>

    <section id="preguntas" class="preguntas seccion-contenido"><div><p class="ceja">06 / PREGUNTAS FRECUENTES</p><h2>¿Tienes dudas?<br /><em>Hablemos.</em></h2></div><div class="lista-preguntas"><details v-for="pregunta in preguntas" :key="pregunta"><summary>{{ pregunta }} <span>+</span></summary><p>Escríbenos y te contamos todos los detalles de tu proyecto sin compromiso.</p></details></div></section>

    <section id="contacto" class="llamada seccion-ancha"><div><p class="ceja">UNA BUENA IDEA MERECE SALIR</p><h2>¿Lo tienes en mente?<br /><em>Ponlo en tela.</em></h2></div><img :src="imagenProducto3" alt="Tote bag personalizada" /><a class="boton boton-rosa" href="mailto:hola@danstamp.com">Cuéntanos tu idea <span>↗</span></a></section>

    <footer class="pie"><NuxtLink to="/" class="marca"><img :src="logo" alt="DAN'STAMP" class="logo-imagen" /><span>DAN'STAMP</span></NuxtLink><p>Diseños que hablan por ti.</p><span>© 2026 DAN'STAMP</span></footer>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Space+Grotesk:wght@500;600;700&display=swap');

:root { --negro: #191718; --rosa: #ff4da3; --rosa-claro: #f3c7ef; --lila: #d9b2f5; --azul: #5274ed; --crema: #f4f0e8; --verde: #c9dfd4; }
* { box-sizing: border-box; }
html { scroll-behavior: smooth; }
body { margin: 0; background: var(--crema); color: var(--negro); font-family: 'DM Sans', sans-serif; }
a { color: inherit; text-decoration: none; }
.pagina-principal { overflow: hidden; }
.seccion-ancha { padding-left: max(5vw, 28px); padding-right: max(5vw, 28px); }
.seccion-contenido { max-width: 1240px; margin: auto; padding-left: 5vw; padding-right: 5vw; }
.barra-navegacion { height: 84px; display: flex; align-items: center; justify-content: space-between; padding: 0 5vw; color: white; background: var(--negro); }
.marca { display: inline-flex; align-items: center; gap: 9px; font-family: 'Space Grotesk', sans-serif; font-weight: 700; letter-spacing: -.05em; }
.logo-imagen { display: block; width: 68px; height: 68px; object-fit: contain; background: transparent; }
.marca-sello { display: grid; place-items: center; width: 28px; height: 28px; border-radius: 50%; color: white; background: var(--negro); font-size: 16px; }
.navegacion { display: flex; gap: 32px; font-size: 13px; font-weight: 700; }
.navegacion a:hover, .enlace:hover { color: var(--rosa); }
.boton { display: inline-flex; align-items: center; gap: 18px; border-radius: 999px; padding: 14px 20px; font-size: 12px; font-weight: 700; transition: transform .25s, background .25s; }
.boton:hover { transform: translateY(-3px); }
.boton-oscuro { color: var(--negro); background: white; }
.boton-oscuro:hover { background: var(--rosa); }
.hero { display: grid; grid-template-columns: .96fr 1.04fr; gap: 28px; min-height: 620px; padding-top: 34px; padding-bottom: 44px; background: var(--rosa-claro); }
.hero-copia { display: flex; flex-direction: column; align-items: flex-start; justify-content: center; max-width: 550px; padding-left: 6vw; }
.ceja { margin: 0 0 22px; color: #6c6269; font-size: 10px; font-weight: 700; letter-spacing: .16em; }
h1, h2 { margin: 0; font-family: 'Space Grotesk', sans-serif; font-weight: 700; letter-spacing: -.07em; line-height: .92; }
h1 { font-size: clamp(3.2rem, 6.5vw, 6.7rem); }
h2 { font-size: clamp(2.8rem, 5vw, 5.1rem); }
em { color: var(--rosa); font-style: normal; }
.hero-descripcion { max-width: 390px; margin: 28px 0; color: #51474d; line-height: 1.5; }
.hero-imagen { position: relative; min-height: 510px; overflow: hidden; border-radius: 4px; background: #edc1c4; }
.hero-imagen img { width: 100%; height: 100%; object-fit: cover; object-position: center; }
.nota-foto { position: absolute; right: 22px; bottom: 22px; padding: 11px 15px; border-radius: 4px; background: white; font-size: 11px; transform: rotate(-3deg); }
.nota-foto strong { margin-left: 15px; color: var(--rosa); }
.productos { padding-top: 110px; padding-bottom: 120px; }
.cabecera-seccion { display: flex; align-items: end; justify-content: space-between; gap: 30px; margin-bottom: 42px; }
.texto-seccion { max-width: 300px; margin: 0 0 5px; color: #6c6269; font-size: 13px; line-height: 1.6; }
.rejilla-productos { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
.producto { overflow: hidden; border-radius: 4px; }
.producto-imagen { aspect-ratio: .78; overflow: hidden; }
.producto-imagen img { width: 100%; height: 100%; object-fit: cover; transition: transform .45s; }
.producto:hover img { transform: scale(1.05); }
.producto-info { position: relative; min-height: 104px; padding: 17px; }
.producto-info span:first-child { font-size: 11px; }
.producto-info h3 { margin: 12px 0 0; font-family: 'Space Grotesk'; font-size: 20px; letter-spacing: -.05em; }
.producto-info .flecha { position: absolute; right: 17px; bottom: 17px; font-size: 20px; }
.producto-rosa .producto-info { background: #ff70b4; }.producto-azul .producto-info { background: #7c92ec; }.producto-verde .producto-info { background: #bdd8c4; }.producto-amarillo .producto-info { background: #f4d676; }
.proceso { padding-top: 90px; padding-bottom: 100px; background: var(--lila); }
.cabecera-proceso { max-width: 1240px; margin-left: auto; margin-right: auto; }
.rejilla-pasos { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; max-width: 1240px; margin: auto; }
.paso { min-height: 190px; padding: 22px; background: rgba(255,255,255,.56); }.paso-numero { font-family: 'Space Grotesk'; font-size: 28px; }.paso h3 { margin: 45px 0 8px; font-family: 'Space Grotesk'; font-size: 20px; }.paso p { margin: 0; color: #6c6269; font-size: 12px; line-height: 1.45; }
.trabajos { display: grid; grid-template-columns: .85fr 1.15fr; align-items: center; gap: 8vw; padding-top: 120px; padding-bottom: 120px; color: white; background: var(--negro); }.trabajos-copia { max-width: 380px; padding-left: 4vw; }.trabajos .ceja, .trabajos-copia > p { color: #b7adb3; }.trabajos-copia > p { line-height: 1.6; }.trabajo-imagen { position: relative; max-width: 560px; }.trabajo-imagen img { display: block; width: 100%; aspect-ratio: .93; object-fit: cover; border-radius: 4px; }.trabajo-imagen span { position: absolute; left: 14px; bottom: 14px; padding: 10px 13px; border-radius: 30px; color: var(--negro); background: white; font-size: 11px; font-weight: 700; }.enlace { display: inline-block; margin-top: 24px; border-bottom: 1px solid var(--rosa); padding-bottom: 5px; color: white; font-size: 12px; font-weight: 700; }.enlace span { margin-left: 22px; color: var(--rosa); font-size: 16px; }
.razones { padding-top: 100px; padding-bottom: 110px; }.rejilla-razones { display: grid; grid-template-columns: repeat(3, 1fr); gap: 14px; }.rejilla-razones article { padding: 24px; min-height: 200px; border-radius: 4px; }.rejilla-razones article:nth-child(1) { background: #f5b7de; }.rejilla-razones article:nth-child(2) { color: white; background: var(--azul); }.rejilla-razones article:nth-child(3) { background: #bd70e7; }.rejilla-razones strong { font-family: 'Space Grotesk'; }.rejilla-razones h3 { margin-top: 60px; font-family: 'Space Grotesk'; font-size: 18px; }.rejilla-razones p { max-width: 210px; font-size: 12px; line-height: 1.45; }
.confianza { display: grid; grid-template-columns: 1.1fr 1fr 1fr; gap: 14px; align-items: end; padding-top: 90px; padding-bottom: 100px; background: var(--verde); max-width: none; }.confianza > div:first-child { max-width: 340px; margin-left: max(0px, calc((100vw - 1140px) / 2)); }.testimonio { padding: 25px; background: white; }.testimonio-claro { background: #e8f8f2; }.testimonio p { min-height: 85px; margin: 0 0 30px; font-family: 'Space Grotesk'; font-size: 18px; line-height: 1.15; }.testimonio strong { font-size: 11px; }
.preguntas { display: grid; grid-template-columns: .8fr 1.2fr; gap: 12vw; padding-top: 120px; padding-bottom: 120px; }.lista-preguntas details { border-top: 1px solid #c7bcc3; }.lista-preguntas details:last-child { border-bottom: 1px solid #c7bcc3; }.lista-preguntas summary { display: flex; justify-content: space-between; padding: 20px 0; cursor: pointer; font-size: 14px; font-weight: 700; list-style: none; }.lista-preguntas summary::-webkit-details-marker { display: none; }.lista-preguntas summary span { color: var(--azul); font-size: 20px; }.lista-preguntas p { margin: 0 0 20px; color: #6c6269; font-size: 12px; }
.llamada { position: relative; display: flex; align-items: center; justify-content: space-between; min-height: 370px; overflow: hidden; background: #a9a4a0; }.llamada h2 { position: relative; z-index: 1; }.llamada img { position: absolute; right: 22%; bottom: -100px; width: 220px; height: 340px; object-fit: cover; transform: rotate(9deg); }.boton-rosa { position: relative; z-index: 1; background: var(--rosa); }.boton-rosa:hover { background: white; }.pie { display: flex; justify-content: space-between; align-items: center; padding: 25px 5vw; color: white; background: var(--negro); font-size: 11px; }.pie p { color: #b7adb3; }
@media (max-width: 800px) { .barra-navegacion { height: 70px; }.navegacion { display: none; }.barra-navegacion .boton { padding: 10px 14px; }.hero { display: flex; flex-direction: column; padding-top: 25px; }.hero-copia { padding: 35px 10px 20px; }.hero-imagen { min-height: 380px; }.productos, .razones, .preguntas { padding-top: 75px; padding-bottom: 75px; }.cabecera-seccion { display: block; }.texto-seccion { margin-top: 24px; }.rejilla-productos { grid-template-columns: repeat(2, 1fr); }.producto-info { min-height: 95px; }.producto-info h3 { font-size: 17px; }.proceso { padding-top: 70px; padding-bottom: 70px; }.rejilla-pasos, .rejilla-razones { grid-template-columns: 1fr; }.paso { min-height: 150px; }.paso h3 { margin-top: 28px; }.trabajos { grid-template-columns: 1fr; padding-top: 75px; padding-bottom: 75px; }.trabajos-copia { padding-left: 0; }.trabajo-imagen { margin-top: 25px; }.confianza { grid-template-columns: 1fr; padding-top: 70px; padding-bottom: 70px; }.confianza > div:first-child { margin-left: 0; }.preguntas { grid-template-columns: 1fr; gap: 45px; }.llamada { display: block; min-height: 500px; padding-top: 70px; padding-bottom: 50px; }.llamada img { right: 16%; bottom: -70px; width: 180px; height: 280px; }.llamada .boton { position: absolute; left: 28px; bottom: 35px; }.pie { align-items: flex-start; flex-direction: column; gap: 18px; } }
</style>
