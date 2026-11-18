<script setup>
import { ref, computed, onMounted } from 'vue'
import { productosApi, ventasApi, dinero } from '../api'

const productos = ref([])
const ventas = ref([])
const error = ref('')
const aviso = ref('')

// Formulario de venta (carrito)
const editandoId = ref(null)
const productoSel = ref('')
const cantidadSel = ref(1)
const carrito = ref([]) // [{ productoId, nombre, precio, cantidad }]

const totalCarrito = computed(() =>
  carrito.value.reduce((suma, i) => suma + i.precio * i.cantidad, 0)
)

async function cargar() {
  try {
    const [p, v] = await Promise.all([productosApi.listar(), ventasApi.listar()])
    productos.value = p
    ventas.value = v
  } catch (e) {
    error.value = 'No se pudo conectar con la API: ' + e.message
  }
}

function agregar() {
  error.value = ''
  const prod = productos.value.find((p) => p.id === Number(productoSel.value))
  const cant = Number(cantidadSel.value)
  if (!prod || !(cant > 0)) {
    error.value = 'Selecciona un producto y una cantidad mayor a 0'
    return
  }
  const existente = carrito.value.find((i) => i.productoId === prod.id)
  if (existente) existente.cantidad += cant
  else carrito.value.push({ productoId: prod.id, nombre: prod.nombre, precio: Number(prod.precio), cantidad: cant })
  productoSel.value = ''
  cantidadSel.value = 1
}

function quitar(index) {
  carrito.value.splice(index, 1)
}

function limpiar() {
  editandoId.value = null
  carrito.value = []
  productoSel.value = ''
  cantidadSel.value = 1
}

function editar(v) {
  editandoId.value = v.id
  carrito.value = v.productos.map((d) => ({
    productoId: d.productoId,
    nombre: d.nombreProducto,
    precio: Number(d.precioUnitario),
    cantidad: d.cantidad,
  }))
  error.value = ''
  aviso.value = ''
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

async function registrar() {
  error.value = ''
  aviso.value = ''
  const body = {
    productos: carrito.value.map((i) => ({ productoId: i.productoId, cantidad: i.cantidad })),
  }
  try {
    if (editandoId.value) {
      await ventasApi.actualizar(editandoId.value, body)
      aviso.value = `Venta #${editandoId.value} actualizada`
    } else {
      const v = await ventasApi.crear(body)
      aviso.value = `Venta #${v.id} registrada por ${dinero(v.total)}`
    }
    limpiar()
    await cargar() // recarga ventas y stock
  } catch (e) {
    error.value = e.message
  }
}

async function eliminar(v) {
  if (!confirm(`¿Cancelar la venta #${v.id}? El stock se regresará al inventario.`)) return
  error.value = ''
  aviso.value = ''
  try {
    await ventasApi.eliminar(v.id)
    aviso.value = `Venta #${v.id} eliminada y stock devuelto`
    if (editandoId.value === v.id) limpiar()
    await cargar()
  } catch (e) {
    error.value = e.message
  }
}

const fecha = (f) => new Date(f).toLocaleString('es-MX')

onMounted(cargar)
</script>

<template>
  <section class="panel">
    <h2>{{ editandoId ? `Editar venta #${editandoId}` : 'Nueva venta' }}</h2>

    <div class="form-grid">
      <label class="ancho">
        Producto
        <select v-model="productoSel">
          <option value="" disabled>Selecciona un producto</option>
          <option v-for="p in productos" :key="p.id" :value="p.id" :disabled="p.stock === 0 && !editandoId">
            {{ p.nombre }} — {{ dinero(p.precio) }} (stock: {{ p.stock }})
          </option>
        </select>
      </label>
      <label>
        Cantidad
        <input v-model="cantidadSel" type="number" min="1" step="1" />
      </label>
      <div class="acciones">
        <button type="button" @click="agregar">Agregar</button>
      </div>
    </div>

    <table v-if="carrito.length" class="carrito">
      <thead>
        <tr><th>Producto</th><th class="num">Precio</th><th class="num">Cantidad</th><th class="num">Subtotal</th><th></th></tr>
      </thead>
      <tbody>
        <tr v-for="(i, idx) in carrito" :key="i.productoId">
          <td>{{ i.nombre }}</td>
          <td class="num">{{ dinero(i.precio) }}</td>
          <td class="num"><input v-model.number="i.cantidad" type="number" min="1" class="mini" /></td>
          <td class="num">{{ dinero(i.precio * i.cantidad) }}</td>
          <td class="acciones-fila"><button class="peligro" @click="quitar(idx)">Quitar</button></td>
        </tr>
      </tbody>
      <tfoot>
        <tr><td colspan="3" class="num"><strong>Total</strong></td><td class="num"><strong>{{ dinero(totalCarrito) }}</strong></td><td></td></tr>
      </tfoot>
    </table>
    <p v-else class="vacio">Agrega productos a la venta</p>

    <div class="acciones">
      <button class="primario" :disabled="!carrito.length" @click="registrar">
        {{ editandoId ? 'Guardar cambios' : 'Registrar venta' }}
      </button>
      <button v-if="editandoId || carrito.length" @click="limpiar">Cancelar</button>
    </div>
    <p v-if="editandoId" class="nota">Los precios se recalculan con el precio actual de cada producto al guardar.</p>

    <p v-if="error" class="error">{{ error }}</p>
    <p v-if="aviso" class="ok">{{ aviso }}</p>
  </section>

  <section class="panel">
    <h2>Ventas registradas ({{ ventas.length }})</h2>
    <p v-if="ventas.length === 0" class="vacio">No hay ventas</p>

    <article v-for="v in ventas" :key="v.id" class="venta">
      <header>
        <div>
          <strong>Venta #{{ v.id }}</strong>
          <span class="fecha">{{ fecha(v.fecha) }}</span>
        </div>
        <div class="acciones-fila">
          <span class="total">{{ dinero(v.total) }}</span>
          <button @click="editar(v)">Editar</button>
          <button class="peligro" @click="eliminar(v)">Eliminar</button>
        </div>
      </header>
      <table>
        <thead>
          <tr><th>Producto vendido</th><th class="num">Cantidad</th><th class="num">Precio unitario</th><th class="num">Subtotal</th></tr>
        </thead>
        <tbody>
          <tr v-for="d in v.productos" :key="d.productoId">
            <td>{{ d.nombreProducto }}</td>
            <td class="num">{{ d.cantidad }}</td>
            <td class="num">{{ dinero(d.precioUnitario) }}</td>
            <td class="num">{{ dinero(d.subtotal) }}</td>
          </tr>
        </tbody>
      </table>
    </article>
  </section>
</template>
