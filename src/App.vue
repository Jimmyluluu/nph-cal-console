<script setup lang="ts">
import JSZip from 'jszip'
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
  FolderOpenIcon,
  Layers3Icon,
  UploadCloudIcon,
  XIcon,
} from '@lucide/vue'

type ImagingKind = 'NIfTI' | 'ZIP'
type UploadStatus = 'queued' | 'signing' | 'uploading' | 'uploaded' | 'failed'
type AnalysisStatus = 'idle' | 'segmenting' | 'calculating' | 'completed' | 'failed'

interface PresignedUploadResponse {
  data?: {
    uploadUrl?: string
    key?: string
  }
}

interface ImagingFile {
  id: string
  file: File
  kind: ImagingKind
  contentType: string
  status: UploadStatus
  progress: number
  sourceFileCount?: number
  originalSize?: number
  key?: string
  error?: string
}

const uploadApiBaseUrl = 'https://nph-s3-api.lujimmy.com'
const analysisApiBaseUrl = 'https://nph-api.lujimmy.com'
const acceptedExtensions = '.nii,.nii.gz,.zip'
const files = ref<ImagingFile[]>([])
const isDragging = ref(false)
const isUploading = ref(false)
const isCompressing = ref(false)
const compressionProgress = ref(0)
const errorMessage = ref('')
const analysisStatus = ref<AnalysisStatus>('idle')
const analysisError = ref('')
const segmentationResult = ref<Record<string, unknown> | null>(null)
const indicatorResult = ref<Record<string, unknown> | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)
const folderInput = ref<HTMLInputElement | null>(null)

const totalSize = computed(() => files.value.reduce((size, item) => size + item.file.size, 0))
const niftiCount = computed(() => files.value.filter((item) => item.kind === 'NIfTI').length)
const zipCount = computed(() => files.value.filter((item) => item.kind === 'ZIP').length)
const hasFiles = computed(() => files.value.length > 0)
const selectedCase = computed(() => files.value[0])
const uploadedCount = computed(() => files.value.filter((item) => item.status === 'uploaded').length)
const uploadableFiles = computed(() =>
  files.value.filter((item) => item.status !== 'uploaded' && item.status !== 'signing' && item.status !== 'uploading'),
)
const isAnalyzing = computed(() => analysisStatus.value === 'segmenting' || analysisStatus.value === 'calculating')
const canStartAnalysis = computed(() => {
  const item = selectedCase.value

  return Boolean(item?.key && item.status === 'uploaded' && !isUploading.value && !isAnalyzing.value)
})
const segmentationEntries = computed(() => Object.entries(segmentationResult.value ?? {}))
const indicatorEntries = computed(() => Object.entries(indicatorResult.value ?? {}))
const uploadProgress = computed(() => {
  if (!hasFiles.value) {
    return 0
  }

  const totalProgress = files.value.reduce((progress, item) => progress + item.progress, 0)

  return Math.round(totalProgress / files.value.length)
})

