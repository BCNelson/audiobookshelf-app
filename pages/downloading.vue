<template>
  <div class="w-full h-full py-6 px-4 overflow-y-auto">
    <div class="flex items-center mb-4">
      <p class="text-base text-fg">{{ $strings.HeaderDownloads }} ({{ downloadItemParts.length }})</p>
      <div class="flex-grow" />
      <ui-btn v-if="isAndroid && downloadItems.length > 1" small color="error" @click="cancelAll">{{ $strings.ButtonCancel }} ({{ downloadItems.length }})</ui-btn>
    </div>
    <div v-if="!downloadItemParts.length" class="py-6 text-center text-lg">No download item parts</div>
    <template v-for="(downloadItem, itemIndex) in downloadItems">
      <div :key="downloadItem.id" class="w-full">
        <div class="flex items-center mb-2">
          <p class="flex-grow font-semibold text-fg truncate pr-2">{{ downloadItem.itemTitle }}</p>
          <template v-if="isAndroid">
            <ui-btn v-if="hasStatus(downloadItem, 'waitingForWifi')" small class="mr-2" @click="allowMobileData(downloadItem)">{{ $strings.ButtonAllowMobileData }}</ui-btn>
            <ui-btn v-if="hasStatus(downloadItem, 'failed')" small class="mr-2" @click="retry(downloadItem)">{{ $strings.ButtonRetry }}</ui-btn>
            <ui-btn small color="error" :disabled="!!cancelling[downloadItem.id]" @click="cancel(downloadItem)">{{ $strings.ButtonCancel }}</ui-btn>
          </template>
        </div>
        <template v-for="itemPart in downloadItem.downloadItemParts">
          <div :key="itemPart.id" class="flex py-1">
            <div class="w-14">
              <span v-if="itemPart.completed" class="material-symbols text-success">check_circle</span>
              <span v-else-if="itemPart.status === 'failed'" class="material-symbols text-error">error</span>
              <span v-else class="font-semibold text-fg">{{ Math.round(itemPart.progress) }}%</span>
            </div>
            <div class="flex-grow px-2 min-w-0">
              <p class="break-all">{{ itemPart.filename }}</p>
              <p v-if="statusLabel(itemPart)" class="text-xs text-fg-muted">{{ statusLabel(itemPart) }}</p>
            </div>
          </div>
        </template>
        <div v-if="itemIndex + 1 < downloadItems.length" class="flex border-t border-border my-3" />
      </div>
    </template>
  </div>
</template>

<script>
import { AbsDownloader } from '@/plugins/capacitor'
import cellularPermissionHelpers from '@/mixins/cellularPermissionHelpers'

export default {
  mixins: [cellularPermissionHelpers],
  data() {
    return {
      cancelling: {}
    }
  },
  computed: {
    isAndroid() {
      return this.$platform === 'android'
    },
    downloadItems() {
      return this.$store.state.globals.itemDownloads
    },
    downloadItemParts() {
      let parts = []
      this.downloadItems.forEach((di) => parts.push(...di.downloadItemParts))
      return parts
    }
  },
  methods: {
    hasStatus(downloadItem, status) {
      return downloadItem.downloadItemParts.some((part) => part.status === status)
    },
    statusLabel(itemPart) {
      switch (itemPart.status) {
        case 'queued':
          return this.$strings.LabelDownloadStatusQueued
        case 'retrying':
          return this.$strings.LabelDownloadStatusRetrying
        case 'waitingForNetwork':
          return this.$strings.MessageWaitingForNetwork
        case 'waitingForWifi':
          return this.$strings.MessageWaitingForWifi
        case 'waitingForStorage':
          return this.$strings.MessageWaitingForAvailableStorage
        case 'finalizing':
          return this.$strings.LabelDownloadStatusFinalizing
        case 'failed':
          return this.$strings.LabelDownloadStatusFailed
        default:
          return null
      }
    },
    async cancel(downloadItem) {
      this.$set(this.cancelling, downloadItem.id, true)
      try {
        const { cancelled } = await AbsDownloader.cancelDownloadItem({ downloadItemId: downloadItem.id })
        if (cancelled) this.$store.commit('globals/removeItemDownload', downloadItem.id)
      } catch (error) {
        console.error('Failed to cancel download', error)
      } finally {
        this.$delete(this.cancelling, downloadItem.id)
      }
    },
    async cancelAll() {
      await AbsDownloader.cancelAllDownloads()
      this.$store.commit('globals/clearItemDownloads')
    },
    allowMobileData(downloadItem) {
      AbsDownloader.allowCellularDownload({ downloadItemId: downloadItem.id })
    },
    async retry(downloadItem) {
      if (!(await this.checkCellularPermission('download'))) return
      const payload = {
        networkMode: this.downloadNetworkMode,
        libraryItemId: downloadItem.libraryItemId,
        episodeId: downloadItem.episodeId
      }
      const res = await AbsDownloader.downloadLibraryItem(payload)
      if (res && res.error) this.$toast.error(res.error)
    }
  }
}
</script>
