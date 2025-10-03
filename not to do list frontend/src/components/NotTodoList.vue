<!-- 백엔드에서 데이터를 가져와 화면에 표시하는 컴포넌트 -->
<template>
    <div class="list-container">
        <h1>Not-To-Do List</h1>

        <!-- Not To do 항목 추가 -->
         <!-- v-model: 양방향 데이터 바인딩 지시어, 입력창에 입력되는 값을 변수로 실시간 연결 -->
        <div class="input-container">
            <input type="text" v-model="newNotTodoTitle" placeholder="안 할 일 추가" @keyup.enter="addNotTodo" />
            <button @click="addNotTodo">추가</button>
        </div>

        <!-- 완료 상태에 따른 필터링 -->
        <div class="filters">
            <button @click="filter = 'all'">전체</button>
            <button @click="filter = 'completed'">완료</button>
            <button @click="filter = 'active'">미완료</button>
        </div>

        <!-- 안 할 일 개수 표시 -->
        <p class="notTodo-count">남은 안 할 일 : {{ notTodos.length }}개</p>

        <!-- 안 할 일 목록 전체보기 -->
        <!-- v-if: Vue의 조건부 렌더링 지시어, 특정 조건이 true일 때만 해당 요소 화면 표시
             v-else: v-if가 false 일때 실행 --> 
        <div v-if="notTodos.length === 0" class="loading-message">
            <p>고치고 싶은 습관을 추가해 보세요.</p>
        </div>
        <ul v-else>
            <!-- v-for: Vue의 반복문 
                 :key="notTodo.id": Vue가 각 리스트 항목을 효율적으로 관리 할 수 있도록 고유한 키 부여 --> 
            
            <li v-for="notTodo in filteredNotTodos" :key="notTodo.id">
                <input type="checkbox" :checked="notTodo.completed" @change="toggleComplete(notTodo)"/>
                
                <!-- 안 할 일 항목 수정 -->
                <!-- blur: 입력창에서 포커스가 벗어나면 함수 호출 -->
                 <!--  -->
                 <input
                    v-if="editingNotTodoId === notTodo.id"
                    type="text"
                    v-model="notTodo.title"
                    @keyup.enter="updateNotTodo(notTodo)"
                    @focus="isFocusing = true"
                    @blur="isFocusing ? updateNotTodo(notTodo) : null"
                    ref="editingInput" 
                 />
                 <!-- v-bind:class: 조건에 따른 클래스 적용 여부 결정 -->
                <span 
                    v-else
                    @dblclick.prevent.stop="editNotTodo(notTodo)"
                    :class="{ 'completed' : notTodo.completed }"
                    class="NotTodo-title-with-progress"
                    :style="{backgroundColor: getCompletionColor(notTodo)}"
                    >     
                    {{ notTodo.title }}        <!-- {{ }}: 데이터 바인딩 문법 -->
                </span>

                <div class="completion-controls">
                    <button @click="toggleTodayCompletion(notTodo)" class="today-complete-button">
                        {{ isTodayCompleted(notTodo) ? '오늘 달성 취소' : '오늘 달성' }}
                    </button>
                    <button @click="openCalendar(notTodo.id)" class="calendar-button">📅</button>
                </div>

                <button @click="deleteNotTodo(notTodo.id)">삭제</button>

            </li>
        </ul>
    </div>
</template>