const detectImagingKind = (fileName: string): ImagingKind | null => {
  const normalizedName = fileName.toLowerCase()

  if (normalizedName.endsWith('.nii') || normalizedName.endsWith('.nii.gz')) {
    return 'NIfTI'
  }

  if (normalizedName.endsWith('.zip')) {
    return 'ZIP'
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

const getContentType = (file: File, kind: ImagingKind) => {
  if (file.type) {
    return file.type
  }

  if (kind === 'ZIP') {
    return 'application/zip'
  }

  return 'application/octet-stream'
}

const resetAnalysis = () => {
  analysisStatus.value = 'idle'
  analysisError.value = ''
  segmentationResult.value = null
  indicatorResult.value = null
}

const formatResultLabel = (key: string) => {
  return key
    .replace(/_/g, ' ')
    .replace(/-/g, ' ')
    .replace(/\b\w/g, (character) => character.toUpperCase())
}

const formatResultValue = (value: unknown): string => {
  if (typeof value === 'number') {
    return Number.isInteger(value) ? String(value) : value.toFixed(4)
  }

  if (typeof value === 'string') {
    return value
  }

  if (typeof value === 'boolean') {
    return value ? 'true' : 'false'
  }

  if (value === null || value === undefined) {
    return '-'
  }

  return JSON.stringify(value)
}

const getRecordValue = (record: Record<string, unknown>, keys: string[]) => {
  for (const key of keys) {
    const value = record[key]

    if (typeof value === 'string' && value.trim()) {
      return value
    }
  }

  return ''
}

const getCaseDirFromSegmentation = (payload: Record<string, unknown>) => {
  const directValue = getRecordValue(payload, ['case_dir', 'output_dir', 'result_dir', 'caseDir', 'outputDir', 'resultDir'])

  if (directValue) {
    return directValue
  }

  const nestedKeys = ['data', 'result', 'results']

  for (const key of nestedKeys) {
    const nestedValue = payload[key]

    if (nestedValue && typeof nestedValue === 'object' && !Array.isArray(nestedValue)) {
      const caseDir = getRecordValue(nestedValue as Record<string, unknown>, [
        'case_dir',
        'output_dir',
        'result_dir',
        'caseDir',
        'outputDir',
        'resultDir',
      ])

      if (caseDir) {
        return caseDir
      }
    }
  }

  return ''
}

const postJson = async (path: string, body: Record<string, unknown>) => {
  const response = await fetch(`${analysisApiBaseUrl}${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  })

  if (!response.ok) {
    throw new Error(`分析 API 失敗：HTTP ${response.status}`)
  }

  return (await response.json()) as Record<string, unknown>
}

const sanitizeZipName = (folderName: string) => {
  const normalizedName = folderName.trim().replace(/[/\\]+/g, '-').replace(/\.zip$/i, '')

  return `${normalizedName || 'imaging-folder'}.zip`
}

const getFolderName = (folderFiles: File[]) => {
  const relativePath = folderFiles[0]?.webkitRelativePath

  return relativePath?.split('/')[0] || 'imaging-folder'
}

const getStatusLabel = (status: UploadStatus) => {
  const labels: Record<UploadStatus, string> = {
    queued: '待上傳',
    signing: '取得 URL',
    uploading: '上傳中',
    uploaded: '已上傳',
    failed: '失敗',
  }

  return labels[status]
}

const getStatusVariant = (status: UploadStatus) => {
  if (status === 'uploaded') {
    return 'secondary'
  }

  if (status === 'failed') {
    return 'destructive'
  }

  return 'outline'
}

const getAnalysisStatusLabel = (status: AnalysisStatus) => {
  const labels: Record<AnalysisStatus, string> = {
    idle: '等待分析',
    segmenting: '分割中',
    calculating: '計算指標中',
    completed: '分析完成',
    failed: '分析失敗',
  }

  return labels[status]
}

const getAnalysisHint = () => {
  if (!hasFiles.value) {
    return '請先加入一個 ZIP 或 NIfTI 案例。'
  }

  if (selectedCase.value?.status !== 'uploaded') {
    return '案例上傳到 R2 後即可開始分析。'
  }

  if (analysisStatus.value === 'segmenting') {
    return '正在由後端讀取 R2 檔案並產生分割結果。'
  }

  if (analysisStatus.value === 'calculating') {
    return '分割完成，正在計算 NPH 指標。'
  }

  if (analysisStatus.value === 'completed') {
    return '已完成分割與全部指標計算。'
  }

  if (analysisStatus.value === 'failed') {
    return analysisError.value || '分析失敗，可以確認 API 狀態後重試。'
  }

  return '已上傳完成，可以開始單案分析。'
}

const requestPresignedUpload = async (file: File, contentType: string) => {
  const response = await fetch(`${uploadApiBaseUrl}/upload/presigned`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      filename: file.name,
      contentType,
    }),
  })

  if (!response.ok) {
    throw new Error(`產生上傳 URL 失敗：HTTP ${response.status}`)
  }

  const payload = (await response.json()) as PresignedUploadResponse
  const uploadUrl = payload.data?.uploadUrl
  const key = payload.data?.key

  if (!uploadUrl || !key) {
    throw new Error('產生上傳 URL 失敗：回應缺少 uploadUrl 或 key')
  }

  return { uploadUrl, key }
}

const uploadToPresignedUrl = (item: ImagingFile, uploadUrl: string) => {
  return new Promise<void>((resolve, reject) => {
    const request = new XMLHttpRequest()

    request.open('PUT', uploadUrl)
    request.setRequestHeader('Content-Type', item.contentType)

    request.upload.onprogress = (event) => {
      if (event.lengthComputable) {
        item.progress = Math.max(1, Math.round((event.loaded / event.total) * 100))
      }
    }

    request.onload = () => {
      if (request.status >= 200 && request.status < 300) {
        item.progress = 100
        resolve()
        return
      }

      reject(new Error(`上傳失敗：HTTP ${request.status}`))
    }

    request.onerror = () => reject(new Error('上傳失敗：網路或 CORS 錯誤'))
    request.send(item.file)
  })
}

const uploadFile = async (item: ImagingFile) => {
  item.status = 'signing'
  item.error = ''
  item.progress = 0

  try {
    const { uploadUrl, key } = await requestPresignedUpload(item.file, item.contentType)

    item.key = key
    item.status = 'uploading'
    await uploadToPresignedUrl(item, uploadUrl)
    item.status = 'uploaded'
  } catch (error) {
    item.status = 'failed'
    item.error = error instanceof Error ? error.message : '上傳失敗'
  }
}

const uploadAllFiles = async () => {
  if (!uploadableFiles.value.length || isUploading.value) {
    return
  }

  resetAnalysis()
  isUploading.value = true
  errorMessage.value = ''

  for (const item of uploadableFiles.value) {
    await uploadFile(item)
  }

  isUploading.value = false
}

const addFiles = (candidateFiles: File[]) => {
  if (files.value.length) {
    errorMessage.value = '一次只能分析一個案例，請先清除目前案例。'
    return
  }

  const validFiles: ImagingFile[] = []
  const invalidNames: string[] = []
  let skippedValidCount = 0

  for (const file of candidateFiles) {
    const kind = detectImagingKind(file.name)

    if (!kind) {
      invalidNames.push(file.name)
      continue
    }

    const id = `${file.name}-${file.size}-${file.lastModified}`
    const alreadyQueued = validFiles.some((item) => item.id === id)

    if (validFiles.length || alreadyQueued) {
      skippedValidCount += 1
      continue
    }

    if (!alreadyQueued) {
      validFiles.push({
        id,
        file,
        kind,
        contentType: getContentType(file, kind),
        status: 'queued',
        progress: 0,
      })
    }
  }

  if (validFiles.length) {
    resetAnalysis()
    files.value = validFiles
  }

  const messages: string[] = []

  if (invalidNames.length) {
    messages.push(`已略過不支援的檔案：${invalidNames.join(', ')}`)
  }

  if (skippedValidCount) {
    messages.push('一次只能分析一個案例，已只加入第一個 ZIP 或 NIfTI。')
  }

  errorMessage.value = messages.join('；')
}

const openFilePicker = () => {
  fileInput.value?.click()
}

const openFolderPicker = () => {
  folderInput.value?.click()
}

const handleFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  addFiles(Array.from(input.files ?? []))
  input.value = ''
}

const handleFolderChange = async (event: Event) => {
  const input = event.target as HTMLInputElement
  const folderFiles = Array.from(input.files ?? [])

  input.value = ''

  if (files.value.length) {
    errorMessage.value = '一次只能分析一個案例，請先清除目前案例。'
    return
  }

  if (!folderFiles.length || isCompressing.value || isUploading.value) {
    return
  }

  isCompressing.value = true
  compressionProgress.value = 0
  errorMessage.value = ''

  try {
    const folderName = getFolderName(folderFiles)
    const zipName = sanitizeZipName(folderName)
    const zip = new JSZip()
    const originalSize = folderFiles.reduce((size, file) => size + file.size, 0)

    for (const file of folderFiles) {
      const relativePath = file.webkitRelativePath || file.name
      zip.file(relativePath, file)
    }

    const zipBlob = await zip.generateAsync(
      {
        type: 'blob',
        compression: 'DEFLATE',
        compressionOptions: {
          level: 6,
        },
      },
      (metadata) => {
        compressionProgress.value = Math.round(metadata.percent)
      },
    )
    const zipFile = new File([zipBlob], zipName, {
      type: 'application/zip',
      lastModified: Date.now(),
    })
    const id = `${zipFile.name}-${zipFile.size}-${zipFile.lastModified}`

    if (files.value.length) {
      errorMessage.value = '一次只能分析一個案例，請先清除目前案例。'
      return
    }

    resetAnalysis()
    files.value = [
      {
        id,
        file: zipFile,
        kind: 'ZIP',
        contentType: 'application/zip',
        status: 'queued',
        progress: 0,
        sourceFileCount: folderFiles.length,
        originalSize,
      },
    ]
  } catch (error) {
    errorMessage.value = error instanceof Error ? `壓縮資料夾失敗：${error.message}` : '壓縮資料夾失敗'
  } finally {
    isCompressing.value = false
    compressionProgress.value = 0
  }
}

const handleDrop = (event: DragEvent) => {
  isDragging.value = false
  addFiles(Array.from(event.dataTransfer?.files ?? []))
}

const removeFile = (id: string) => {
  if (files.value.some((item) => item.id === id && ['signing', 'uploading'].includes(item.status))) {
    return
  }

  files.value = files.value.filter((item) => item.id !== id)
  resetAnalysis()
}

const clearFiles = () => {
  if (isUploading.value || isAnalyzing.value) {
    return
  }

  files.value = []
  errorMessage.value = ''
  resetAnalysis()
}

const startAnalysis = async () => {
  const item = selectedCase.value

  if (!item?.key || item.status !== 'uploaded' || isAnalyzing.value) {
    return
  }

  resetAnalysis()
  analysisStatus.value = 'segmenting'

  try {
    const segmentationPayload = await postJson('/segmentation/patient', {
      r2_key: item.key,
      output_dir: 'results',
      skip_existing: false,
    })

    segmentationResult.value = segmentationPayload
    const caseDir = getCaseDirFromSegmentation(segmentationPayload)

    if (!caseDir) {
      throw new Error('分割完成，但回應缺少 case_dir / output_dir / result_dir，無法計算指標。')
    }

    analysisStatus.value = 'calculating'
    indicatorResult.value = await postJson('/indicators/calculate-all', {
      case_dir: caseDir,
    })
    analysisStatus.value = 'completed'
  } catch (error) {
    analysisStatus.value = 'failed'
    analysisError.value = error instanceof Error ? error.message : '分析失敗'
  }
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
              上傳並分析單一 3D 腦部案例
            </h1>
            <p class="text-base leading-7 text-slate-600 sm:text-lg">
              一次只處理一個案例。支援整個 DICOM folder 壓成 <span class="font-medium text-slate-900">.zip</span>，或直接上傳 <span class="font-medium text-slate-900">.zip/.nii/.nii.gz</span>。
            </p>
          </div>
        </div>
        <div class="grid grid-cols-3 gap-2 rounded-2xl border bg-white/80 p-2 text-center shadow-sm">
          <div class="rounded-xl bg-slate-950 px-4 py-3 text-white">
            <p class="text-2xl font-semibold">{{ files.length }}</p>
            <p class="text-xs text-slate-300">案例</p>
          </div>
          <div class="rounded-xl bg-slate-100 px-4 py-3">
            <p class="text-2xl font-semibold text-slate-950">{{ niftiCount }}</p>
            <p class="text-xs text-slate-500">NIfTI</p>
          </div>
          <div class="rounded-xl bg-slate-100 px-4 py-3">
            <p class="text-2xl font-semibold text-slate-950">{{ zipCount }}</p>
            <p class="text-xs text-slate-500">ZIP</p>
          </div>
        </div>
      </div>

      <div class="grid gap-6 lg:grid-cols-[minmax(0,1.05fr)_minmax(360px,0.95fr)]">
        <Card class="overflow-hidden border-white/70 bg-white/85 shadow-sm backdrop-blur">
          <CardHeader>
            <CardTitle class="flex items-center gap-2 text-2xl">
              <UploadCloudIcon class="size-6 text-sky-600" />
              單案例上傳
            </CardTitle>
            <CardDescription>
              可以選擇一個 NIfTI/ZIP 檔，也可以選擇整個 DICOM folder 並壓縮成單一 ZIP。
            </CardDescription>
          </CardHeader>
          <CardContent class="space-y-5">
            <div
              class="rounded-3xl border border-dashed p-4 transition-all sm:p-5"
              :class="isDragging ? 'border-sky-500 bg-sky-50 shadow-inner' : 'border-slate-300 bg-slate-50/80'"
              @dragenter.prevent="isDragging = true"
              @dragover.prevent="isDragging = true"
              @dragleave.prevent="isDragging = false"
              @drop.prevent="handleDrop"
            >
              <div class="mb-5 flex flex-col gap-2 text-center">
                <div class="mx-auto grid size-16 place-items-center rounded-3xl bg-slate-950 text-white shadow-lg shadow-slate-950/20">
                  <UploadCloudIcon class="size-8" />
                </div>
                <h2 class="text-2xl font-semibold text-slate-950">選擇你的 3D 影像來源</h2>
                <p class="text-sm leading-6 text-slate-500">
                  系統會強制單一案例：資料夾先壓縮成 ZIP；已壓縮 ZIP 或 NIfTI 直接加入上傳清單。
                </p>
              </div>

              <div class="grid gap-3 md:grid-cols-2">
                <button
                  type="button"
                  class="group rounded-2xl border bg-white p-5 text-left shadow-sm transition hover:-translate-y-0.5 hover:border-sky-300 hover:shadow-md disabled:pointer-events-none disabled:opacity-60"
                  :disabled="isCompressing || isUploading || isAnalyzing"
                  @click="openFolderPicker"
                >
                  <span class="mb-4 grid size-12 place-items-center rounded-2xl bg-sky-100 text-sky-700 transition group-hover:bg-sky-600 group-hover:text-white">
                    <FolderOpenIcon class="size-6" />
                  </span>
                  <span class="block text-lg font-semibold text-slate-950">我有一整個 folder</span>
                  <span class="mt-2 block text-sm leading-6 text-slate-600">
                    適合 DICOM series 或多檔組成的 3D 影像。會保留相對路徑，先壓縮成 .zip 再上傳。
                  </span>
                  <span class="mt-4 inline-flex text-sm font-medium text-sky-700">
                    {{ isCompressing ? `壓縮中 ${compressionProgress}%` : '選擇資料夾，自動壓縮' }}
                  </span>
                </button>

                <button
                  type="button"
                  class="group rounded-2xl border bg-white p-5 text-left shadow-sm transition hover:-translate-y-0.5 hover:border-slate-400 hover:shadow-md disabled:pointer-events-none disabled:opacity-60"
                  :disabled="isCompressing || isUploading || isAnalyzing"
                  @click="openFilePicker"
                >
                  <span class="mb-4 grid size-12 place-items-center rounded-2xl bg-slate-100 text-slate-700 transition group-hover:bg-slate-950 group-hover:text-white">
                    <FileScanIcon class="size-6" />
                  </span>
                  <span class="block text-lg font-semibold text-slate-950">我已經有檔案或壓縮檔</span>
                  <span class="mt-2 block text-sm leading-6 text-slate-600">
                    支援 .zip、.nii、.nii.gz。已壓縮的 ZIP 不會再壓縮，會直接上傳到 R2。
                  </span>
                  <span class="mt-4 inline-flex text-sm font-medium text-slate-900">選擇 ZIP / NIfTI</span>
                </button>
              </div>

              <div v-if="isCompressing" class="mt-5 space-y-2 rounded-2xl border border-sky-100 bg-white p-4">
                <div class="h-2 overflow-hidden rounded-full bg-slate-100">
                  <div class="h-full rounded-full bg-sky-500 transition-all" :style="{ width: `${compressionProgress}%` }" />
                </div>
                <p class="text-xs text-slate-500">正在建立 ZIP，完成後會自動加入待上傳清單。</p>
              </div>

              <p class="mt-4 text-center text-xs text-slate-500">
                也可以把已壓縮 ZIP 或 NIfTI 直接拖拉到這個區塊。
              </p>
            </div>

            <input
              ref="fileInput"
              class="sr-only"
              type="file"
              :accept="acceptedExtensions"
              @change="handleFileChange"
            >

            <input
              ref="folderInput"
              class="sr-only"
              type="file"
              webkitdirectory
              directory
              multiple
              @change="handleFolderChange"
            >

            <p v-if="errorMessage" class="rounded-xl border border-amber-200 bg-amber-50 px-4 py-3 text-sm text-amber-800">
              {{ errorMessage }}
            </p>

            <div class="space-y-4 rounded-2xl border bg-white px-4 py-4">
              <div class="flex flex-wrap items-center justify-between gap-3">
                <div class="flex flex-wrap items-center gap-2 text-sm text-slate-600">
                  <Badge variant="secondary">總大小 {{ formatFileSize(totalSize) }}</Badge>
                  <Badge variant="outline">已上傳 {{ uploadedCount }}/{{ files.length }}</Badge>
                  <Badge variant="outline">單一案例</Badge>
                </div>
                <div class="flex flex-wrap items-center gap-2">
                  <Button
                    v-if="hasFiles"
                    type="button"
                    :disabled="!uploadableFiles.length || isUploading || isCompressing || isAnalyzing"
                    @click="uploadAllFiles"
                  >
                    {{ isUploading ? '上傳中...' : '上傳到 R2' }}
                  </Button>
                  <Button v-if="hasFiles" type="button" variant="outline" :disabled="isUploading || isAnalyzing" @click="clearFiles">
                    清除全部
                  </Button>
                </div>
              </div>
              <div v-if="hasFiles" class="space-y-2">
                <div class="h-2 overflow-hidden rounded-full bg-slate-100">
                  <div class="h-full rounded-full bg-sky-500 transition-all" :style="{ width: `${uploadProgress}%` }" />
                </div>
                <p class="text-xs text-slate-500">整體上傳進度 {{ uploadProgress }}%</p>
              </div>
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
                <div class="min-w-0 flex-1 space-y-1">
                  <p class="truncate text-sm font-medium text-slate-950">{{ item.file.name }}</p>
                  <p class="text-xs text-slate-500">
                    {{ item.kind }} · {{ formatFileSize(item.file.size) }} · {{ item.contentType }}
                  </p>
                  <p v-if="item.sourceFileCount" class="text-xs text-slate-500">
                    來源 {{ item.sourceFileCount }} 個檔案 · 原始大小 {{ formatFileSize(item.originalSize ?? 0) }}
                  </p>
                  <div class="h-1.5 overflow-hidden rounded-full bg-slate-100">
                    <div class="h-full rounded-full bg-sky-500 transition-all" :style="{ width: `${item.progress}%` }" />
                  </div>
                  <p v-if="item.key" class="truncate text-xs text-emerald-700">R2 key: {{ item.key }}</p>
                  <p v-if="item.error" class="text-xs text-red-600">{{ item.error }}</p>
                </div>
                <Badge variant="secondary">
                  {{ item.kind }}
                </Badge>
                <Badge :variant="getStatusVariant(item.status)">
                  {{ getStatusLabel(item.status) }}
                </Badge>
                <Button
                  type="button"
                  variant="ghost"
                  size="icon"
                  aria-label="移除檔案"
                  :disabled="item.status === 'signing' || item.status === 'uploading'"
                  @click="removeFile(item.id)"
                >
                  <XIcon class="size-4" />
                </Button>
              </div>
            </div>

            <div v-else class="rounded-2xl border bg-white/70 p-5 text-sm leading-6 text-slate-500">
              尚未選取案例。請加入一個 DICOM folder、ZIP、NIfTI volume。
            </div>
          </CardContent>
        </Card>

        <Card class="border-slate-900 bg-slate-950 text-white shadow-xl shadow-slate-950/20">
          <CardHeader>
            <CardTitle class="flex items-center gap-2 text-2xl text-white">
              <BrainIcon class="size-6 text-sky-300" />
              單案分析結果
            </CardTitle>
            <CardDescription class="text-slate-400">
              上傳完成後會呼叫分割 API，再計算全部 NPH 指標。
            </CardDescription>
          </CardHeader>
          <CardContent>
            <div class="relative min-h-[520px] overflow-hidden rounded-3xl border border-white/10 bg-[radial-gradient(circle_at_center,_rgba(56,189,248,0.22),_transparent_36%),linear-gradient(160deg,_#0f172a,_#020617)] p-6">
              <div class="absolute inset-x-10 top-12 h-px bg-gradient-to-r from-transparent via-sky-300/40 to-transparent" />
              <div class="absolute inset-y-10 left-12 w-px bg-gradient-to-b from-transparent via-sky-300/30 to-transparent" />
              <div class="absolute bottom-10 right-8 h-28 w-28 rounded-full border border-sky-300/20" />
              <div class="absolute bottom-20 right-20 h-52 w-52 rounded-full border border-sky-300/10" />

              <div class="relative z-10 flex flex-col gap-5">
                <div class="flex flex-col gap-4 rounded-[2rem] border border-sky-200/20 bg-white/10 p-5 shadow-2xl shadow-sky-500/20 backdrop-blur sm:flex-row sm:items-center sm:justify-between">
                  <div class="flex items-center gap-4">
                    <div class="grid size-16 shrink-0 place-items-center rounded-2xl bg-sky-300/15 text-sky-200">
                      <Layers3Icon class="size-8" />
                    </div>
                    <div class="space-y-1">
                      <p class="text-sm text-slate-400">分析狀態</p>
                      <h2 class="text-2xl font-semibold tracking-tight text-white">
                        {{ getAnalysisStatusLabel(analysisStatus) }}
                      </h2>
                      <p class="text-sm leading-6 text-slate-300">
                        {{ getAnalysisHint() }}
                      </p>
                    </div>
                  </div>
                  <Button
                    type="button"
                    class="bg-sky-300 text-slate-950 hover:bg-sky-200"
                    :disabled="!canStartAnalysis"
                    @click="startAnalysis"
                  >
                    {{ isAnalyzing ? '分析中...' : analysisStatus === 'completed' ? '重新分析' : '開始分析' }}
                  </Button>
                </div>

                <div class="grid gap-3 text-left text-sm sm:grid-cols-3">
                  <div class="rounded-2xl border border-white/10 bg-white/5 p-4">
                    <p class="text-slate-400">案例</p>
                    <p class="truncate font-medium text-white">{{ selectedCase?.file.name || '尚未選取' }}</p>
                  </div>
                  <div class="rounded-2xl border border-white/10 bg-white/5 p-4">
                    <p class="text-slate-400">R2</p>
                    <p class="font-medium text-white">{{ uploadedCount }} 已上傳</p>
                  </div>
                  <div class="rounded-2xl border border-white/10 bg-white/5 p-4">
                    <p class="text-slate-400">API</p>
                    <p class="font-medium text-white">{{ isAnalyzing ? '執行中' : '待命' }}</p>
                  </div>
                </div>

                <div v-if="isAnalyzing" class="space-y-3 rounded-2xl border border-sky-300/20 bg-sky-300/10 p-4">
                  <div class="flex items-center justify-between text-sm">
                    <span class="font-medium text-sky-100">{{ getAnalysisStatusLabel(analysisStatus) }}</span>
                    <span class="text-sky-200">請稍候</span>
                  </div>
                  <div class="h-2 overflow-hidden rounded-full bg-white/10">
                    <div
                      class="h-full rounded-full bg-sky-300 transition-all"
                      :style="{ width: analysisStatus === 'segmenting' ? '45%' : '78%' }"
                    />
                  </div>
                </div>

                <p v-if="analysisError" class="rounded-2xl border border-red-300/30 bg-red-500/10 px-4 py-3 text-sm leading-6 text-red-100">
                  {{ analysisError }}
                </p>

                <div v-if="indicatorEntries.length" class="space-y-3">
                  <div class="flex items-center justify-between gap-3">
                    <h3 class="text-lg font-semibold text-white">NPH 指標</h3>
                    <Badge variant="outline" class="border-sky-200/30 bg-white/10 text-sky-100">
                      {{ indicatorEntries.length }} 筆結果
                    </Badge>
                  </div>
                  <div class="grid gap-3 sm:grid-cols-2">
                    <div
                      v-for="[key, value] in indicatorEntries"
                      :key="key"
                      class="rounded-2xl border border-white/10 bg-white/10 p-4"
                    >
                      <p class="text-xs uppercase tracking-wide text-slate-400">{{ formatResultLabel(key) }}</p>
                      <p class="mt-2 break-words text-lg font-semibold text-white">{{ formatResultValue(value) }}</p>
                    </div>
                  </div>
                </div>

                <details v-if="segmentationEntries.length" class="rounded-2xl border border-white/10 bg-white/5 p-4 text-sm text-slate-300">
                  <summary class="cursor-pointer font-medium text-white">分割 API 回應</summary>
                  <div class="mt-3 space-y-2">
                    <div v-for="[key, value] in segmentationEntries" :key="key" class="grid gap-1">
                      <span class="text-slate-500">{{ formatResultLabel(key) }}</span>
                      <span class="break-words text-slate-200">{{ formatResultValue(value) }}</span>
                    </div>
                  </div>
                </details>
              </div>
            </div>
          </CardContent>
        </Card>
      </div>
    </section>
  </main>
</template>
