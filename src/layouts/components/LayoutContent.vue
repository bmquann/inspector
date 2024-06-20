<template>
  <n-layout-content >
    <router-view v-slot="{ Component }">
      <keep-alive>
        <component :is="Component" />
      </keep-alive>
    </router-view>
  </n-layout-content>
</template>
