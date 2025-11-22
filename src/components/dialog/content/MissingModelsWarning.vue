<template>
  <NoResultsPlaceholder
    class="pb-0"
    icon="pi pi-exclamation-circle"
    :title="t('missingModelsDialog.missingModels')"
    :message="t('missingModelsDialog.missingModelsMessage')"
  />
  <div class="mb-4 flex gap-1">
    <Checkbox v-model="doNotAskAgain" binary input-id="doNotAskAgain" />
    <label for="doNotAskAgain">{{
      t('missingModelsDialog.doNotAskAgain')
    }}</label>
  </div>
  <ListBox :options="missingModels" class="comfy-missing-models">
    <template #option="{ option }">
      <Suspense v-if="isElectron()">
        <ElectronFileDownload
          :url="option.url"
          :label="option.label"
          :error="option.error"
        />
      </Suspense>
      <FileDownload
        v-else
        :url="option.url"
        :label="option.label"
        :error="option.error"
      />
    </template>
  </ListBox>
</template>

<script setup lang="ts">
import Checkbox from 'primevue/checkbox'
import ListBox from 'primevue/listbox'
import { computed, onBeforeUnmount, ref } from 'vue'
import { useI18n } from 'vue-i18n'

import ElectronFileDownload from '@/components/common/ElectronFileDownload.vue'
import FileDownload from '@/components/common/FileDownload.vue'
import NoResultsPlaceholder from '@/components/common/NoResultsPlaceholder.vue'
import { useSettingStore } from '@/platform/settings/settingStore'
import { isElectron } from '@/utils/envUtil'
import { isValidUrl } from '@/utils/formatUtil'

// TODO: Read this from server internal API rather than hardcoding here
// as some installations may wish to use custom sources
const allowedSources = [
  'https://civitai.com/',
  'https://huggingface.co/',
  'http://localhost:' // Included for testing usage only
]
const allowedSuffixes = ['.safetensors', '.sft']
// Models that fail above conditions but are still allowed
const whiteListedUrls = new Set([
  'https://huggingface.co/stabilityai/stable-zero123/resolve/main/stable_zero123.ckpt',
  'https://huggingface.co/TencentARC/T2I-Adapter/resolve/main/models/t2iadapter_depth_sd14v1.pth?download=true',
  'https://github.com/xinntao/Real-ESRGAN/releases/download/v0.1.0/RealESRGAN_x4plus.pth'
])

interface ModelInfo {
  name: string
  directory: string
  directory_invalid?: boolean
  url: string
  originalUrl?: string
  downloading?: boolean
  completed?: boolean
  progress?: number
  error?: string
  folder_path?: string
}

const props = defineProps<{
  missingModels: ModelInfo[]
  paths: Record<string, string[]>
}>()

const { t } = useI18n()

const doNotAskAgain = ref(false)

/**
 * Gets the mirror URL for a given source URL based on settings
 * @param originalUrl The original URL to check for mirroring
 * @returns The mirrored URL if enabled and available, otherwise the original URL
 */
const getMirrorUrl = (originalUrl: string): string => {
  const settingStore = useSettingStore()
  const mirrorEnabled = settingStore.get(
    'Comfy.Workflow.ModelDownloadMirror.Enabled'
  )

  if (!mirrorEnabled) {
    return originalUrl
  }

  const mirrorEndpoint = settingStore.get(
    'Comfy.Workflow.ModelDownloadMirror.Endpoint'
  )

  if (!isValidUrl(mirrorEndpoint)) {
    return originalUrl
  }

  // Replace huggingface.co URLs with mirror endpoint
  if (originalUrl.startsWith('https://huggingface.co/')) {
    try {
      const urlObj = new URL(originalUrl)
      const mirrorObj = new URL(mirrorEndpoint)

      // Keep the original path and query parameters
      mirrorObj.pathname = urlObj.pathname
      mirrorObj.search = urlObj.search

      const mirroredUrl = mirrorObj.toString()

      return mirroredUrl
    } catch (error) {
      console.error('[Mirror] Error constructing mirror URL:', error)
      return originalUrl
    }
  }

  return originalUrl
}

const modelDownloads = ref<Record<string, ModelInfo>>({})
const missingModels = computed(() => {
  return props.missingModels.map((model) => {
    const paths = props.paths[model.directory]
    const mirroredUrl = getMirrorUrl(model.url)

    if (model.directory_invalid || !paths) {
      return {
        label: `${model.directory} / ${model.name}`,
        url: mirroredUrl,
        originalUrl: model.url,
        error: 'Invalid directory specified (does this require custom nodes?)'
      }
    }
    const downloadInfo: ModelInfo = modelDownloads.value[model.name] ?? {
      downloading: false,
      completed: false,
      progress: 0,
      error: null,
      name: model.name,
      directory: model.directory,
      url: mirroredUrl,
      originalUrl: model.url,
      folder_path: paths[0]
    }
    modelDownloads.value[model.name] = downloadInfo
    if (!whiteListedUrls.has(model.url)) {
      if (!allowedSources.some((source) => model.url.startsWith(source))) {
        return {
          label: `${model.directory} / ${model.name}`,
          url: mirroredUrl,
          originalUrl: model.url,
          error: `Download not allowed from source '${model.url}', only allowed from '${allowedSources.join("', '")}'`
        }
      }
      if (!allowedSuffixes.some((suffix) => model.name.endsWith(suffix))) {
        return {
          label: `${model.directory} / ${model.name}`,
          url: mirroredUrl,
          originalUrl: model.url,
          error: `Only allowed suffixes are: '${allowedSuffixes.join("', '")}'`
        }
      }
    }
    return {
      url: mirroredUrl,
      originalUrl: model.url,
      label: `${model.directory} / ${model.name}`,
      downloading: downloadInfo.downloading,
      completed: downloadInfo.completed,
      progress: downloadInfo.progress,
      error: downloadInfo.error,
      name: model.name,
      paths: paths,
      folderPath: downloadInfo.folder_path
    }
  })
})

onBeforeUnmount(async () => {
  if (doNotAskAgain.value) {
    await useSettingStore().set(
      'Comfy.Workflow.ShowMissingModelsWarning',
      false
    )
  }
})
</script>

<style scoped>
.comfy-missing-models {
  max-height: 300px;
  overflow-y: auto;
}
</style>
