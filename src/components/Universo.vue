<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import musicaUrl from '../assets/music.mp4'

const canvasEl = ref(null)
const audioEl = ref(null)
let cleanup = () => {}

// Reproduce la música con volumen (llamado al primer gesto del usuario)
function activarMusica() {
  const audio = audioEl.value
  if (!audio) return
  audio.muted = false
  audio.volume = 0.55
  audio.play().catch(() => {})
}

// ---------- Dedicatorias bonitas (rotan en pantalla) ----------
const DEDICATORIAS = [
  'Eres la flor más bonita de mi jardín. 💛',
  'Gracias por pintar mis días de amarillo. 🌻',
  'En el cielo hay estrellas; en mi vida, estás tú. ✨',
  'La amistad es la flor que nunca se marchita. 🌸',
  'Un ramo de flores y mil buenos deseos para ti. 🤍',
  'Contigo, hasta el infinito se queda corto. 💫',
  'Que nunca te falten amor, flores ni razones para reír. 🌷',
  'Este universo entero conspira para regalarte sonrisas. 🌼',
]
const idxDedicatoria = ref(0)
const dedicatoria = ref(DEDICATORIAS[0])
let intervaloDedicatoria = 0

// ---------- Textura de flor (dibujada en canvas 2D) ----------
function crearTexturaFlor(colorPetalo, colorCentro) {
  const size = 256
  const c = document.createElement('canvas')
  c.width = c.height = size
  const ctx = c.getContext('2d')
  const cx = size / 2
  const cy = size / 2
  const petalos = 6
  const largoPetalo = size * 0.36
  const anchoPetalo = size * 0.16

  ctx.clearRect(0, 0, size, size)

  for (let i = 0; i < petalos; i++) {
    const ang = (i / petalos) * Math.PI * 2
    ctx.save()
    ctx.translate(cx, cy)
    ctx.rotate(ang)

    const grad = ctx.createLinearGradient(0, 0, 0, -largoPetalo)
    grad.addColorStop(0, colorCentro)
    grad.addColorStop(0.35, colorPetalo)
    grad.addColorStop(1, colorPetalo)

    ctx.beginPath()
    ctx.ellipse(0, -largoPetalo * 0.55, anchoPetalo, largoPetalo * 0.55, 0, 0, Math.PI * 2)
    ctx.fillStyle = grad
    ctx.shadowColor = colorPetalo
    ctx.shadowBlur = 14
    ctx.fill()

    ctx.shadowBlur = 0
    ctx.beginPath()
    ctx.ellipse(0, -largoPetalo * 0.55, anchoPetalo * 0.35, largoPetalo * 0.45, 0, 0, Math.PI * 2)
    ctx.fillStyle = 'rgba(255,255,255,0.35)'
    ctx.fill()

    ctx.restore()
  }

  const gc = ctx.createRadialGradient(cx, cy, 2, cx, cy, size * 0.15)
  gc.addColorStop(0, '#fff3b0')
  gc.addColorStop(0.5, colorCentro)
  gc.addColorStop(1, colorPetalo)
  ctx.beginPath()
  ctx.arc(cx, cy, size * 0.15, 0, Math.PI * 2)
  ctx.fillStyle = gc
  ctx.fill()

  const tex = new THREE.CanvasTexture(c)
  tex.colorSpace = THREE.SRGBColorSpace
  tex.needsUpdate = true
  return tex
}

// ---------- Textura circular para estrellas ----------
function crearTexturaPunto() {
  const size = 64
  const c = document.createElement('canvas')
  c.width = c.height = size
  const ctx = c.getContext('2d')
  const g = ctx.createRadialGradient(size / 2, size / 2, 0, size / 2, size / 2, size / 2)
  g.addColorStop(0, 'rgba(255,255,255,1)')
  g.addColorStop(0.35, 'rgba(255,255,255,0.85)')
  g.addColorStop(1, 'rgba(255,255,255,0)')
  ctx.fillStyle = g
  ctx.beginPath()
  ctx.arc(size / 2, size / 2, size / 2, 0, Math.PI * 2)
  ctx.fill()
  const tex = new THREE.CanvasTexture(c)
  tex.colorSpace = THREE.SRGBColorSpace
  tex.needsUpdate = true
  return tex
}

