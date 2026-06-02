<script setup lang="ts">
import { computed, ref } from 'vue'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from '@/components/ui/card'
import {
  BrainIcon,
  FileScanIcon,
  Layers3Icon,
  UploadCloudIcon,
  XIcon,
} from '@lucide/vue'

type ImagingKind = 'DICOM' | 'NIfTI'

interface ImagingFile {
  id: string
  file: File
  kind: ImagingKind
}

const acceptedExtensions = '.dcm,.nii,.nii.gz'
const files = ref<ImagingFile[]>([])
const isDragging = ref(false)
const errorMessage = ref('')
const fileInput = ref<HTMLInputElement | null>(null)

const totalSize = computed(() => files.value.reduce((size, item) => size + item.file.size, 0))
const dicomCount = computed(() => files.value.filter((item) => item.kind === 'DICOM').length)
const niftiCount = computed(() => files.value.filter((item) => item.kind === 'NIfTI').length)
const hasFiles = computed(() => files.value.length > 0)

const detectImagingKind = (fileName: string): ImagingKind | null => {
  const normalizedName = fileName.toLowerCase()

  if (normalizedName.endsWith('.dcm')) {
    return 'DICOM'
  }

  if (normalizedName.endsWith('.nii') || normalizedName.endsWith('.nii.gz')) {
    return 'NIfTI'
  }

  return null
}

const formatFileSize = (bytes: number) => {
  if (bytes === 0) {
    return '0 B'
  }

  const units = ['B', 'KB', 'MB', 'GB']
  const unitIndex = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1)
  const size = bytes / 1024 ** unitIndex

  return `${size.toFixed(size >= 10 || unitIndex === 0 ? 0 : 1)} ${units[unitIndex]}`
}

const addFiles = (candidateFiles: File[]) => {
  const validFiles: ImagingFile[] = []
  const invalidNames: string[] = []

  for (const file of candidateFiles) {
    const kind = detectImagingKind(file.name)

    if (!kind) {
      invalidNames.push(file.name)
      continue
    }

    const id = `${file.name}-${file.size}-${file.lastModified}`
    const alreadySelected = files.value.some((item) => item.id === id)
    const alreadyQueued = validFiles.some((item) => item.id === id)

    if (!alreadySelected && !alreadyQueued) {
      validFiles.push({ id, file, kind })
    }
  }

  files.value = [...files.value, ...validFiles]
  errorMessage.value = invalidNames.length
    ? `已略過不支援的檔案：${invalidNames.join(', ')}`
    : ''
}

const openFilePicker = () => {
  fileInput.value?.click()
}

const handleFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  addFiles(Array.from(input.files ?? []))
  input.value = ''
}

const handleDrop = (event: DragEvent) => {
  isDragging.value = false
  addFiles(Array.from(event.dataTransfer?.files ?? []))
}

const removeFile = (id: string) => {
  files.value = files.value.filter((item) => item.id !== id)
}

const clearFiles = () => {
  files.value = []
  errorMessage.value = ''
}
</script>

