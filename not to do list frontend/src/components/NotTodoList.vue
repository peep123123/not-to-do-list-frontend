<!-- 백엔드에서 데이터를 가져와 화면에 표시하는 컴포넌트 -->
<template>
    <div class="NotTodo-List">
        <h1>Not-To-Do List</h1>

        <!-- v-if: Vue의 조건부 렌더링 지시어, 특정 조건이 true일 때만 해당 요소 화면 표시
             v-else: v-if가 false 일때 실행 --> 
        <p v-if="notTodos.length === 0">Loading...</p>
        <ul v-else>
            <!-- v-for: Vue의 반복문 
                 :key="todo.id": Vue가 각 리스트 항목을 효율적으로 관리 할 수 있도록 고유한 키 부여 --> 
            
            <li v-for="todo in notTodos" :key="todo.id">
                <!-- {{ }}: 데이터 바인딩 문법 -->
                {{ todo.title }}
            </li>
        </ul>
    </div>
</template>

<!-- setup: 변수나 함수를 따로 return 할 필요 없음 -->
<script setup>
// ref: 일반 변수를 반응형(Reactive)으로 만듬
// onMounted: 컴포넌트가 화면에 처음 나타났을때 특정 코드를 실행하도록 하는 함수
// axios: 백엔드와 HTTP통신을 하기위한 라이브러리
import { ref, onMounted } from 'vue';
import axios from 'axios'

// notTodos 라는 반응형 변수 생성
const notTodos = ref([]);

// async: 비동기 동작
onMounted(async () => {
    try {
        // 백엔드 GET API 호출
        // await: 백엔드에서 응답이 올 때가지 기다렸다가 다음코드 실행
        const response = await axios.get('http://localhost:8080/nottodo');
        
        // 응답 데이터를 'notTodos' 변수에 저장
        notTodos.value = response.data;
    } catch (error) {
        // 오류 발생 시 콘솔에 에러 메시지 출력
        console.error("Error fetching notTodos:", error)
    }
});
</script>
<!-- scoped: 해당컴포넌트 내부의 스타일만 적용 -->
<style scoped>
    .todo-list {
        font-family: Arial, sans-serif;
        padding: 20px;
    }
</style>