onMounted(() => {
  // ---------- Música de fondo (autoplay + bucle) ----------
  const audio = audioEl.value
  let quitarDesbloqueo = () => {}
  if (audio) {
    // Arranca en silencio (el autoplay silenciado SÍ lo permiten todos
    // los navegadores); al primer gesto del usuario le damos volumen.
    audio.muted = true
    audio.play().catch(() => {})

    const desbloquear = () => {
      activarMusica()
      quitarDesbloqueo()
    }
    window.addEventListener('pointerdown', desbloquear)
    window.addEventListener('keydown', desbloquear)
    window.addEventListener('touchstart', desbloquear)
    quitarDesbloqueo = () => {
      window.removeEventListener('pointerdown', desbloquear)
      window.removeEventListener('keydown', desbloquear)
      window.removeEventListener('touchstart', desbloquear)
    }
  }

  const canvas = canvasEl.value
  const scene = new THREE.Scene()
  scene.fog = new THREE.FogExp2(0x05010f, 0.014)

  const camera = new THREE.PerspectiveCamera(
    60, window.innerWidth / window.innerHeight, 0.1, 1000
  )

  // Distancia de cámara adaptable: en pantallas angostas (móvil vertical)
  // se aleja para que el ramo completo siempre quepa en cuadro.
  let radioCamara = 26
  function calcularRadioCamara() {
    const aspecto = window.innerWidth / window.innerHeight
    // a menor aspecto (más vertical), más lejos la cámara
    radioCamara = aspecto < 1 ? 26 / Math.max(aspecto, 0.42) : 26
  }
  calcularRadioCamara()
  camera.position.set(0, 0, radioCamara)

  const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(window.innerWidth, window.innerHeight)

  // ---------- Estrellas ----------
  const texPunto = crearTexturaPunto()
  function crearEstrellas(cantidad, radio, tam, color) {
    const geo = new THREE.BufferGeometry()
    const pos = new Float32Array(cantidad * 3)
    for (let i = 0; i < cantidad; i++) {
      const r = radio * Math.cbrt(Math.random())
      const theta = Math.random() * Math.PI * 2
      const phi = Math.acos(2 * Math.random() - 1)
      pos[i * 3] = r * Math.sin(phi) * Math.cos(theta)
      pos[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta)
      pos[i * 3 + 2] = r * Math.cos(phi)
    }
    geo.setAttribute('position', new THREE.BufferAttribute(pos, 3))
    const mat = new THREE.PointsMaterial({
      size: tam, color, map: texPunto, alphaMap: texPunto,
      transparent: true, opacity: 0.95, sizeAttenuation: true,
      depthWrite: false, blending: THREE.AdditiveBlending,
    })
    return new THREE.Points(geo, mat)
  }

  const estrellas = crearEstrellas(2600, 130, 0.9, 0xffffff)
  const estrellasDoradas = crearEstrellas(900, 110, 1.4, 0xffd85a)
  scene.add(estrellas, estrellasDoradas)

  // =====================================================
  //                   RAMO DE FLORES
  // =====================================================
  const ramo = new THREE.Group()
  ramo.position.y = -1
  scene.add(ramo)

  const baseRamo = new THREE.Vector3(0, -9, 0) // donde se atan los tallos
  const centroDomo = new THREE.Vector3(0, 3, 0)
  const radioDomo = 6

  // --- Papel de envoltura (cono kraft, abierto) ---
  const geoPapel = new THREE.ConeGeometry(5, 10, 24, 1, true)
  const matPapel = new THREE.MeshStandardMaterial({
    color: 0xe9d8b0, roughness: 0.9, metalness: 0.0,
    side: THREE.DoubleSide, transparent: true, opacity: 0.96,
  })
  const papel = new THREE.Mesh(geoPapel, matPapel)
  papel.rotation.x = Math.PI          // boca ancha hacia arriba
  papel.position.y = -4               // apex ~ -9, boca ~ +1
  ramo.add(papel)

  // capa interior del papel (tono más claro)
  const papelInt = new THREE.Mesh(
    new THREE.ConeGeometry(4.4, 9, 24, 1, true),
    new THREE.MeshStandardMaterial({
      color: 0xfff4dc, roughness: 1, side: THREE.BackSide,
      transparent: true, opacity: 0.9,
    })
  )
  papelInt.rotation.x = Math.PI
  papelInt.position.y = -4
  ramo.add(papelInt)

  // --- Moño / listón dorado en el cuello ---
  const matListon = new THREE.MeshStandardMaterial({
    color: 0xffcf3d, roughness: 0.4, metalness: 0.3, emissive: 0x5a4300,
  })
  const liston = new THREE.Mesh(new THREE.TorusGeometry(2.05, 0.32, 12, 32), matListon)
  liston.position.y = -6
  liston.rotation.x = Math.PI / 2
  ramo.add(liston)
  // lazo (dos esferas achatadas)
  for (const s of [-1, 1]) {
    const lazo = new THREE.Mesh(new THREE.SphereGeometry(0.7, 12, 12), matListon)
    lazo.position.set(s * 0.9, -5.6, 0)
    lazo.scale.set(1, 0.6, 0.5)
    ramo.add(lazo)
  }

  // --- Tallos verdes (cilindros del cuello a cada flor) ---
  const matTallo = new THREE.MeshStandardMaterial({ color: 0x3a8f3a, roughness: 0.7 })
  function crearTallo(punta) {
    const dir = new THREE.Vector3().subVectors(punta, baseRamo)
    const len = dir.length()
    const geo = new THREE.CylinderGeometry(0.05, 0.09, len, 6)
    const mesh = new THREE.Mesh(geo, matTallo)
    mesh.position.copy(baseRamo).addScaledVector(dir, 0.5)
    mesh.quaternion.setFromUnitVectors(
      new THREE.Vector3(0, 1, 0), dir.clone().normalize()
    )
    return mesh
  }

  // --- Cabezas de flores (sprites) sobre un domo ---
  const texFlorAmarilla = crearTexturaFlor('#ffdf4d', '#a4670a')
  const texFlorBlanca = crearTexturaFlor('#ffffff', '#ffcf3d')

  const flores = []
  const NUM_FLORES = 34
  const goldenAngle = Math.PI * (3 - Math.sqrt(5))
  for (let i = 0; i < NUM_FLORES; i++) {
    // punto en el casquete superior de una esfera (distribución fibonacci)
    const y = 1 - (i / (NUM_FLORES - 1)) * 0.95   // 1 -> arriba, 0.05 -> lado
    const rAnillo = Math.sqrt(1 - y * y)
    const theta = i * goldenAngle
    const dir = new THREE.Vector3(
      Math.cos(theta) * rAnillo,
      y,
      Math.sin(theta) * rAnillo
    )
    const jitter = 0.85 + Math.random() * 0.3
    const punta = centroDomo.clone().addScaledVector(dir, radioDomo * jitter)

    // tallo
    ramo.add(crearTallo(punta))

    // flor
    const amarilla = Math.random() > 0.45
    const tex = amarilla ? texFlorAmarilla : texFlorBlanca
    const mat = new THREE.SpriteMaterial({
      map: tex, transparent: true, depthWrite: false, opacity: 1,
    })
    const sp = new THREE.Sprite(mat)
    sp.position.copy(punta)
    const escala = 2.6 + Math.random() * 1.6
    sp.scale.set(escala, escala, 1)
    sp.userData = {
      spin: (Math.random() - 0.5) * 0.006,
      base: punta.clone(),
      float: Math.random() * Math.PI * 2,
      floatSpeed: 0.5 + Math.random() * 0.6,
      amp: 0.15 + Math.random() * 0.2,
    }
    flores.push(sp)
    ramo.add(sp)
  }

  // brillo cálido detrás del ramo
  const glow = new THREE.Sprite(new THREE.SpriteMaterial({
    map: texPunto, color: 0xffd85a, transparent: true, opacity: 0.35,
    depthWrite: false, blending: THREE.AdditiveBlending,
  }))
  glow.scale.set(28, 28, 1)
  glow.position.set(0, 3, -4)
  ramo.add(glow)

  // ---------- Luces ----------
  scene.add(new THREE.AmbientLight(0xfff2d0, 0.9))
  const luzPrincipal = new THREE.PointLight(0xffe9a8, 1.4, 200)
  luzPrincipal.position.set(6, 10, 18)
  scene.add(luzPrincipal)
  const luzRelleno = new THREE.PointLight(0xa9c4ff, 0.5, 200)
  luzRelleno.position.set(-12, -4, 10)
  scene.add(luzRelleno)

  // ---------- Interacción con mouse ----------
  const mouse = { x: 0, y: 0 }
  const target = { x: 0, y: 0 }
  function onMouseMove(e) {
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1
    mouse.y = (e.clientY / window.innerHeight) * 2 - 1
  }
  window.addEventListener('mousemove', onMouseMove)

  // ---------- Resize ----------
  function onResize() {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    calcularRadioCamara()
    renderer.setSize(window.innerWidth, window.innerHeight)
  }
  window.addEventListener('resize', onResize)

  // ---------- Dedicatorias rotando ----------
  intervaloDedicatoria = window.setInterval(() => {
    idxDedicatoria.value = (idxDedicatoria.value + 1) % DEDICATORIAS.length
    dedicatoria.value = DEDICATORIAS[idxDedicatoria.value]
  }, 5000)

  // ---------- Loop ----------
  const t0 = performance.now()
  let raf = 0
  function animate() {
    raf = requestAnimationFrame(animate)
    const t = (performance.now() - t0) / 1000

    target.x += (mouse.x - target.x) * 0.05
    target.y += (mouse.y - target.y) * 0.05

    // cámara: casi frontal, con vaivén suave y parallax del mouse
    const vaiven = Math.sin(t * 0.2) * 0.35 + target.x * 0.5
    camera.position.x = Math.sin(vaiven) * radioCamara
    camera.position.z = Math.cos(vaiven) * radioCamara
    camera.position.y = 1 + target.y * -4
    camera.lookAt(0, 0, 0)

    estrellas.rotation.y = t * 0.01
    estrellasDoradas.rotation.y = -t * 0.014

    // el ramo respira suavemente
    ramo.rotation.y = Math.sin(t * 0.25) * 0.12
    ramo.position.y = -1 + Math.sin(t * 0.6) * 0.15

    flores.forEach((f) => {
      f.material.rotation += f.userData.spin
      f.position.y = f.userData.base.y +
        Math.sin(t * f.userData.floatSpeed + f.userData.float) * f.userData.amp
    })

    renderer.render(scene, camera)
  }
  animate()

  // ---------- Cleanup ----------
  cleanup = () => {
    cancelAnimationFrame(raf)
    clearInterval(intervaloDedicatoria)
    quitarDesbloqueo()
    if (audio) audio.pause()
    window.removeEventListener('mousemove', onMouseMove)
    window.removeEventListener('resize', onResize)
    renderer.dispose()
    scene.traverse((obj) => {
      if (obj.geometry) obj.geometry.dispose()
      if (obj.material) {
        if (obj.material.map) obj.material.map.dispose()
        obj.material.dispose()
      }
    })
  }
})