<template>
  <main class="min-h-svh bg-[radial-gradient(circle_at_top_left,_#e0f2fe_0,_transparent_28%),linear-gradient(135deg,_#f8fafc_0%,_#eef2ff_45%,_#f5f5f5_100%)] px-4 py-6 text-foreground sm:px-6 lg:px-10">
    <section class="mx-auto flex max-w-7xl flex-col gap-6">
      <div class="flex flex-col gap-4 rounded-3xl border border-white/70 bg-white/70 p-6 shadow-sm backdrop-blur md:flex-row md:items-end md:justify-between">
        <div class="max-w-3xl space-y-3">
          <Badge variant="outline" class="bg-white/80">
            NPH Cal Console
          </Badge>
          <div class="space-y-2">
            <h1 class="text-3xl font-semibold tracking-tight text-slate-950 sm:text-4xl lg:text-5xl">
              上傳 3D 腦部影像資料
            </h1>
            <p class="text-base leading-7 text-slate-600 sm:text-lg">
              支援 DICOM <span class="font-medium text-slate-900">.dcm</span> 與 NIfTI <span class="font-medium text-slate-900">.nii/.nii.gz</span>。先建立穩定的上傳流程，後續再接 3D 腦部重建與檢視器。
            </p>
          </div>
        </div>
        <div class="grid grid-cols-3 gap-2 rounded-2xl border bg-white/80 p-2 text-center shadow-sm">
          <div class="rounded-xl bg-slate-950 px-4 py-3 text-white">
            <p class="text-2xl font-semibold">{{ files.length }}</p>
            <p class="text-xs text-slate-300">檔案</p>
          </div>
          <div class="rounded-xl bg-slate-100 px-4 py-3">
            <p class="text-2xl font-semibold text-slate-950">{{ dicomCount }}</p>
            <p class="text-xs text-slate-500">DICOM</p>
          </div>
          <div class="rounded-xl bg-slate-100 px-4 py-3">
            <p class="text-2xl font-semibold text-slate-950">{{ niftiCount }}</p>
            <p class="text-xs text-slate-500">NIfTI</p>
          </div>
        </div>
      </div>

      <div class="grid gap-6 lg:grid-cols-[minmax(0,1.05fr)_minmax(360px,0.95fr)]">
        <Card class="overflow-hidden border-white/70 bg-white/85 shadow-sm backdrop-blur">
          <CardHeader>
            <CardTitle class="flex items-center gap-2 text-2xl">
              <UploadCloudIcon class="size-6 text-sky-600" />
              影像檔案上傳
            </CardTitle>
            <CardDescription>
              可以拖拉檔案到上傳區，或點選按鈕選取本機檔案。DICOM series 可一次選多個 .dcm。
            </CardDescription>
          </CardHeader>
          <CardContent class="space-y-5">
            <div
              class="group flex min-h-80 w-full cursor-pointer flex-col items-center justify-center gap-5 rounded-3xl border border-dashed p-8 text-center transition-all"
              :class="isDragging ? 'border-sky-500 bg-sky-50 shadow-inner' : 'border-slate-300 bg-slate-50/80 hover:border-sky-400 hover:bg-white'"
              @click="openFilePicker"
              @dragenter.prevent="isDragging = true"
              @dragover.prevent="isDragging = true"
              @dragleave.prevent="isDragging = false"
              @drop.prevent="handleDrop"
            >
              <span class="grid size-20 place-items-center rounded-3xl bg-slate-950 text-white shadow-lg shadow-slate-950/20 transition-transform group-hover:-translate-y-1">
                <UploadCloudIcon class="size-9" />
              </span>
              <span class="space-y-2">
                <span class="block text-2xl font-semibold text-slate-950">
                  拖拉 DICOM 或 NIfTI 到這裡
                </span>
                <span class="block text-sm leading-6 text-slate-500">
                  支援格式：.dcm、.nii、.nii.gz。上傳後先保留在瀏覽器端，尚未送出到伺服器。
                </span>
              </span>
              <Button type="button" size="lg">
                選擇檔案
              </Button>
            </div>

            <input
              ref="fileInput"
              class="sr-only"
              type="file"
              :accept="acceptedExtensions"
              multiple
              @change="handleFileChange"
            >

            <p v-if="errorMessage" class="rounded-xl border border-amber-200 bg-amber-50 px-4 py-3 text-sm text-amber-800">
              {{ errorMessage }}
            </p>

            <div class="flex flex-wrap items-center justify-between gap-3 rounded-2xl border bg-white px-4 py-3">
              <div class="flex flex-wrap items-center gap-2 text-sm text-slate-600">
                <Badge variant="secondary">總大小 {{ formatFileSize(totalSize) }}</Badge>
                <Badge variant="outline">可多選 DICOM series</Badge>
              </div>
              <Button v-if="hasFiles" type="button" variant="outline" @click="clearFiles">
                清除全部
              </Button>
            </div>

            <div v-if="hasFiles" class="space-y-3">
              <div
                v-for="item in files"
                :key="item.id"
                class="flex items-center gap-3 rounded-2xl border bg-white p-3 shadow-sm"
              >
                <div class="grid size-11 shrink-0 place-items-center rounded-xl bg-slate-100 text-slate-700">
                  <FileScanIcon class="size-5" />
                </div>
                <div class="min-w-0 flex-1">
                  <p class="truncate text-sm font-medium text-slate-950">{{ item.file.name }}</p>
                  <p class="text-xs text-slate-500">{{ item.kind }} · {{ formatFileSize(item.file.size) }}</p>
                </div>
                <Badge :variant="item.kind === 'DICOM' ? 'outline' : 'secondary'">
                  {{ item.kind }}
                </Badge>
                <Button type="button" variant="ghost" size="icon" aria-label="移除檔案" @click="removeFile(item.id)">
                  <XIcon class="size-4" />
                </Button>
              </div>
            </div>

            <div v-else class="rounded-2xl border bg-white/70 p-5 text-sm leading-6 text-slate-500">
              尚未選取檔案。若要建立 3D 腦部模型，通常需要完整 DICOM series 或單一 NIfTI volume。
            </div>
          </CardContent>
        </Card>

        <Card class="border-slate-900 bg-slate-950 text-white shadow-xl shadow-slate-950/20">
          <CardHeader>
            <CardTitle class="flex items-center gap-2 text-2xl text-white">
              <BrainIcon class="size-6 text-sky-300" />
              3D 腦部檢視器
            </CardTitle>
            <CardDescription class="text-slate-400">
              目前先保留呈現區，後續可接 volume rendering、切片瀏覽或 DICOM/NIfTI 解析流程。
            </CardDescription>
          </CardHeader>
          <CardContent>
            <div class="relative grid min-h-[520px] place-items-center overflow-hidden rounded-3xl border border-white/10 bg-[radial-gradient(circle_at_center,_rgba(56,189,248,0.22),_transparent_36%),linear-gradient(160deg,_#0f172a,_#020617)] p-6">
              <div class="absolute inset-x-10 top-12 h-px bg-gradient-to-r from-transparent via-sky-300/40 to-transparent" />
              <div class="absolute inset-y-10 left-12 w-px bg-gradient-to-b from-transparent via-sky-300/30 to-transparent" />
              <div class="absolute bottom-10 right-8 h-28 w-28 rounded-full border border-sky-300/20" />
              <div class="absolute bottom-20 right-20 h-52 w-52 rounded-full border border-sky-300/10" />

              <div class="relative z-10 flex max-w-sm flex-col items-center gap-5 text-center">
                <div class="grid size-28 place-items-center rounded-[2rem] border border-sky-200/20 bg-white/10 shadow-2xl shadow-sky-500/20 backdrop-blur">
                  <Layers3Icon class="size-12 text-sky-200" />
                </div>
                <div class="space-y-2">
                  <h2 class="text-2xl font-semibold tracking-tight">等待載入 3D brain volume</h2>
                  <p class="text-sm leading-6 text-slate-300">
                    選取檔案後，這裡會作為未來顯示 3D 腦部、切片與量測結果的主區塊。
                  </p>
                </div>
                <div class="grid w-full grid-cols-2 gap-3 text-left text-sm">
                  <div class="rounded-2xl border border-white/10 bg-white/5 p-4">
                    <p class="text-slate-400">狀態</p>
                    <p class="font-medium text-white">{{ hasFiles ? '檔案已就緒' : '等待上傳' }}</p>
                  </div>
                  <div class="rounded-2xl border border-white/10 bg-white/5 p-4">
                    <p class="text-slate-400">Renderer</p>
                    <p class="font-medium text-white">待串接</p>
                  </div>
                </div>
              </div>
            </div>
          </CardContent>
        </Card>
      </div>
    </section>
  </main>
</template>