<!-- setup: 변수나 함수를 따로 return 할 필요 없음 -->
<script setup>
    // ref: 일반 변수를 반응형(Reactive)으로 만듬
    // onMounted: 컴포넌트가 화면에 처음 나타났을때 특정 코드를 실행하도록 하는 함수
    import { ref, onMounted, nextTick, computed } from 'vue';
    import axios from 'axios'       // axios: 백엔드와 HTTP통신을 하기위한 라이브러리
    import { format, differenceInDays, addDays, parseISO} from 'date-fns';

    const notTodos = ref([]);               // notTodos 라는 반응형 변수 생성
    const newNotTodoTitle = ref('');        // 새로운 안 할 일 제목을 저장할 반응형 변수
    const editingNotTodoId = ref(null);     // 현재 수정 중인 안 할 일의 ID를 저장한 변수
    const filter =ref('all');               // 필터 상태를 저장할 변수
    const editingInput = ref(null);         // 목록 내용 편집
    const isFocusing = ref(false);          // 현재 포커스 중인지 확인

    // 데이터를 불러오는 함수를 별로로 분리
    const fetchNotTodos = async () => {
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

    
    // 새로운 항목을 추가 함수
    const addNotTodo = async () => {
        // 입력창이 비어있으면 함수를 종료
        if (newNotTodoTitle.value.trim() === '') {
            alert('안 할 일을 입력하세요.');
            return;
        }
        try {
            // 1. POST 요청으로 백엔드에 데이터 전송
            await axios.post('http://localhost:8080/nottodo', {
                title: newNotTodoTitle.value,
                completed: false
            });
            
            // 2. 입력창 비우기
            newNotTodoTitle.value = '';

            // 3. nextTick을 사용하여 DOM이 업데이트된 후 커서 위치시키기
            await nextTick(() => {
                document.querySelector('.input-container input').focus();
            });
            
            // 4.새로운 목록을 다시 불러와 화면 업데이트
            await fetchNotTodos();
        } catch (error) {
            console.error("Error fetching notTodos: ", error);
        }
    }

    // 안 할 일 완료 기능 함수
    const toggleComplete = async (notTodo) => {
        try {
            // 1. 백엔드에 PUT 요청 보내기(completed 상태 토글)
            await axios.put(`http://localhost:8080/nottodo/${notTodo.id}`, {
                // ...NotTodo spread 문법: 기존 todo 객체의 모든 속송을 그대로 유지하고, 
                // compelted 속성만 !NotTodo.completed로 바꿔서 백엔드에 보냄
                ...notTodo,
                completed: !notTodo.completed
            });
            // 2. 전체 목록을 다시 불러와 화면 업데이트
            await fetchNotTodos();
        } catch (error) {
            console.error("Error updating todo:", error);
        }
    };

    // 항목 삭제 함수
    const deleteNotTodo = async (id) => {
        try {
            // 1. qordpsemdp DELETE 요청 보내기
            await axios.delete(`http://localhost:8080/nottodo/${id}`);
            // 2. 전체 목록을 다시 불러와 화면 업데이트
            await fetchNotTodos();
        } catch (error) {
            console.error("Error deleting todo:", error);
        }
    };

    // 안 할 일 항목 수정 (자동 select 되면서 입력안되는거 수정)
    const editNotTodo = (notTodo) => {

        isFocusing.value = false;
        console.log(`[1] editTodo 진입: ID ${notTodo.id}`);
        // 1. 수정 모드 진입
        editingNotTodoId.value = notTodo.id;

        nextTick(() => {
            
            let inputElement;
            
            // 2. ref를 통해 DOM 요소 접근
            if (Array.isArray(editingInput.value)) {
                inputElement = editingInput.value[editingInput.value.length - 1];
            } else {
                inputElement = editingInput.value;
            }

            // 3. 포커스 설정
            if(inputElement) {
                inputElement.focus();
                //inputElement.select();

                //4. 텍스트 선택 방지 및 커서 위치 고정
                const len = inputElement.value.length;
                inputElement.setSelectionRange(len, len);

                //5. isFocusing 상태를 켜서 @blur 활성화
                isFocusing.value = true;
            } 
        });
    };

    // 안 할 일 수정을 완료하고 백엔드에 업데이트하는 함수
    const updateNotTodo = async (notTodo) => {

        // isFocusing이 false면 즉시 함수를 종료하여 @blur 무시      
        if (!isFocusing.value) {
            return;
        }
        isFocusing.value = false;

        try {
            // 1. PUT 요청으로 백엔드에 수정된 데이터 전송
            await axios.put(`http://localhost:8080/nottodo/${notTodo.id}`, notTodo);
            
            // 2. 수정 모드 종료
            editingNotTodoId.value = null;

            // 3. 업데이트된 목록을 다시 불러와 화면 업데이트
            await fetchNotTodos();
        } catch (error) {
            console.error("Error updating NotTodo:", error);
        }

        
    };

    // 안 할 일 목록 필터링 추가
    // computed: filter나 notTodos 데이터 변경 시 자동으로 계산
    const filteredNotTodos = computed(() => {
        if (filter.value === 'completed') {
            return notTodos.value.filter(notTodo => notTodo.completed);
        } else if (filter.value === 'active') {
            return notTodos.value.filter(notTodo => !notTodo.completed);
        } else {
            return notTodos.value;      //all인 경우
        }
    });

    // 오늘 날짜를 가져오는 함수
    const getTodayDateString = () => format(new Date(), 'yyyy-MM-dd');

    // 날짜 범위 생성 함수(등록일부터 오늘까지)
    const getAllDatesFromRegistration = (startDateString) => {
        const dates = [];
        let currentDate = parseISO(startDateString);       //날짜문자열 Date 객체로 파싱
        const today = new Date();       // 오늘 날짜를 Date 객체로 가져옴

        while (currentDate <= today) {
            dates.push(format(currentDate, 'yyyy-MM-dd'));
            currentDate = addDays(currentDate, 1);  // addDays: 특정날짜에 지정된 일 수 더해줌
        }
        return dates;
    };

    // 각 NotTodo 달성률을 계산하고 색상을 반환하는 함수 
    const getCompletionColor = (notTodo) => {
        const allPossibleDates = getAllDatesFromRegistration(notTodo.createdAt);
        const totalDays = allPossibleDates.length;

        if (totalDays === 0) return '#ffffff' // 등록일이 오늘인 경우(qnsah 0 방지)

        const compeltedCount = (notTodo.completonDates || []).length;
        const completionRate = compeltedCount / totalDays;      // 달성률 계산

        // 달성률에 따른 색상 변환 (연한 초록 -> 진한 초록)
        if (completionRate === 0) return '#f0f0f0';
        if (completionRate < 0.25) return '#d4edda';
        if (completionRate < 0.5) return '#a2e0ae';
        if (completionRate < 0.75) return '#70d283';
        return '#28a745'
    };

    // 오늘 날짜 달성 여부 확인 (체크박스용)
    const isTodayCompleted = (notTodo) => {
        if(!notTodo.completionDates) return false;
        return notTodo.completionDates.includes(getTodayDateString());
    };

    // 오늘 날짜 달성 토글 (백엔드 통신)
    const toggleTodayCompletion = async (notTodo) => {
        const today = getTodayDateString();
        const currentCompletionDates = new Set(notTodo.completionDates || []);

        if (currentCompletionDates.has(today)) {
            currentCompletionDates.delete(today);
        } else {
            currentCompletionDates.add(today);
        }

        try {
            const updateNotTodo = {
                ...notTodo,
                completionDates: Array.from(currentCompletionDates) //Set을 다시 배열로 변환
            };
            await axios.put(`http://localhost:8080/nottodo/${notTodo.id}`, updateNotTodo);
            await fetchNotTodos();
        } catch (error) {
            console.error("Error updating completion dates:", error);
        }
    };

    //캘린더 팝업 열기 함수(아직 미구현)\
    const openCalendar = (id) => {
        console.log(`open calendar for NotTodo ID: ${id}`);
    };



    // async: 비동기 동작
    onMounted(async () => {
        fetchNotTodos();
    });

</script>
<!-- scoped: 해당컴포넌트 내부의 스타일만 적용 -->
<style scoped>
    /* 전체 컨테이너에 적용할 스타일 */
    .list-container {
        max-width: 600px;
        margin: 40px auto;
        padding: 20px;
        background: #f9f9f9;
        border-radius: 23px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);  /* 요소 주변에 그림자 추가 (가로 오프셋, 세로 오프셋, 흐림, 색상 */
    }                                                /* rgba: rgb 색상+불투명도 */

    /* 입력창과 추가 버튼을 감싸는 컨테이너 */
    .input-container {
        display: flex;
        gap: 10px;      /* 아이템들 사이에 10px의 간격을 둠 */
        margin-bottom: 20px;
    }

    /* 안할 일 입력창 스타일 -> 입력창을 공통 CSS로 사용, 추후 필요시 분리예정 */
    input[type="text"] {
        flex-grow: 1;       /* flex 요소의 자식들이 남는 공간을 어떤 비율로 차지할지 설정, 기본값 0  
                                input과 button의 비율은 1:0 */
        padding: 10px;
        border: 1px solid #ccc;
        border-radius: 4px;
    } 

    /* 안할 일 추가 버튼 스타일 */
    .input-container button, 
    li button {
        padding: 10px 15px;
        border: none;
        border-radius: 4px;
        color: white;
        background-color: #3498db;
        cursor: pointer;
    }

    /* 할 일 목록 스타일 */
    ul {
        list-style: none;
        padding: 0;
    }

    li {
        display: flex;
        align-items: center;
        gap: 10px;
        padding: 15px;
        margin-bottom: 10px;
        background: white;
        border-radius: 4px;
        transition: all 0.3s ease;      /* 상태변화 애니메이션 효과
                                            all: 모든 속성, 0.3s: 애니메이션 걸리는 시간, ease: 시작과 끝은 느리고 중간은 빠름 */
    }
    
    li:hover {
        background-color: #f1f1f1;
    }

    /* 완료된 할 일 텍스트에 취소선 적용 */
    .completed {
        text-decoration: line-through;
        color: #888;
    }
    
    /* 안 할 일 텍스트를 감싸는 span 태그 스타일 */
    li span {
        flex-grow: 1;
    }


    /* 삭제 버튼 스타일 */
    li button {
        background-color: #e74c3c;
        font-size: 12px;
    }

    /* loading-message */
    .loading-message {
        text-align: center;
        color: #888;
        font-style: italic;
        margin-top: 50px;
    }

    /* 안 할 일 개수 표시 스타일 */
    .notTodo-count {
        text-align: right;
        color: #666;
        font-size: 14px;
        margin-bottom: 10px;
    }

    /* 필터 버튼 컨테이너 */
    .filters {
        display: flex;
        gap: 10px;
        justify-content: left;
        margin-bottom: 20px;        
    }

    /* 필터 버튼 스타일 */
    .filters button {
        padding: 8px 15px;
        border: 1px solid #ccc;
        border-radius: 4px;
        background-color: white;
        cursor: pointer;
        transition: all 0.3s ease;
    }

    .filters button:hover {
        background-color: #f0f0f0;
    }

    /* 달성률  */
    .NotTodo-title-with-progress {
        flex-grow: 1;
        padding: 8px 12px;
        border-radius: 4px;
        margin-right: 10px;
        transition: background-color 0.3s ease;
        display: flex;
        align-items: center;
    }

    .completion-controls {
        display: flex;
        gap: 5px;
        margin-left: auto;   /* 버튼을 오른쪽으로 밀어냅니다 */
    }

    .today-complete-button, .calendar-button {
        padding: 8px 12px;
        border: 1px solid #ddd;
        border-radius: 4px;
        background-color: #f8f8f8;
        cursor: pointer;
        transition: background-color 0.2s;
    }

    .today-complete-button:hover, .calendar-button:hover {
        background-color: #e0e0e0;
    }
   

</style>