onBeforeUnmount(() => cleanup())
</script>

<template>
  <div class="universo">
    <audio ref="audioEl" :src="musicaUrl" loop autoplay playsinline hidden preload="auto"></audio>
    <canvas ref="canvasEl" class="lienzo"></canvas>


    <div class="overlay-top">
      <h1 class="titulo">Feliz Día del Amor y la Amistad</h1>
      <p class="subtitulo">Un ramo de flores viajando por el universo, solo para ti, Niki Nicole sonsaaaaaaa 💛🤍</p>
    </div>

    <div class="dedicatoria-zona">
      <Transition name="fade" mode="out-in">
        <p class="dedicatoria" :key="dedicatoria">{{ dedicatoria }}</p>
      </Transition>
      <p class="firma">— Con cariño de Robert Gabriel Flores, para alguien especial ✨</p>
    </div>
  </div>
</template>

<style scoped>
.universo {
  position: fixed;
  inset: 0;
  overflow: hidden;
  background: radial-gradient(ellipse at center, #1a0f36 0%, #05010f 72%);
}

.lienzo {
  position: absolute;
  inset: 0;
  display: block;
}


.overlay-top {
  position: absolute;
  top: 6%;
  left: 0;
  right: 0;
  text-align: center;
  pointer-events: none;
  padding: 0 1rem;
}

.titulo {
  margin: 0;
  font-family: "Segoe UI", system-ui, sans-serif;
  font-size: clamp(1.6rem, 4.8vw, 3.3rem);
  font-weight: 800;
  letter-spacing: 0.02em;
  background: linear-gradient(90deg, #fff7d6, #ffd85a, #ffffff);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  text-shadow: 0 0 30px rgba(255, 216, 90, 0.4);
}

.subtitulo {
  margin: 0.5rem 0 0;
  color: #f4ecff;
  font-family: "Segoe UI", system-ui, sans-serif;
  font-size: clamp(0.85rem, 2.2vw, 1.1rem);
  opacity: 0.88;
}

.dedicatoria-zona {
  position: absolute;
  bottom: 7%;
  left: 0;
  right: 0;
  text-align: center;
  pointer-events: none;
  padding: 0 1.2rem;
}

.dedicatoria {
  margin: 0 auto;
  max-width: 720px;
  min-height: 2.4em;
  font-family: "Georgia", "Segoe UI", serif;
  font-style: italic;
  font-size: clamp(1.1rem, 3.2vw, 1.9rem);
  line-height: 1.35;
  color: #fff6da;
  text-shadow: 0 0 24px rgba(255, 207, 61, 0.55);
}

.firma {
  margin: 0.9rem 0 0;
  font-family: "Segoe UI", system-ui, sans-serif;
  font-size: clamp(0.75rem, 2vw, 0.95rem);
  color: #d9c8ff;
  opacity: 0.8;
}

/* transición fade de las dedicatorias */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.8s ease, transform 0.8s ease;
}
.fade-enter-from {
  opacity: 0;
  transform: translateY(12px);
}
.fade-leave-to {
  opacity: 0;
  transform: translateY(-12px);
}
</style>
