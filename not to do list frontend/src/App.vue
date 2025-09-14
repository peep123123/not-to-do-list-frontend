<!-- Vue 애플리케이션의 최상위 컴포넌트, 모든 컴포넌트들이 App.vue에 포함됨 -->
<template>
  <main>
    <!-- script에서 가져온 NotTodoList 컴포넌트를 실제로 화면에 표시하라는 명령 -->
    <NotTodoList />
  </main>
</template>

<script setup>
// 경로에서 NotTodoList.vue 파일을 가져와 NotTodoList라는 이름으로 사용
import NotTodoList from './components/NotTodoList.vue';
</script>

<style>
/* 전역스타일 */
</style>