<template>
  <div id="page-wrapper" class="bg-bg page p-8 overflow-auto relative" :class="streamLibraryItem ? 'streaming' : ''">
    <div class="flex items-center justify-center mb-6">
      <div class="w-full max-w-2xl">
        <div class="flex items-center mb-4">
          <nuxt-link :to="`/item/${libraryItem.id}`" class="hover:underline">
            <h1 class="text-lg lg:text-xl">{{ mediaMetadata.title }}</h1>
          </nuxt-link>
          <button class="w-7 h-7 flex items-center justify-center mx-4 hover:scale-110 duration-100 transform text-gray-200 hover:text-white" @click="editItem">
            <span class="material-symbols text-base">edit</span>
          </button>
        </div>
      </div>
      <div class="w-full max-w-2xl">
        <div class="flex justify-end">
          <ui-dropdown v-model="selectedTool" :items="availableTools" :disabled="processing" class="max-w-sm" @input="selectedToolUpdated" />
        </div>
      </div>
    </div>
    <div class="flex justify-center mb-2">
      <div class="w-full max-w-2xl">
        <p class="text-lg">{{ isEditTool ? $strings.HeaderEditMetadata || 'Edit Metadata' : $strings.HeaderMetadataToEmbed }}</p>
      </div>
      <div class="w-full max-w-2xl"></div>
    </div>

    <div class="flex justify-center flex-wrap lg:flex-nowrap gap-4">
      <div class="w-full max-w-2xl border border-white/10 bg-bg">
        <div class="flex py-2 px-4">
          <div class="w-28 min-w-28 text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelMetaTag }}</div>
          <div class="grow text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelValue }}</div>
        </div>
        <div class="w-full max-h-72 overflow-auto">
          <template v-for="(value, key, index) in metadataObject">
            <div :key="key" class="flex py-1 px-4 text-sm" :class="index % 2 === 0 ? 'bg-primary/25' : ''">
              <div class="w-28 min-w-28 font-semibold">{{ key }}</div>
              <div class="grow">
                {{ value }}
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="w-full max-w-2xl border border-white/10 bg-bg">
        <div class="flex py-2 px-4 bg-primary/25">
          <div class="grow text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelChapterTitle }}</div>
          <div class="w-16 min-w-16 text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelStart }}</div>
          <div class="w-16 min-w-16 text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelEnd }}</div>
        </div>
        <div class="w-full max-h-72 overflow-auto">
          <p v-if="!metadataChapters.length" class="py-5 text-center text-gray-200">{{ $strings.MessageNoChapters }}</p>
          <template v-for="(chapter, index) in metadataChapters">
            <div :key="index" class="flex py-1 px-4 text-sm" :class="index % 2 === 1 ? 'bg-primary/25' : ''">
              <div class="grow font-semibold">{{ chapter.title }}</div>
              <div class="w-16 min-w-16">
                {{ $secondsToTimestamp(chapter.start) }}
              </div>
              <div class="w-16 min-w-16">
                {{ $secondsToTimestamp(chapter.end) }}
              </div>
            </div>
            <div class="w-24">
              {{ $secondsToTimestamp(chapter.end) }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="w-full h-px bg-white/10 my-8" />

    <div class="w-full max-w-4xl mx-auto">
      <!-- queued alert -->
      <widgets-alert v-if="isMetadataEmbedQueued" type="warning" class="mb-4">
        <p class="text-lg">{{ $getString('MessageEmbedQueue', [queuedEmbedLIds.length]) }}</p>
      </widgets-alert>
      <!-- metadata embed action buttons -->
      <div v-else-if="isEmbedTool" class="w-full flex justify-end items-center mb-4">
        <ui-checkbox v-if="!isTaskFinished" v-model="shouldBackupAudioFiles" :disabled="processing" :label="$strings.LabelBackupAudioFiles" medium checkbox-bg="bg" label-class="pl-2 text-base md:text-lg" @input="toggleBackupAudioFiles" />

        <div class="grow" />

        <ui-btn v-if="!isTaskFinished" color="bg-primary" :loading="processing" :progress="progress" @click.stop="embedClick">{{ $strings.ButtonStartMetadataEmbed }}</ui-btn>
        <p v-else-if="taskFailed" class="text-error text-lg font-semibold">{{ $strings.MessageEmbedFailed }} {{ taskError }}</p>
        <p v-else class="text-success text-lg font-semibold">{{ $strings.MessageEmbedFinished }}</p>
      </div>
      <!-- metadata edit interface -->
      <div v-else-if="isEditTool" class="w-full mb-4">
        <div class="flex justify-between items-center mb-4">
          <div class="flex items-center space-x-2">
            <ui-btn color="bg-primary" @click.stop="formatJson">{{ $strings.ButtonFormatJson || 'Format JSON' }}</ui-btn>
            <ui-btn color="bg-warning" @click.stop="resetJson">{{ $strings.ButtonResetJson || 'Reset' }}</ui-btn>
          </div>
          <ui-btn color="bg-success" :loading="savingMetadata" @click.stop="saveMetadata">{{ $strings.ButtonSaveMetadata || 'Save Metadata' }}</ui-btn>
        </div>

        <!-- JSON Editor -->
        <div class="mb-4">
          <label class="text-sm font-semibold uppercase text-gray-200 mb-2 block">{{ $strings.LabelFullMetadataJson || 'Full Metadata JSON' }}</label>
          <textarea v-model="editableMetadataJson" class="w-full h-96 p-4 bg-gray-800 text-white border border-gray-600 rounded font-mono text-sm" placeholder="Loading metadata..." :class="{ 'border-red-500': jsonError }"></textarea>
          <p v-if="jsonError" class="text-red-400 text-sm mt-1">{{ jsonError }}</p>
          <p class="text-gray-400 text-xs mt-1">{{ $strings.LabelJsonEditorHint || 'Edit the JSON above to modify metadata, chapters, and all other properties. Click Format JSON to prettify the structure.' }}</p>
        </div>
      </div>
      <!-- m4b embed action buttons -->
      <div v-else-if="isM4BTool" class="w-full flex items-center mb-4">
        <div class="grow" />

        <ui-btn v-if="!isTaskFinished && processing" color="bg-error" :loading="isCancelingEncode" class="mr-2" @click.stop="cancelEncodeClick">{{ $strings.ButtonCancelEncode }}</ui-btn>
        <ui-btn v-if="!isTaskFinished" color="bg-primary" :loading="processing" :progress="progress" @click.stop="encodeM4bClick">{{ $strings.ButtonStartM4BEncode }}</ui-btn>
        <p v-else-if="taskFailed" class="text-error text-lg font-semibold">{{ $strings.MessageM4BFailed }} {{ taskError }}</p>
        <p v-else class="text-success text-lg font-semibold">{{ $strings.MessageM4BFinished }}</p>
      </div>

      <!-- show encoding options for running task -->
      <div v-if="encodeTaskHasEncodingOptions" class="mb-4 pb-4 border-b border-white/10">
        <div class="flex flex-wrap -mx-2">
          <ui-text-input-with-label ref="bitrateInput" v-model="encodingOptions.bitrate" readonly :label="$strings.LabelAudioBitrate" class="m-2 max-w-40" @input="bitrateChanged" />
          <ui-text-input-with-label ref="channelsInput" v-model="encodingOptions.channels" readonly :label="$strings.LabelAudioChannels" class="m-2 max-w-40" @input="channelsChanged" />
          <ui-text-input-with-label ref="codecInput" v-model="encodingOptions.codec" readonly :label="$strings.LabelAudioCodec" class="m-2 max-w-40" @input="codecChanged" />
        </div>
      </div>
      <div v-else-if="isM4BTool" class="mb-4">
        <widgets-encoder-options-card ref="encoderOptionsCard" :audio-tracks="audioFiles" :disabled="processing || isTaskFinished" />
      </div>
      <div class="mb-4">
        <div v-if="isEmbedTool" class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">{{ $strings.LabelEncodingInfoEmbedded }}</p>
        </div>
        <div v-else-if="isEditTool" class="flex items-start mb-2">
          <span class="material-symbols text-base text-info pt-1">edit</span>
          <p class="text-gray-200 ml-2">{{ $strings.LabelEditMetadataInfo || 'Edit the metadata fields above and click Save to update the audiobook information.' }}</p>
        </div>
        <div v-else-if="isM4BTool" class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">
            {{ $strings.LabelEncodingFinishedM4B }} <span class="rounded-md bg-neutral-600 text-sm text-white py-0.5 px-1 font-mono">.../{{ libraryItemRelPath }}/</span>.
          </p>
        </div>

        <div v-if="shouldBackupAudioFiles || isM4BTool" class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">
            {{ $strings.LabelEncodingBackupLocation }} <span class="rounded-md bg-neutral-600 text-sm text-white py-0.5 px-1 font-mono">/metadata/cache/items/{{ libraryItemId }}/</span>. {{ $strings.LabelEncodingClearItemCache }}
          </p>
        </div>
        <div v-if="isEmbedTool && audioFiles.length > 1" class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">{{ $strings.LabelEncodingChaptersNotEmbedded }}</p>
        </div>
        <div v-if="isM4BTool" class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">{{ $strings.LabelEncodingTimeWarning }}</p>
        </div>
        <div v-if="isM4BTool" class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">{{ $strings.LabelEncodingWatcherDisabled }}</p>
        </div>
        <div class="flex items-start mb-2">
          <span class="material-symbols text-base text-warning pt-1">star</span>
          <p class="text-gray-200 ml-2">{{ $strings.LabelEncodingStartedNavigation }}</p>
        </div>
      </div>
    </div>

    <div class="w-full max-w-4xl mx-auto">
      <p class="mb-2 font-semibold">{{ $strings.HeaderAudioTracks }}</p>
      <div class="w-full mx-auto border border-white/10 bg-bg">
        <div class="flex py-2 px-4 bg-primary/25">
          <div class="w-10 text-xs font-semibold text-gray-200">#</div>
          <div class="grow text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelFilename }}</div>
          <div class="w-20 text-xs font-semibold uppercase text-gray-200 hidden lg:block">{{ $strings.LabelChannels }}</div>
          <div class="w-16 text-xs font-semibold uppercase text-gray-200 hidden md:block">{{ $strings.LabelCodec }}</div>
          <div class="w-16 text-xs font-semibold uppercase text-gray-200 hidden md:block">{{ $strings.LabelBitrate }}</div>
          <div class="w-16 text-xs font-semibold uppercase text-gray-200">{{ $strings.LabelSize }}</div>
          <div class="w-24"></div>
        </div>
        <div v-for="file in audioFiles" :key="file.index" class="flex py-2 px-4 text-xs sm:text-sm" :class="file.index % 2 === 0 ? 'bg-primary/25' : ''">
          <div class="w-10 min-w-10">{{ file.index }}</div>
          <div class="grow">
            {{ file.metadata.filename }}
          </div>
          <div class="w-20 min-w-20 text-gray-200 hidden lg:block">{{ file.channels || 'unknown' }} ({{ file.channelLayout || 'unknown' }})</div>
          <div class="w-16 min-w-16 text-gray-200 hidden md:block">
            {{ file.codec || 'unknown' }}
          </div>
          <div class="w-16 min-w-16 text-gray-200 hidden md:block">
            {{ $bytesPretty(file.bitRate || 0, 0) }}
          </div>
          <div class="w-16 min-w-16 text-gray-200">
            {{ $bytesPretty(file.metadata.size) }}
          </div>
          <div class="w-24 min-w-24">
            <div class="flex justify-center">
              <span v-if="audioFilesFinished[file.ino]" class="material-symbols text-xl text-success leading-none">check_circle</span>
              <div v-else-if="audioFilesEncoding[file.ino]">
                <span class="font-mono text-success leading-none">{{ audioFilesEncoding[file.ino] }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  async asyncData({ store, params, app, redirect, route }) {
    if (!store.state.user.user) {
      return redirect(`/login?redirect=${route.path}`)
    }
    if (!store.getters['user/getIsAdminOrUp']) {
      return redirect('/?error=unauthorized')
    }
    const libraryItem = await app.$axios.$get(`/api/items/${params.id}?expanded=1`).catch((error) => {
      console.error('Failed', error)
      return false
    })
    if (!libraryItem) {
      console.error('Not found...', params.id)
      return redirect('/?error=not found')
    }
    if (libraryItem.mediaType !== 'book') {
      console.error('Invalid media type')
      return redirect('/?error=invalid media type')
    }
    if (!libraryItem.media.audioFiles.length) {
      console.error('No audio files')
      return redirect('/?error=no audio files')
    }

    // Fetch and set library if this items library does not match the current
    if (store.state.libraries.currentLibraryId !== libraryItem.libraryId || !store.state.libraries.filterData) {
      await store.dispatch('libraries/fetch', libraryItem.libraryId)
    }

    return {
      libraryItem
    }
  },
  data() {
    return {
      processing: false,
      metadataObject: null,
      selectedTool: 'embed',
      isCancelingEncode: false,
      shouldBackupAudioFiles: true,
      encodingOptions: {
        bitrate: '128k',
        channels: '2',
        codec: 'aac'
      }, // Edit tool data
      savingMetadata: false,
      editableMetadataJson: '',
      jsonError: null
    }
  },
  watch: {
    task: {
      handler(newVal) {
        if (newVal) {
          this.taskUpdated(newVal)
        }
      }
    }
  },
  computed: {
    audioFilesEncoding() {
      return this.$store.getters['tasks/getAudioFilesEncoding'](this.libraryItemId) || {}
    },
    audioFilesFinished() {
      return this.$store.getters['tasks/getAudioFilesFinished'](this.libraryItemId) || {}
    },
    progress() {
      return this.$store.getters['tasks/getTaskProgress'](this.libraryItemId) || '0%'
    },
    isEmbedTool() {
      return this.selectedTool === 'embed'
    },
    isM4BTool() {
      return this.selectedTool === 'm4b'
    },
    isEditTool() {
      return this.selectedTool === 'edit'
    },
    libraryItemId() {
      return this.libraryItem.id
    },
    libraryItemRelPath() {
      return this.libraryItem.relPath
    },
    media() {
      return this.libraryItem.media || {}
    },
    mediaMetadata() {
      return this.media.metadata || {}
    },
    audioFiles() {
      return (this.media.audioFiles || []).filter((af) => !af.exclude)
    },
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    metadataChapters() {
      return this.media.chapters || []
    },
    availableTools() {
      return [
        { value: 'embed', text: this.$strings.LabelToolsEmbedMetadata },
        { value: 'm4b', text: this.$strings.LabelToolsM4bEncoder },
        { value: 'edit', text: this.$strings.LabelToolsEditMetadata || 'Edit Metadata' }
      ]
    },
    taskFailed() {
      return this.isTaskFinished && this.task.isFailed
    },
    taskError() {
      return this.taskFailed ? this.task.error || 'Unknown Error' : null
    },
    isTaskFinished() {
      return this.task && this.task.isFinished
    },
    tasks() {
      return this.$store.getters['tasks/getTasksByLibraryItemId'](this.libraryItemId)
    },
    embedTask() {
      return this.tasks.find((t) => t.action === 'embed-metadata')
    },
    encodeTask() {
      return this.tasks.find((t) => t.action === 'encode-m4b')
    },
    task() {
      if (this.isEmbedTool) return this.embedTask
      else if (this.isM4BTool) return this.encodeTask
      else if (this.isEditTool) return null // Edit tool doesn't use background tasks
      return null
    },
    taskRunning() {
      return this.task && !this.task.isFinished
    },
    queuedEmbedLIds() {
      return this.$store.state.tasks.queuedEmbedLIds || []
    },
    isMetadataEmbedQueued() {
      return this.queuedEmbedLIds.some((lid) => lid === this.libraryItemId)
    },
    encodeTaskHasEncodingOptions() {
      return this.isM4BTool && !!this.encodeTask?.data.encodeOptions && Object.keys(this.encodeTask.data.encodeOptions).length > 0
    }
  },
  methods: {
    initEditableMetadata() {
      // Create a comprehensive metadata object including everything
      const fullMetadata = {
        // Basic metadata
        metadata: { ...this.mediaMetadata },
        // Chapters
        chapters: [...(this.metadataChapters || [])],
        // Audio files info (read-only reference)
        audioFiles: this.audioFiles.map((file) => ({
          index: file.index,
          filename: file.metadata.filename,
          size: file.metadata.size,
          duration: file.duration,
          codec: file.codec,
          bitRate: file.bitRate,
          channels: file.channels,
          channelLayout: file.channelLayout
        }))
      }

      this.editableMetadataJson = JSON.stringify(fullMetadata, null, 2)
      this.jsonError = null
    },
    formatJson() {
      try {
        const parsed = JSON.parse(this.editableMetadataJson)
        this.editableMetadataJson = JSON.stringify(parsed, null, 2)
        this.jsonError = null
      } catch (error) {
        this.jsonError = 'Invalid JSON: ' + error.message
      }
    },
    resetJson() {
      this.initEditableMetadata()
    },
    async saveMetadata() {
      this.savingMetadata = true
      this.jsonError = null

      try {
        // Parse and validate the JSON
        let parsedData
        try {
          parsedData = JSON.parse(this.editableMetadataJson)
        } catch (error) {
          this.jsonError = 'Invalid JSON: ' + error.message
          return
        }

        // Extract metadata and chapters from the parsed data
        const updatedMetadata = parsedData.metadata || {}
        const updatedChapters = parsedData.chapters || []

        // Build the payload for the API
        const payload = {
          media: {
            metadata: updatedMetadata,
            chapters: updatedChapters
          }
        }

        await this.$axios.$patch(`/api/items/${this.libraryItemId}/media`, payload.media)

        this.$toast.success(this.$strings.ToastItemSaveSuccess || 'Metadata updated successfully')

        // Refresh the library item data
        const updatedItem = await this.$axios.$get(`/api/items/${this.libraryItemId}?expanded=1`)
        this.libraryItem = updatedItem

        // Re-fetch metadata object for the current view
        this.fetchMetadataEmbedObject()

        // Reinitialize the JSON editor with the updated data
        this.initEditableMetadata()
      } catch (error) {
        console.error('Failed to save metadata', error)
        const errorMsg = error.response?.data || 'Failed to save metadata'
        this.$toast.error(errorMsg)
      } finally {
        this.savingMetadata = false
      }
    },
    toggleBackupAudioFiles(val) {
      localStorage.setItem('embedMetadataShouldBackup', val ? 1 : 0)
    },
    bitrateChanged(val) {
      localStorage.setItem('embedMetadataBitrate', val)
    },
    channelsChanged(val) {
      localStorage.setItem('embedMetadataChannels', val)
    },
    codecChanged(val) {
      localStorage.setItem('embedMetadataCodec', val)
    },
    cancelEncodeClick() {
      this.isCancelingEncode = true
      this.$axios
        .$delete(`/api/tools/item/${this.libraryItemId}/encode-m4b`)
        .then(() => {
          this.$toast.success(this.$strings.ToastEncodeCancelSucces)
        })
        .catch((error) => {
          console.error('Failed to cancel encode', error)
          this.$toast.error(this.$strings.ToastEncodeCancelFailed)
        })
        .finally(() => {
          this.isCancelingEncode = false
        })
    },
    encodeM4bClick() {
      if (this.$refs.bitrateInput) this.$refs.bitrateInput.blur()
      if (this.$refs.channelsInput) this.$refs.channelsInput.blur()
      if (this.$refs.codecInput) this.$refs.codecInput.blur()

      const encodeOptions = this.$refs.encoderOptionsCard.getEncodingOptions()

      this.encodingOptions = encodeOptions

      const queryParams = new URLSearchParams(encodeOptions)

      this.processing = true
      this.$axios
        .$post(`/api/tools/item/${this.libraryItemId}/encode-m4b?${queryParams.toString()}`)
        .then(() => {
          console.log('Ab m4b merge started')
        })
        .catch((error) => {
          var errorMsg = error.response ? error.response.data || 'Unknown Error' : 'Unknown Error'
          this.$toast.error(errorMsg)
          this.processing = false
        })
    },
    embedClick() {
      const payload = {
        message: this.$getString('MessageConfirmEmbedMetadataInAudioFiles', [this.audioFiles.length]),
        callback: (confirmed) => {
          if (confirmed) {
            this.updateAudioFileMetadata()
          }
        },
        type: 'yesNo'
      }
      this.$store.commit('globals/setConfirmPrompt', payload)
    },
    updateAudioFileMetadata() {
      this.processing = true
      this.$axios
        .$post(`/api/tools/item/${this.libraryItemId}/embed-metadata?backup=${this.shouldBackupAudioFiles ? 1 : 0}`)
        .then(() => {
          console.log('Audio metadata encode started')
        })
        .catch((error) => {
          console.error('Audio metadata encode failed', error)
          this.processing = false
        })
    },
    selectedToolUpdated() {
      let newurl = window.location.protocol + '//' + window.location.host + window.location.pathname + `?tool=${this.selectedTool}`
      window.history.replaceState({ path: newurl }, '', newurl)

      // Initialize editable metadata when edit tool is selected
      if (this.isEditTool) {
        this.initEditableMetadata()
      }
    },
    init() {
      this.fetchMetadataEmbedObject()
      if (this.$route.query.tool === 'm4b') {
        if (this.availableTools.some((t) => t.value === 'm4b')) {
          this.selectedTool = 'm4b'
        } else {
          this.selectedToolUpdated()
        }
      } else if (this.$route.query.tool === 'edit') {
        if (this.availableTools.some((t) => t.value === 'edit')) {
          this.selectedTool = 'edit'
          this.initEditableMetadata()
        } else {
          this.selectedToolUpdated()
        }
      }

      if (this.task) this.taskUpdated(this.task)

      const shouldBackupAudioFiles = localStorage.getItem('embedMetadataShouldBackup')
      this.shouldBackupAudioFiles = shouldBackupAudioFiles != 0

      if (this.encodeTaskHasEncodingOptions) {
        if (this.encodeTask.data.encodeOptions.bitrate) this.encodingOptions.bitrate = this.encodeTask.data.encodeOptions.bitrate
        if (this.encodeTask.data.encodeOptions.channels) this.encodingOptions.channels = this.encodeTask.data.encodeOptions.channels
        if (this.encodeTask.data.encodeOptions.codec) this.encodingOptions.codec = this.encodeTask.data.encodeOptions.codec
      }
    },
    fetchMetadataEmbedObject() {
      this.$axios
        .$get(`/api/items/${this.libraryItemId}/metadata-object`)
        .then((metadataObject) => {
          this.metadataObject = metadataObject
        })
        .catch((error) => {
          console.error('Failed to fetch metadata object', error)
        })
    },
    taskUpdated(task) {
      this.processing = !task.isFinished
    },
    editItem() {
      this.$store.commit('showEditModal', this.libraryItem)
    },
    libraryItemUpdated(libraryItem) {
      if (libraryItem.id === this.libraryItem.id) {
        this.libraryItem = libraryItem
        this.fetchMetadataEmbedObject()
      }
    }
  },
  mounted() {
    this.init()

    this.$eventBus.$on(`${this.libraryItem.id}_updated`, this.libraryItemUpdated)
  },
  beforeDestroy() {
    this.$eventBus.$off(`${this.libraryItem.id}_updated`, this.libraryItemUpdated)
  }
}
</script>
