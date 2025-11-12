<template>
  <v-container>
    <v-card>
      <v-card-text>
        <v-data-table-server v-model:options="options" :items-per-page="options.itemsPerPage" :headers="headers"
          item-key="id" :items="serverItems" :items-length="totalItems" :loading="loading"
          :search="`${search.category},${search.type},${search.name},${search.state}`"
          :mobile="$vuetify.display.smAndDown" @update:options="loadItems">
          <template v-slot:top>
            <v-row>
              <v-col>
                <v-text-field hide-details v-model="search.name" class="pa-2" label="请输入课程名称"
                  append-inner-icon="mdi-magnify" clearable></v-text-field>
              </v-col>
              <v-col>
                <v-select hide-details v-model="search.type" class="pa-2" label="课程类型" :items="courseStore.types"
                  item-title="name" item-value="id" clearable></v-select>
              </v-col>
              <v-col>
                <v-select hide-details v-model="search.category" class="pa-2" label="课程分类" :items="courseStore.classify"
                  item-title="name" item-value="id" clearable></v-select>
              </v-col>
              <v-col>
                <v-select hide-details v-model="search.state" class="pa-2" label="课程状态" :items="status"
                  item-title="name" item-value="id" clearable></v-select>
              </v-col>
              <v-col style="text-align: end;">
                <v-btn style="margin-top: 19px; margin-right: 10px;" class="pa-2" :color="'primary'">批量删除</v-btn>
                <v-btn style="margin-top: 19px;" class="pa-2" :color="'primary'" @click="editItem(null)">新增课程</v-btn>
              </v-col>
            </v-row>
          </template>
          <template v-slot:item.thumbnail="{ item }">
            <v-img :src="item.thumbnail" :width="80" :height="45" cover>
              <template v-slot:placeholder>
                <v-img src="https://cdn.vuetifyjs.com/images/parallax/material.jpg"></v-img>
              </template>
              <template v-slot:error>
                <v-img src="https://cdn.vuetifyjs.com/images/parallax/material.jpg"></v-img>
              </template>
            </v-img>
          </template>
          <template v-slot:item.state="{ item }">
            <span v-if="item.state === 0" class="status-pending">审核中</span>
            <span v-else-if="item.state === 1" class="status-approved">已通过</span>
            <span v-else class="status-rejected">未通过</span>
          </template>
          <template v-slot:item.actions="{ item }">
            <VBtn prepend-icon="mdi-pencil" :color="'purple-accent-3'" variant="text">
              章节管理
              <ChapterAdd :courseId="item.id" />
            </VBtn>
            <VBtn prepend-icon="mdi-check-circle" :color="'yellow-darken-4'" v-if="item.isPublished === 0"
              variant="text" text="已发布"></VBtn>
            <VBtn prepend-icon="mdi-publish" :color="'yellow-darken-4'" v-else variant="text" text="发布"
              @click="editItem(item)"></VBtn>
            <!-- <VBtn prepend-icon="mdi-magnify-expand" :color="'green-darken-3'" variant="text" text="预览"
            @click="editItem(item)"></VBtn> -->
            <VBtn prepend-icon="mdi-pencil" :color="'light-blue-darken-4'" variant="text" text="编辑"
              @click="editItem(item)"></VBtn>
            <VBtn prepend-icon="mdi-delete" :color="'red'" variant="text" text="删除" @click="deleteItem(item)"></VBtn>
          </template>
        </v-data-table-server>
      </v-card-text>
      <v-dialog v-model="dialog" max-width="600px" :persistent="loadingEdit" :fullscreen="$vuetify.display.smAndDown"
        scrollable>
        <v-card :loading="loadingEdit">
          <v-card-title>
            <span class="text-h5">{{ formTitle }}</span>
          </v-card-title>
          <v-card-text>
            <v-container v-if="editedItem">
              <v-row>
                <v-col cols="12">
                  <v-text-field v-model="editedItem.name" label="标题" :disabled="loadingEdit"></v-text-field>
                </v-col>
                <v-col cols="12">
                  <v-hover v-if="editedItem.thumbnail" v-slot="{ isHovering, props }">
                    <v-card v-bind="props" color="surface-light" height="300px">
                      <v-img :src="useFileUri(editedItem.thumbnail)" height="300px"></v-img>
                      <v-btn icon="mdi-close" class="opacity-0 position-absolute" :class="{ 'opacity-100': isHovering }"
                        style="left: 50%; top: 50%; transform: translate(-50%,-50%);" :disabled="loadingEdit"
                        @click="editedItem.thumbnail = null; coverFile = null"></v-btn>
                    </v-card>
                  </v-hover>
                  <v-file-upload v-else v-model="coverFile" :disabled="loadingEdit" density="comfortable" title="封面"
                    height="300px" accept=".png,.jpg" @update:model-value="handleCoverFileUpdate"></v-file-upload>
                </v-col>
                <v-col cols="12" sm="6">
                  <v-select v-model="editedItem.type" label="课程分类" :items="courseStore.types" item-title="name"
                    item-value="id" :disabled="loadingEdit"></v-select>
                </v-col>
                <v-col cols="12" sm="6">
                  <v-select v-model="editedItem.classifyId" label="课程类型" :items="courseStore.classify" item-title="name"
                    item-value="id" :disabled="loadingEdit"></v-select>
                </v-col>
                <v-col cols="12">
                  <v-textarea v-model="editedItem.introduce" label="课程介绍" :disabled="loadingEdit"></v-textarea>
                </v-col>
              </v-row>
            </v-container>
            <small class="text-caption text-medium-emphasis" v-show="loadingEdit">* 等待中请勿关闭窗口或刷新页面</small>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue-darken-1" variant="text" :disabled="loadingEdit" @click="dialog = false">
              取消
            </v-btn>
            <v-btn color="blue-darken-1" variant="text" :loading="loadingEdit" @click="save">
              保存
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-dialog v-model="dialogDelete" max-width="500px">
        <v-card>
          <v-card-title class="text-h5">您确定要删除此项目吗？</v-card-title>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue-darken-1" variant="text" @click="dialogDelete = false">取消</v-btn>
            <v-btn color="blue-darken-1" variant="text" @click="deleteItemConfirm">确定</v-btn>
            <v-spacer></v-spacer>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-card>
  </v-container>
