<script setup>
import { ref } from 'vue'
import ProductosView from './components/ProductosView.vue'
import VentasView from './components/VentasView.vue'

const vista = ref('productos')
</script>

<template>
  <header class="topbar">
    <h1>Inventario</h1>
    <nav>
      <button :class="{ activo: vista === 'productos' }" @click="vista = 'productos'">Productos</button>
      <button :class="{ activo: vista === 'ventas' }" @click="vista = 'ventas'">Ventas</button>
    </nav>
  </header>

  <main class="contenido">
    <!-- KeepAlive no se usa a propósito: al cambiar de pestaña se recargan los datos (stock actualizado) -->
    <ProductosView v-if="vista === 'productos'" />
    <VentasView v-else />
  </main>
</template>
