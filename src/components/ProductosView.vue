<script setup>
import { ref, reactive, onMounted } from 'vue'
import { productosApi, dinero } from '../api'

const productos = ref([])
const error = ref('')
const aviso = ref('')
const editandoId = ref(null)
const form = reactive({ nombre: '', precio: '', stock: '' })

async function cargar() {
  try {
    productos.value = await productosApi.listar()
  } catch (e) {
    error.value = 'No se pudo conectar con la API: ' + e.message
  }
}

function limpiarForm() {
  editandoId.value = null
  form.nombre = ''
  form.precio = ''
  form.stock = ''
}

function editar(p) {
  editandoId.value = p.id
  form.nombre = p.nombre
  form.precio = p.precio
  form.stock = p.stock
  error.value = ''
}

async function guardar() {
  error.value = ''
  aviso.value = ''
  const datos = {
    nombre: form.nombre,
    precio: form.precio === '' ? null : Number(form.precio),
    stock: form.stock === '' ? null : Number(form.stock),
  }
  try {
    if (editandoId.value) {
      await productosApi.actualizar(editandoId.value, datos)
      aviso.value = 'Producto actualizado'
    } else {
      await productosApi.crear(datos)
      aviso.value = 'Producto creado'
    }
    limpiarForm()
    await cargar()
  } catch (e) {
    error.value = e.message
  }
}

async function eliminar(p) {
  if (!confirm(`¿Eliminar "${p.nombre}"?`)) return
  error.value = ''
  aviso.value = ''
  try {
    await productosApi.eliminar(p.id)
    aviso.value = 'Producto eliminado'
    if (editandoId.value === p.id) limpiarForm()
    await cargar()
  } catch (e) {
    error.value = e.message
  }
}

onMounted(cargar)
</script>

<template>
  <section class="panel">
    <h2>{{ editandoId ? `Editar producto #${editandoId}` : 'Nuevo producto' }}</h2>
    <form class="form-grid" @submit.prevent="guardar">
      <label>
        Nombre
        <input v-model="form.nombre" placeholder="Teclado Mecánico RGB" />
      </label>
      <label>
        Precio
        <input v-model="form.precio" type="number" step="0.01" placeholder="1250.50" />
      </label>
      <label>
        Stock
        <input v-model="form.stock" type="number" step="1" placeholder="15" />
      </label>
      <div class="acciones">
        <button type="submit" class="primario">{{ editandoId ? 'Guardar cambios' : 'Crear' }}</button>
        <button v-if="editandoId" type="button" @click="limpiarForm">Cancelar</button>
      </div>
    </form>
    <p v-if="error" class="error">{{ error }}</p>
    <p v-if="aviso" class="ok">{{ aviso }}</p>
  </section>

  <section class="panel">
    <h2>Productos ({{ productos.length }})</h2>
    <table>
      <thead>
        <tr><th>ID</th><th>Nombre</th><th class="num">Precio</th><th class="num">Stock</th><th></th></tr>
      </thead>
      <tbody>
        <tr v-for="p in productos" :key="p.id">
          <td>{{ p.id }}</td>
          <td>{{ p.nombre }}</td>
          <td class="num">{{ dinero(p.precio) }}</td>
          <td class="num" :class="{ agotado: p.stock === 0 }">{{ p.stock }}</td>
          <td class="acciones-fila">
            <button @click="editar(p)">Editar</button>
            <button class="peligro" @click="eliminar(p)">Eliminar</button>
          </td>
        </tr>
        <tr v-if="productos.length === 0">
          <td colspan="5" class="vacio">No hay productos</td>
        </tr>
      </tbody>
    </table>
  </section>
</template>
