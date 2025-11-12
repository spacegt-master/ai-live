<template>
  <v-dialog v-model="dialog" activator="parent" max-width="800px" scrollable :fullscreen="$vuetify.display.smAndDown">
    <v-card>
      <v-card-title style="text-align: center;" class="text-h5">AI生成直播课</v-card-title>
      <v-card-text v-if="chapters.length > 0">
        <v-list>
          <template v-for="item in chapters" :key="item.id">
            <v-list-group v-if="item.sections" :value="item.id">
              <template>
                <v-list-item class="d-flex justify-space-between align-center">
                  <v-list-item-title>{{ item.title }}</v-list-item-title>
                  <!-- <template v-slot:append>
                    <spacegt-aippt @click.stop @export-images="handleExportImages" />
                  </template> -->
                </v-list-item>
              </template>

              <v-list-item class="d-flex justify-space-between align-center" v-for="child in item.sections"
                :key="child.id" :value="child.id">
                <v-list-item-title>{{ child.title }}</v-list-item-title>
                <!-- <template v-slot:append>
                  <spacegt-aippt @click.stop @export-images="handleExportImages" />
                </template> -->
              </v-list-item>
            </v-list-group>

            <v-list-item class="d-flex justify-space-between align-center" v-else :value="item.id">
              <v-list-item-title>{{ item.title }}</v-list-item-title>
              <template v-slot:append>
                <spacegt-aippt @click.stop="save(item.id)" @export-images="handleExportImages"
                  @create-outline="handleCreateOutline" @createPPT="handleCreatePPT" />
              </template>
            </v-list-item>
          </template>
        </v-list>

      </v-card-text>
      <v-card-text class="d-flex justify-space-between align-center">
        <!-- <span>添加章节</span> -->
        <v-card-title>添加章节</v-card-title>

        <spacegt-aippt @click.stop @export-images="handleExportImages" @create-outline="handleCreateOutline"
          @createPPT="handleCreatePPT" />
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="blue" @click="dialog = false">关闭</v-btn>
        <v-btn color="blue" @click="saveChapter">保存</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import {
  courseApi
} from '@/api/course';
import {
  ChapterApi
} from '@/api/chapter';
import {
  PPTApi
} from '@/api/ppt';
import { FileApi } from '@/api/file';

const props = defineProps<{
  courseId?: number | string
}>()

const dialog = ref(false)
const loading = ref(false)
const loadingEdit = ref(false)
const serverItems = ref<Chapter[]>([])
interface Section {
  id: number
  title: string
}

interface Chapter {
  id: number
  title: string
  detailed: string
  courseId: number
  sections: Section[]
  outline: string
  pptjson: string
  thumbnail: string
}

interface PPT {
  id: number
  title: string
  pageNumber: number
  chapterId: number
  imagePath: string
  contentText: string
  pptjson: string
}

const chapters = ref()
const textOutput = ref('')
const coverFile = ref(null)
const editedItem = ref<Partial<Chapter>>()
const pptItem = ref<Partial<PPT>>()
const outline = ref()
const pptjson = ref()
const urls = ref()
const chapterId = ref()

const handleExportImages = async (data: any) => {
  try {
    const uploadPromises = data.map(async (item: string) => {
      const coverConfig = await FileApi.upload(base64ToFile(item), 'live/ppt', true) as any;
      return FileApi.filePath + coverConfig.url;
    });

    urls.value = await Promise.all(uploadPromises);

    const thumbnail = urls.value.find((item: any, index: number) => index === 0)

    const chid = await ChapterApi.save({
      ...editedItem.value,
      id: chapterId.value,
      courseId: props.courseId,
      title: pptjson.value.find((item: { type: string; })  => item.type === 'cover')?.data?.title,
      pptjson: JSON.stringify(pptjson.value),
      outline: JSON.stringify(outline.value),
      thumbnail: thumbnail,
    })

    pptjson.value.map(async (item: any, index: number) => {
      await PPTApi.save({
        title: item.data.title,
        pageNumber: index,
        chapterId: chid,
        imagePath: urls.value[index],
        contentText: item.data.items == null ? item.data.title : JSON.stringify(item.data.items),
        thumbnail: urls.value[index],
      })
    })
    loadItems()
  } catch {
    textOutput.value = '解码失败'
  }
  console.log(textOutput.value)
}

const handleCreateOutline = (data: any) => {
  console.log(data)
  outline.value = data
}

const handleCreatePPT = (data: any) => {
  console.log(data)
  pptjson.value = data
}

const base64ToFile = (
  base64: string,
  filename: string = "image.jpg"
): File => {
  // 分割 MIME 类型和实际数据
  const [header, data] = base64.split(",");

  // 提取 MIME 类型
  const mimeMatch = header.match(/:(.*?);/);
  const mime = mimeMatch ? mimeMatch[1] : "image/jpeg";

  // 解码 Base64
  const byteString = atob(data);
  const buffer = new ArrayBuffer(byteString.length);
  const uintArray = new Uint8Array(buffer);

  for (let i = 0; i < byteString.length; i++) {
    uintArray[i] = byteString.charCodeAt(i);
  }

  // 生成 File 对象
  return new File([buffer], filename, { type: mime });
};

const save = async (id: string) => {
  chapterId.value = id
  console.log(chapterId.value)
}

const saveChapter = () => {
  // 实现保存逻辑
  dialog.value = false
  console.log('保存按钮被点击');
}

const loadItems = async () => {
  loading.value = true
  try {
    /* const res = await courseApi.detail(props.courseId) */
    const res = await ChapterApi.list(props.courseId)
    chapters.value = res
    console.log(chapters.value)
  } catch (error) {
    console.error('加载失败:', error)
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadItems()
})
</script>

<style scoped>
.v-list-item {
  margin-bottom: 10px;
}
</style>