</template>

<script setup lang="ts">
  import {
    ref,
    computed,
    onMounted
  } from 'vue';
  import {
    courseApi
  } from '@/api/course';
  import { useFileUri } from '@/utils/file-uri';
  import { FileApi } from '@/api/file';
  import { useObjectUrl } from '@vueuse/core';
  import { VFileUpload } from 'vuetify/labs/VFileUpload';
  import { useCourseStore } from '@/stores/course';

  interface CourseItem {
    id : string
    name : string
    type : any
    typeName : string | null
    classifyId : any
    classifyName : string | null
    state : number | null
    isPublished : number | null
    thumbnail : any
    introduce : string | null
    courseCode : string | null
    url : string | null
    urlOld : string | null
    file : string | null
    inputType : 'file'
  }

  const courseStore = useCourseStore()
  const totalItems = ref(0)
  const loaded = ref(false)
  const loading = ref(false)
  const loadingEdit = ref(false)
  const dialogDelete = ref(false)
  const dialogChapter = ref(false)
  const dialog = ref(false)
  const editedIndex = ref(-1)
  const types = ref<string[]>([])
  const classify = ref<string[]>([])
  const coverFile = ref()
  const currentChapter = ref(null)

  const options = ref({
    page: 1,
    itemsPerPage: 10
  })

  const headers : any = [{
    title: '序号',
    align: 'start',
    sortable: false,
    key: 'index',
  },
  {
    title: '课程封面',
    sortable: false,
    key: 'thumbnail',
    nowrap: true
  },
  {
    title: '课程分类',
    key: 'typeName',
  },
  {
    title: '课程类型',
    key: 'classifyName',
  },
  {
    title: '标题',
    key: 'name',
  },
  {
    title: '课程代码',
    key: 'courseCode',
  },
  {
    title: '创建时间',
    key: 'createTime',
  },
  {
    title: '审核状态',
    key: 'state'
  },
  {
    title: '操作',
    key: 'actions',
    sortable: false,
    align: 'center'
  }]

  const search = ref({
    name: '',
    category: null,
    type: null,
    state: null
  })

  const status = ref([
    { id: 0, name: '审核中' }, { id: 1, name: '已通过' }, { id: 2, name: '未通过' }
  ])
  const serverItems = ref<CourseItem[]>([])
  const formTitle = computed(() => editedIndex.value === -1 ? '新增课程' : '编辑课程')
  const editedItem = ref<Partial<CourseItem>>()
  const defaultItem = ref<Partial<CourseItem>>()

  const editItem = async (item : any) => {
    if (item) {
      const itemCopy = { ...item }
      delete itemCopy.createTime
      editedIndex.value = serverItems.value.indexOf(itemCopy)
      editedItem.value = Object.assign({}, itemCopy)
      if (editedItem.value) {
        editedItem.value.urlOld = editedItem.value?.url
        editedItem.value.inputType = 'file'
      }
    } else {
      editedItem.value = Object.assign({}, defaultItem.value)
      editedIndex.value = -1;
    }
    dialog.value = true
  }

  const deleteItem = async (item : any) => {
    editedItem.value = Object.assign({}, item)
    dialogDelete.value = true;
  }

  const deleteItemConfirm = async () => {
    if (editedItem.value && editedItem.value.id) {
      await courseApi.del(editedItem.value.id)
      loadItems(options.value)
      dialogDelete.value = false;
    }
  }

  const save = async () => {
    loadingEdit.value = true
    try {
      if (coverFile.value) {
        const coverConfig = await FileApi.upload(coverFile.value, 'live/course/cover', true) as any
        if (editedItem.value)
          editedItem.value.thumbnail = FileApi.filePath + coverConfig.url
      }
      await courseApi.save({
        ...editedItem.value
      })
      loadItems(options.value)
      dialog.value = false
    } catch (e) {
      /* empty */
    }
    loadingEdit.value = false
  }

  const loadItems = async ({ page, itemsPerPage } : any) => {
    loading.value = true
    await courseStore.loadTypes()
    await courseStore.loadClassifys()

    const res = await courseApi.manage({
      types: search.value.type,
      categories: search.value.category,
      page: page,
      size: itemsPerPage,
      name: search.value.name,
      state: search.value.state
    }) as any
    serverItems.value = res.records.map((item : any, index : number) => {
      return {
        ...item,
        index: (page - 1) * 10 + index + 1,
      }
    })
    /* console.log(serverItems.value); */
    totalItems.value = res.total
    loading.value = false
  }

  onMounted(() => {
    loadItems(options.value)
  })

  const handleCoverFileUpdate = (file : any) => {
    if (editedItem.value)
      editedItem.value.thumbnail = useObjectUrl(file).value
  }
</script>

<style scoped>
  .status-pending {
    color: orange;
  }

  .status-approved {
    color: green;
  }

  .status-rejected {
    color: red;
  }
</style>
