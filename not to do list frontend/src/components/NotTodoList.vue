<!-- 백엔드에서 데이터를 가져와 화면에 표시하는 컴포넌트 -->
<template>
    <div class="NotTodo-List">
        <h1>Not-To-Do List</h1>

        <!-- Not To do 항목 추가 -->
         <!-- v-model: 양방향 데이터 바인딩 지시어, 입력창에 입력되는 값을 변수로 실시간 연결 -->
        <div class="input-container">
            <input type="text" v-model="newNotTodoTitle" placeholder="안 할 일 추가" @keyup.enter="addNotTodo" />
            <button @click="addNotTodo">추가</button>
        </div>

        <!-- 하지 말아야할 목록 전체보기 -->
        <!-- v-if: Vue의 조건부 렌더링 지시어, 특정 조건이 true일 때만 해당 요소 화면 표시
             v-else: v-if가 false 일때 실행 --> 
        <p v-if="notTodos.length === 0">Loading...</p>
        <ul v-else>
            <!-- v-for: Vue의 반복문 
                 :key="todo.id": Vue가 각 리스트 항목을 효율적으로 관리 할 수 있도록 고유한 키 부여 --> 
            
            <li v-for="todo in notTodos" :key="todo.id">
                <input type="checkbox" :checked="todo.completed" @change="toggleComplete(todo)"/>
                <span :class="{ 'completed' : todo.completed }">
                    <!-- {{ }}: 데이터 바인딩 문법 -->
                    [{{ todo.title }}] {{ todo.title }}
                </span>
                <button @click="deleteNotTodo(todo.id)">삭제</button>
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

const notTodos = ref([]);       // notTodos 라는 반응형 변수 생성

//데이터를 불러오는 함수를 별로로 분리
const fetchTodos = async () => {
    try {
        // 1. GET 요청으로 백엔드에서 데이터 가져오기
        // await: 백엔드에서 응답이 올 때가지 기다렸다가 다음코드 실행
        const response = await axios.get('http://localhost:8080/nottodo');

        //2. 응답 데이터를 'notTodos' 변수에 저장
        notTodos.value = response.data;
    } catch (error) {
        console.error("Error fetching notTodos: ", error);
    }
};

const newNotTodoTitle = ref('');        //새로운 하지 말아야 할 일 제목을 저장할 반응형 변수

//새로운 항목을 추가하는 함수
const addNotTodo = async () => {
    // 입력창이 비어있으면 함수를 종료
    if (newNotTodoTitle.value.trim() === '') {
        alert('하지 말아야 할 일을 입력하세요.');
        return;
    }
    try {
        //1. POST 요청으로 백엔드에 데이터 전송
        await axios.post('http://localhost:8080/nottodo', {
            title: newNotTodoTitle.value,
            completed: false
        });
        //2. 입력창 비우기
        newNotTodoTitle.value = '';
        //3.새로운 목록을 다시 불러와 화면 업데이트
        await fetchTodos();
    } catch (error) {
        console.error("Error fetching notTodos: ", error);
    }
}

// 하지 말아야 될 일 완료 기능 추가 함수
const toggleComplete = async (todo) => {
    try {
        //1. 백엔드에 PUT 요청 보내기(completed 상태 토글)
        await axios.put(`http://localhost:8080/nottodo/${todo.id}`, {
            //...todo spread 문법: 기존 todo 객체의 모든 속송을 그대로 유지하고, 
            // compelted 속성만 !todo.completed로 바꿔서 백엔드에 보냄
            ...todo,
            completed: !todo.completed
        });
        //2. 전체 목록을 다시 불러와 화면 업데이트
        await fetchTodos();
    } catch (error) {
        console.error("Error updating todo:", error);
    }
};

// 항목 삭제 함수
const deleteNotTodo = async (id) => {
    try {
        //1. qordpsemdp DELETE 요청 보내기
        await axios.delete(`http://localhost:8080/nottodo/${id}`);
        //2. 전체 목록을 다시 불러와 화면 업데이트
        await fetchTodos();
    } catch (error) {
        console.error("Error deleting todo:", error);
    }
};

// async: 비동기 동작
onMounted(async () => {
    fetchTodos();
});
</script>
<!-- scoped: 해당컴포넌트 내부의 스타일만 적용 -->
<style scoped>
    .todo-list {
        font-family: Arial, sans-serif;
        padding: 20px;
    }

    .completed {
        text-decoration: line-through;
        color: #888;
    }
</style>