<template>
  <v-container :max-width="$vuetify.display.width > 1900 ? '100%' : '1100px'">
        <!-- 新增课程详情弹窗 -->
    <CourseCardDetail 
      v-model="dialogVisible"
      :course="selectedCourse"
      @course-added="handleCourseAdded"
    />
    <div fluid class=" px-20  ">
      <div>
        <!-- 筛选组件 -->
        <FilterSection :types="coursesTypes" :categories="coursesCategory" :search="search"
          :selected-types="selectedCoursesTypes || []" :selected-category="selectedCoursesCategory || []"
          @update:types="updateSelectedTypes" @update:category="updateSelectedCategory" @update:search="updateSearch" />
      </div>

      <div >
        <!-- 课程列表 -->
        <v-container max-width="100%">
          <v-row :gap="0" >
            <template v-for="(course, index) in displayedCourses" :key="course.id || index">
              <!-- 超小屏幕（xs）每行1个 -->
              <!-- 小屏幕（sm）每行2个 -->
              <!-- 中等屏幕（md）每行4个 -->
              <!-- 大屏幕（lg）每行6个 -->
              <v-col cols="12" sm="6" md="4" :lg="$vuetify.display.width > 1900 ? 2 : 3" class="mb-0">
                <CourseCard v-if="course" :course="course" @view-details="viewCourseDetails" />
              </v-col>
            </template>
          </v-row>
        </v-container>
      </div>
  
      <div>
        <!-- 分页 --> <!-- 动态计算显示页码数  -->
        <v-footer>
          <v-pagination v-model="currentPage" :length="totalPages" :total-visible="100"
            @update:model-value="handlePageChange" color="primary" class="mt-4" />

        </v-footer>

      </div>


      <!-- <div v-if="loading" class="loading-overlay">
        <v-progress-circular indeterminate color="primary"></v-progress-circular>
      </div> -->
    </div>
  </v-container>
</template>

<script setup lang="ts">

import { ref, computed, watch, onMounted, onUnmounted } from 'vue'
import FilterSection from '@/components/course/FilterSection.vue'
import CourseCard from '@/components/course/CourseCard.vue'
import {
  courseService,
  type Course,
  type CourseCategory,
  type CourseType,
  type PaginatedResponse
} from '@/types/course'


// 状态管理
const selectedCoursesTypes = ref<string[]>([])
const selectedCoursesCategory = ref<string[]>([])
const search = ref<string>('')
const currentPage = ref(1)
const pageSize = 12
const loading = ref(false)

// 数据源
const coursesTypes = ref<CourseType[]>([])
const coursesCategory = ref<CourseCategory[]>([])
const coursesInfos = ref<PaginatedResponse<Course>>({
  records: [],
  total: 0,
  size: pageSize,
  current: 1,
  pages: 0
})





// 计算属性
const displayedCourses = computed(() => {
  return coursesInfos.value.records || [];
}
)

// 新增分页处理方法
const handlePageChange = (newPage: number) => {
  console.log("切换页码:", newPage); // 调试日志
  currentPage.value = newPage;
  fetchCourses()
}


// 修改分页总数计算逻辑
const totalPages = computed(() =>
  Math.max(1, coursesInfos.value.pages || 0) // 确保至少显示1页
)



// 方法封装
const updateSelectedTypes = (types: string[]) => {
  selectedCoursesTypes.value = types
  currentPage.value = 1 // 重置分页
  fetchCourses()
}

const updateSelectedCategory = (categories: string[]) => {
  selectedCoursesCategory.value = categories
  currentPage.value = 1 // 重置分页
  fetchCourses()
}

const updateSearch = (keyword: string) => {
  search.value = keyword
  currentPage.value = 1 // 重置分页
  fetchCourses()
}

const buildQueryParams = () => ({
  types: selectedCoursesTypes.value,
  categories: selectedCoursesCategory.value,
  page: currentPage.value,
  size: pageSize,
  name: search.value
})
// API 封装
const fetchCourseTypes = async () => {
  try {
    const response = await courseService.fetchAllCourseTypes()
    coursesTypes.value = response

  } catch (error) {
    console.error('获取课程类型失败:', error)
  }
}

const fetchCourseCategories = async () => {
  try {
    const response = await courseService.fetchAllCourseCategory()
    coursesCategory.value = response

  } catch (error) {
    console.error('获取课程分类失败:', error)
  }
}
const fetchCourses = async () => {
  try {
    if (currentPage.value < 1) currentPage.value = 1 // 保证页码最小为1
    loading.value = true
    const params = buildQueryParams()
    const response = await courseService.getFilteredCourses(params)
    coursesInfos.value = response
  } catch (error) {
    console.error('获取课程失败:', error)
    // 这里可以添加错误提示
  } finally {
    loading.value = false
  }
}
// 生命周期
onMounted(async () => {
  
  await Promise.all([
    fetchCourseTypes(),
    fetchCourseCategories(),
    fetchCourses(),

  ])

})

// 新增防抖函数和变量
let debounceTimeout: number;
watch(
  () => ({
    types: selectedCoursesTypes.value,
    categories: selectedCoursesCategory.value,
    keyword: search.value
  }),
  (newVal, oldVal) => {
    clearTimeout(debounceTimeout);
    debounceTimeout = window.setTimeout(() => {
      const typesEqual = arraysEqual(
        newVal.types || [],
        (oldVal?.types || [])
      );
      const categoriesEqual = arraysEqual(
        newVal.types || [],
        (oldVal?.types || [])
      );
      const searchEqual = newVal.keyword === oldVal.keyword;

      if (!typesEqual || !categoriesEqual || !searchEqual) {
        currentPage.value = 1;
        fetchCourses();
      }
    }, 500); // 防抖时间500ms
  },
  { deep: true }
);

// 在组件卸载时清除定时器
onUnmounted(() => {
  clearTimeout(debounceTimeout);
});

// 添加辅助函数
const arraysEqual = (a: string[], b: string[]) => {
  if (a === b) return true;
  if (a.length !== b.length) return false;
  return a.every((val, index) => val === b[index]);
};

// 新增状态管理
const dialogVisible = ref(false)
const selectedCourse = ref<Course | null>(null)
const viewCourseDetails = (course: Course) => {
  // router.push(`/${course.id}`)
  console.log("查看详情",course);
    selectedCourse.value = course
  dialogVisible.value = true
}

const handleCourseAdded = () => {
  fetchCourses();  // 刷新课程列表
  // 可以添加其他逻辑，例如提示添加成功
  console.log('课程添加成功，已刷新列表');
};
</script>

<style scoped>
/* 保证弹窗在滚动时位置固定 */
.v-dialog {
  position: fixed;
  top: 50% !important;
  left: 50% !important;
  transform: translate(-50%, -50%);
}

.content {
  display: flex;
  justify-content: center;
  padding: 0 16px;
}

.v-pagination {
  z-index: 1;
}

.v-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  /* 新增 */
  right: 0;
  /* 新增 */
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 16px;
  /* 添加左右内边距防止溢出 */
  box-sizing: border-box;
}
</style>