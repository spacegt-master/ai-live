<!-- CourseCard.vue -->
<template>
  <v-card  
    class="mx-auto "
    max-width="220">
    <!-- 课程封面照片 -->
    <v-img :src="course.thumbnail" 
    height="110px"  
    cover ></v-img>
     <v-card-item>
    <v-list-item class="w-100">
        <template v-slot:prepend>
          <v-avatar
            color="grey-darken-3"
          :image="course.teacherIcon"
          ></v-avatar>
        </template>

        <v-list-item-title> {{course.name}}</v-list-item-title>
 
        <v-list-item-subtitle>{{"("+course.teacher+")"}}</v-list-item-subtitle>
        <template v-slot:append>
          <div class="justify-self-end">
          </div>
        </template>
      </v-list-item>

     <v-card-text
     style="overflow-y: hidden ; max-height: 50px;
      font-size: 12px;    
    line-height: 1.3;  
     "
     >
     {{ "简介："+course.introduce }}
    </v-card-text>
   <v-card-actions >
        <v-btn color="secondary" @click="$emit('view-details', course)">
          查看课程详情
        </v-btn>
      </v-card-actions>

     </v-card-item>
  
    </v-card>
</template>

<script setup lang="ts">
import type { Course } from '@/types/course';
import { defineProps, defineEmits } from 'vue'


const props = defineProps<{
  course: Course
}>()

const emit = defineEmits<{
  (e: 'view-details', course: Course): void
}>()
</script>

<style scoped>

</style>