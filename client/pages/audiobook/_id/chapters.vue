<template>
  <div id="page-wrapper" class="bg-bg page overflow-y-auto relative" :class="streamLibraryItem ? 'streaming' : ''">
    <div class="flex items-center py-4 px-4 max-w-7xl mx-auto">
      <nuxt-link :to="`/item/${libraryItem.id}`" class="hover:underline">
        <h1 class="text-lg lg:text-xl">{{ title }}</h1>
      </nuxt-link>
      <button class="w-7 h-7 flex items-center justify-center mx-4 hover:scale-110 duration-100 transform text-gray-200 hover:text-white" @click="editItem">
        <span class="material-symbols text-base">edit</span>
      </button>
      <div class="grow hidden md:block" />
      <p class="text-base hidden md:block">{{ $strings.LabelDuration }}:</p>
      <p class="text-base font-mono ml-4 hidden md:block">{{ $secondsToTimestamp(mediaDurationRounded) }}</p>
    </div>

    <div class="flex flex-wrap-reverse lg:flex-nowrap justify-center py-4 px-4">
      <div class="w-full max-w-3xl py-4">
        <div class="flex items-center">
          <div class="w-12 hidden lg:block" />
          <p class="text-lg mb-4 font-semibold">{{ $strings.HeaderChapters }}</p>
          <div class="grow" />
          <ui-checkbox v-model="showSecondInputs" checkbox-bg="primary" small label-class="text-sm text-gray-200 pl-1" :label="$strings.LabelShowSeconds" class="mx-2" />
          <div class="w-32 hidden lg:block" />
        </div>
        <div class="flex items-center mb-3 py-1 -mx-1">
          <div class="w-12 hidden lg:block" />
          <ui-btn v-if="chapters.length" color="bg-primary" small class="mx-1 whitespace-nowrap" @click.stop="removeAllChaptersClick">{{ $strings.ButtonRemoveAll }}</ui-btn>
          <ui-btn v-if="newChapters.length > 1" :color="showShiftTimes ? 'bg-bg' : 'bg-primary'" class="mx-1 whitespace-nowrap" small @click="showShiftTimes = !showShiftTimes">{{ $strings.ButtonShiftTimes }}</ui-btn>
          <ui-btn color="bg-primary" small :class="{ 'mx-1': newChapters.length > 1 }" @click="showFindChaptersModal = true">{{ $strings.ButtonLookup }}</ui-btn>
          <div class="grow" />
          <ui-btn v-if="hasChanges" small class="mx-1" @click.stop="resetChapters">{{ $strings.ButtonReset }}</ui-btn>
          <ui-btn v-if="hasChanges" color="bg-success" class="mx-1" :disabled="!hasChanges" small @click="saveChapters">{{ $strings.ButtonSave }}</ui-btn>
          <div class="w-32 hidden lg:block" />
        </div>

        <div class="overflow-hidden">
          <transition name="slide">
            <div v-if="showShiftTimes" class="flex mb-4">
              <div class="w-12 hidden lg:block" />
              <div class="grow">
                <div class="flex items-center">
                  <p class="text-sm mb-1 font-semibold pr-2">{{ $strings.LabelTimeToShift }}</p>
                  <ui-text-input v-model="shiftAmount" type="number" class="max-w-20" style="height: 30px" />
                  <ui-btn color="bg-primary" class="mx-1" small @click="shiftChapterTimes">{{ $strings.ButtonAdd }}</ui-btn>
                  <div class="grow" />
                  <span class="material-symbols text-gray-200 hover:text-white cursor-pointer" @click="showShiftTimes = false">expand_less</span>
                </div>
                <p class="text-xs py-1.5 text-gray-300 max-w-md">{{ $strings.NoteChapterEditorTimes }}</p>
              </div>
              <div class="w-32 hidden lg:block" />
            </div>
          </transition>
        </div>

        <div class="flex text-xs uppercase text-gray-300 font-semibold mb-2">
          <div class="w-8 min-w-8 md:w-12 md:min-w-12"></div>
          <div class="w-38 min-w-38 md:w-40 md:min-w-40 px-1 pl-8">{{ $strings.LabelStart }}</div>
          <div class="grow px-1 min-w-54">{{ $strings.LabelTitle }}</div>
          <div class="w-7 min-w-7 px-1 flex items-center justify-center">
            <ui-tooltip :text="allChaptersLocked ? $strings.TooltipUnlockAllChapters : $strings.TooltipLockAllChapters" direction="bottom">
              <button class="w-7 h-7 rounded-full flex items-center justify-center cursor-pointer transition-colors duration-150" :class="allChaptersLocked ? 'text-orange-400 hover:text-orange-300' : 'text-gray-300 hover:text-white'" @click="toggleAllChaptersLock">
                <span class="material-symbols text-xl">{{ allChaptersLocked ? 'lock' : 'lock_open' }}</span>
              </button>
            </ui-tooltip>
          </div>
          <div class="w-32"></div>
        </div>
        <div v-for="chapter in newChapters" :key="chapter.id" class="flex py-1">
          <div class="w-8 min-w-8 md:w-12 md:min-w-12">#{{ chapter.id + 1 }}</div>
          <div class="w-38 min-w-38 md:w-40 md:min-w-40 px-1">
            <div class="flex items-center gap-1">
              <ui-tooltip :text="$strings.TooltipSubtractOneSecond" direction="bottom">
                <button
                  class="w-6 h-6 rounded-full flex items-center justify-center text-gray-300 hover:text-white transform hover:scale-110 duration-150 flex-shrink-0"
                  :class="{ 'opacity-50 cursor-not-allowed': chapter.id === 0 && chapter.start - timeIncrementAmount < 0 }"
                  @click="incrementChapterTime(chapter, -timeIncrementAmount)"
                  :disabled="chapter.id === 0 && chapter.start - timeIncrementAmount < 0"
                >
                  <span class="material-symbols text-sm">remove</span>
                </button>
              </ui-tooltip>              <div class="flex-1 min-w-0">
                <ui-text-input v-if="showSecondInputs" v-model="chapter.start" type="number" class="text-xs" @change="debouncedCheckChapters" />
                <ui-time-picker v-else class="text-xs" v-model="chapter.start" :show-three-digit-hour="mediaDuration >= 360000" @change="debouncedCheckChapters" />
              </div>

              <ui-tooltip :text="$strings.TooltipAddOneSecond" direction="bottom">
                <button class="w-6 h-6 rounded-full flex items-center justify-center text-gray-300 hover:text-white transform hover:scale-110 duration-150 flex-shrink-0" :class="{ 'opacity-50 cursor-not-allowed': chapter.start + timeIncrementAmount >= mediaDuration }" @click="incrementChapterTime(chapter, timeIncrementAmount)" :disabled="chapter.start + timeIncrementAmount >= mediaDuration">
                  <span class="material-symbols text-sm">add</span>
                </button>
              </ui-tooltip>
            </div>
          </div>          <div class="grow px-1">
            <ui-text-input v-model="chapter.title" @change="debouncedCheckChapters" class="text-xs min-w-52" />
          </div>
          <div class="w-7 min-w-7 px-1 py-1">
            <div class="flex items-center justify-center">
              <ui-tooltip :text="lockedChapters.has(chapter.id) ? $strings.TooltipUnlockChapter : $strings.TooltipLockChapter" direction="bottom">
                <button class="w-7 h-7 rounded-full flex items-center justify-center transform hover:scale-110 duration-150 flex-shrink-0" :class="lockedChapters.has(chapter.id) ? 'text-orange-400 hover:text-orange-300' : 'text-gray-300 hover:text-white'" @click="toggleChapterLock(chapter, $event)">
                  <span class="material-symbols text-base">{{ lockedChapters.has(chapter.id) ? 'lock' : 'lock_open' }}</span>
                </button>
              </ui-tooltip>
            </div>
          </div>
          <div class="w-32 min-w-32 px-2 py-1">
            <div class="flex items-center">
              <ui-tooltip :text="$strings.MessageRemoveChapter" direction="bottom">
                <button v-if="newChapters.length > 1" class="w-7 h-7 rounded-full flex items-center justify-center text-gray-300 hover:text-error transform hover:scale-110 duration-150" @click="removeChapter(chapter)">
                  <span class="material-symbols text-base">delete</span>
                </button>
              </ui-tooltip>

              <ui-tooltip :text="$strings.MessageInsertChapterBelow" direction="bottom">
                <button class="w-7 h-7 rounded-full flex items-center justify-center text-gray-300 hover:text-success transform hover:scale-110 duration-150" @click="addChapter(chapter)">
                  <span class="material-symbols text-lg">add_row_below</span>
                </button>
              </ui-tooltip>
              <ui-tooltip :text="selectedChapterId === chapter.id && isPlayingChapter ? $strings.MessagePauseChapter : $strings.MessagePlayChapter" direction="bottom">
                <button class="w-7 h-7 rounded-full flex items-center justify-center text-gray-300 hover:text-white transform hover:scale-110 duration-150" @click="playChapter(chapter)">
                  <widgets-loading-spinner v-if="selectedChapterId === chapter.id && isLoadingChapter" />
                  <span v-else-if="selectedChapterId === chapter.id && isPlayingChapter" class="material-symbols text-base">pause</span>
                  <span v-else class="material-symbols text-base">play_arrow</span>
                </button>
              </ui-tooltip>
              <ui-tooltip v-if="selectedChapterId === chapter.id && (isPlayingChapter || isLoadingChapter)" :text="$strings.TooltipAdjustChapterStart" direction="bottom">
                <div class="ml-2 text-xs text-gray-300 font-mono min-w-10 cursor-pointer hover:text-white transition-colors duration-150" @click="adjustChapterStartTime(chapter)">{{ elapsedTime }}s</div>
              </ui-tooltip>
              <ui-tooltip v-if="chapter.error" :text="chapter.error" direction="left">
                <button class="w-7 h-7 rounded-full flex items-center justify-center text-error">
                  <span class="material-symbols text-lg">error_outline</span>
                </button>
              </ui-tooltip>
            </div>
          </div>
        </div>
        <div class="flex items-center mt-4 mb-2">
          <div class="w-8 min-w-8 md:w-12 md:min-w-12"></div>
          <div class="w-38 min-w-38 md:w-40 md:min-w-40 px-1"></div>
          <div class="flex items-center gap-2 grow px-1">
            <ui-text-input v-model="bulkChapterInput" :placeholder="$strings.PlaceholderBulkChapterInput" class="text-xs grow min-w-52" @keyup.enter="handleBulkChapterAdd" />
          </div>
          <div class="w-39 min-w-39 px-1 py-1">
            <ui-tooltip :text="$strings.TooltipAddChapters" direction="bottom">
              <button class="w-5 h-5 rounded-full flex items-center justify-center text-gray-300 hover:text-success transform hover:scale-110 duration-150 flex-shrink-0" :class="{ 'opacity-50 cursor-not-allowed': !bulkChapterInput.trim() }" :disabled="!bulkChapterInput.trim()" @click="handleBulkChapterAdd">
                <span class="material-symbols text-lg">add</span>
              </button>
            </ui-tooltip>
          </div>
        </div>
      </div>
      <div class="w-full max-w-xl py-4 px-2">
        <!-- Tab Navigation -->
        <div class="flex border-b border-gray-600 mb-4">
          <button v-for="tab in chapterSourceTabs" :key="tab.id" @click="activeTab = tab.id" class="px-4 py-2 text-sm font-medium transition-colors duration-200" :class="activeTab === tab.id ? 'text-white border-b-2 border-primary' : 'text-gray-400 hover:text-gray-200'">
            {{ tab.label }}
          </button>
        </div>

        <!-- Audio Tracks Tab -->
        <div v-if="activeTab === 'tracks'" class="space-y-4">
          <div class="flex items-center justify-between">
            <p class="text-lg font-semibold">{{ $strings.HeaderAudioTracks }}</p>
            <div class="flex items-center">
              <ui-btn small @click="setChaptersFromTracks">{{ $strings.ButtonSetChaptersFromTracks }}</ui-btn>
              <ui-tooltip :text="$strings.MessageSetChaptersFromTracksDescription" direction="top" class="flex items-center mx-1 cursor-default">
                <span class="material-symbols text-xl text-gray-200">info</span>
              </ui-tooltip>
            </div>
          </div>
          <div class="flex text-xs uppercase text-gray-300 font-semibold mb-2">
            <div class="grow">{{ $strings.LabelFilename }}</div>
            <div class="w-20">{{ $strings.LabelDuration }}</div>
            <div class="w-20 hidden md:block text-center">{{ $strings.HeaderChapters }}</div>
          </div>
          <div v-for="track in audioTracks" :key="track.ino" class="flex items-center py-2" :class="currentTrackIndex === track.index && isPlayingChapter ? 'bg-success/10' : ''">
            <div class="grow max-w-[calc(100%-80px)] pr-2">
              <p class="text-xs truncate max-w-sm">{{ track.metadata.filename }}</p>
            </div>
            <div class="w-20" style="min-width: 80px">
              <p class="text-xs font-mono text-gray-200">{{ $secondsToTimestamp(Math.round(track.duration), false, true) }}</p>
            </div>
            <div class="w-20 hidden md:flex justify-center" style="min-width: 80px"><span v-if="(track.chapters || []).length" class="material-symbols text-success text-sm">check</span></div>
          </div>
        </div>
        <!-- Cue File Tab -->
        <div v-if="activeTab === 'cue'" class="space-y-4">
          <div class="flex items-center justify-between">
            <p class="text-lg font-semibold">Cue File</p>
            <div class="flex items-center gap-2">
              <ui-btn small @click="loadCurrentChaptersAsCue" :disabled="!newChapters.length">Load Current</ui-btn>
              <ui-btn small @click="setChaptersFromCueFile" :disabled="!cueFileInput.trim()">Parse Cue File</ui-btn>
            </div>
          </div>
          <div class="space-y-2">
            <p class="text-sm text-gray-300">Paste your cue file content below:</p>
            <textarea
              v-model="cueFileInput"
              placeholder='REM Example cue file format:&#10;FILE "audiobook.flac" WAVE&#10;  TRACK 01 AUDIO&#10;    TITLE "Chapter 1: Introduction"&#10;    INDEX 01 00:00:00&#10;  TRACK 02 AUDIO&#10;    TITLE "Chapter 2: The Beginning"&#10;    INDEX 01 05:30:25&#10;  TRACK 03 AUDIO&#10;    TITLE "Chapter 3: The Middle"&#10;    INDEX 01 12:45:50'
              class="w-full h-48 p-3 text-sm bg-gray-800 border border-gray-600 rounded-md resize-none focus:border-primary focus:outline-none"
              style="font-family: 'Courier New', monospace"
            ></textarea>
            <p class="text-xs text-gray-400">Supports standard CUE sheet format with TRACK, TITLE, and INDEX entries</p>
          </div>
        </div>
        <!-- JSON Tab -->
        <div v-if="activeTab === 'json'" class="space-y-4">
          <div class="flex items-center justify-between">
            <p class="text-lg font-semibold">JSON Chapters</p>
            <div class="flex items-center gap-2">
              <ui-btn small @click="loadCurrentChaptersAsJson" :disabled="!newChapters.length">Load Current</ui-btn>
              <ui-btn small @click="setChaptersFromJson" :disabled="!jsonInput.trim()">Parse JSON</ui-btn>
            </div>
          </div>
          <div class="space-y-2">
            <p class="text-sm text-gray-300">Paste JSON chapter data below:</p>
            <textarea
              v-model="jsonInput"
              placeholder='[&#10;  {"title": "Chapter 1: Introduction", "start": 0},&#10;  {"title": "Chapter 2: The Beginning", "start": 330},&#10;  {"title": "Chapter 3: The Middle", "start": 1250, "end": 2100}&#10;]'
              class="w-full h-48 p-3 text-sm bg-gray-800 border border-gray-600 rounded-md resize-none focus:border-primary focus:outline-none"
              style="font-family: 'Courier New', monospace"
            ></textarea>
            <p class="text-xs text-gray-400">Array of objects with "title" (string) and "start" (seconds). Optional "end" field.</p>
          </div>
        </div>
      </div>
    </div>

    <div v-if="saving" class="w-full h-full absolute top-0 left-0 bottom-0 right-0 z-30 bg-black/25 flex items-center justify-center">
      <ui-loading-indicator />
    </div>

    <modals-modal v-model="showFindChaptersModal" name="edit-book" :width="500" :processing="findingChapters">
      <template #outer>
        <div class="absolute top-0 left-0 p-5 w-2/3 overflow-hidden pointer-events-none">
          <p class="text-3xl text-white truncate pointer-events-none">{{ $strings.HeaderFindChapters }}</p>
        </div>
      </template>
      <div class="w-full h-full max-h-full text-sm rounded-lg bg-bg shadow-lg border border-black-300 relative">
        <div v-if="!chapterData" class="flex flex-col items-center justify-center p-20">
          <div class="relative">
            <div class="flex items-end space-x-2">
              <ui-text-input-with-label v-model.trim="asinInput" label="ASIN" class="flex-grow" />
              <ui-dropdown v-model="regionInput" :label="$strings.LabelRegion" small :items="audibleRegions" class="w-20 max-w-20" />
              <ui-btn color="bg-primary" @click="findChapters">{{ $strings.ButtonSearch }}</ui-btn>
            </div>
            <div class="mt-4">
              <ui-checkbox v-model="removeBranding" :label="$strings.LabelRemoveAudibleBranding" small checkbox-bg="bg" label-class="pl-2 text-base text-sm" @click="toggleRemoveBranding" />
            </div>
            <div class="absolute left-0 mt-1.5 text-error text-s h-5">
              <p v-if="asinError">{{ asinError }}</p>
              <p v-if="asinError">{{ $strings.MessageAsinCheck }}</p>
            </div>
            <div class="invisible mt-1 text-xs"></div>
          </div>
        </div>
        <div v-else class="w-full p-4">
          <div class="flex justify-between mb-4">
            <p>
              {{ $strings.LabelDurationFound }} <span class="font-semibold">{{ $secondsToTimestamp(chapterData.runtimeLengthSec) }}</span
              ><br />
              <span class="font-semibold" :class="{ 'text-warning': chapters.length !== chapterData.chapters.length }">{{ chapterData.chapters.length }}</span> {{ $strings.LabelChaptersFound }}
            </p>
            <p>
              {{ $strings.LabelYourAudiobookDuration }}: <span class="font-semibold">{{ $secondsToTimestamp(mediaDurationRounded) }}</span
              ><br />
              Your audiobook has <span class="font-semibold" :class="{ 'text-warning': chapters.length !== chapterData.chapters.length }">{{ chapters.length }}</span> chapters
            </p>
          </div>
          <widgets-alert v-if="chapterData.runtimeLengthSec > mediaDurationRounded" type="warning" class="mb-2"> {{ $strings.MessageYourAudiobookDurationIsShorter }} </widgets-alert>
          <widgets-alert v-else-if="chapterData.runtimeLengthSec < mediaDurationRounded" type="warning" class="mb-2"> {{ $strings.MessageYourAudiobookDurationIsLonger }} </widgets-alert>

          <div class="flex py-0.5 text-xs font-semibold uppercase text-gray-300 mb-1">
            <div class="w-24 px-2">{{ $strings.LabelStart }}</div>
            <div class="grow px-2">{{ $strings.LabelTitle }}</div>
          </div>
          <div class="w-full max-h-80 overflow-y-auto my-2">
            <div v-for="(chapter, index) in chapterData.chapters" :key="index" class="flex py-0.5 text-xs" :class="chapter.startOffsetSec > mediaDuration ? 'bg-error/20' : chapter.startOffsetSec + chapter.lengthMs / 1000 > mediaDuration ? 'bg-warning/20' : index % 2 === 0 ? 'bg-primary/30' : ''">
              <div class="w-24 min-w-24 px-2">
                <p class="font-mono">{{ $secondsToTimestamp(chapter.startOffsetSec) }}</p>
              </div>
              <div class="grow px-2">
                <p class="truncate max-w-sm">{{ chapter.title }}</p>
              </div>
            </div>
          </div>
          <div v-if="chapterData.runtimeLengthSec > mediaDurationRounded" class="w-full pt-2">
            <div class="flex items-center">
              <div class="w-2 h-2 bg-warning/50" />
              <p class="pl-2">{{ $strings.MessageChapterEndIsAfter }}</p>
            </div>
            <div class="flex items-center">
              <div class="w-2 h-2 bg-error/50" />
              <p class="pl-2">{{ $strings.MessageChapterStartIsAfter }}</p>
            </div>
          </div>
          <div class="flex items-center pt-2">
            <ui-btn small color="bg-primary" class="mr-1" @click="applyChapterNamesOnly">{{ $strings.ButtonMapChapterTitles }}</ui-btn>
            <ui-tooltip :text="$strings.MessageMapChapterTitles" direction="top" class="flex items-center">
              <span class="material-symbols text-xl text-gray-200">info</span>
            </ui-tooltip>
            <div class="grow" />
            <ui-btn small color="bg-success" @click="applyChapterData">{{ $strings.ButtonApplyChapters }}</ui-btn>
          </div>
        </div>
      </div>
    </modals-modal>
    <modals-modal v-model="showBulkChapterModal" name="bulk-chapters" :width="400">
      <template #outer>
        <div class="absolute top-0 left-0 p-5 w-2/3 overflow-hidden pointer-events-none">
          <p class="text-3xl text-white truncate pointer-events-none">{{ $strings.HeaderBulkChapterModal }}</p>
        </div>
      </template>
      <div class="w-full h-full max-h-full text-sm rounded-lg bg-bg shadow-lg border border-black-300 relative p-6">
        <div class="flex flex-col space-y-4">
          <p class="text-lg font-semibold">{{ $strings.HeaderBulkChapterModal }}</p>
          <p class="text-gray-300">{{ $strings.MessageBulkChapterPattern }}</p>

          <div v-if="detectedPattern" class="text-sm text-gray-400 bg-gray-800 p-2 rounded">
            <strong>{{ $strings.LabelDetectedPattern }}</strong> "{{ detectedPattern.before }}{{ formatNumberWithPadding(detectedPattern.startingNumber, detectedPattern) }}{{ detectedPattern.after }}"
            <br />
            <strong>{{ $strings.LabelNextChapters }}</strong>
            "{{ detectedPattern.before }}{{ formatNumberWithPadding(detectedPattern.startingNumber + 1, detectedPattern) }}{{ detectedPattern.after }}", "{{ detectedPattern.before }}{{ formatNumberWithPadding(detectedPattern.startingNumber + 2, detectedPattern) }}{{ detectedPattern.after }}", etc.
          </div>
          <div class="flex items-center space-x-2">
            <label class="text-sm font-medium">{{ $strings.LabelNumberOfChapters }}</label>
            <ui-text-input v-model="bulkChapterCount" type="number" min="1" max="50" class="w-20" @keyup.enter="addBulkChapters" />
          </div>
          <div class="flex items-center space-x-2 pt-4">
            <ui-btn color="bg-success" @click="addBulkChapters">{{ $strings.ButtonAddChapters }}</ui-btn>
            <ui-btn @click="showBulkChapterModal = false">{{ $strings.ButtonCancel }}</ui-btn>
          </div>
        </div>
      </div>
    </modals-modal>
  </div>
</template>

<script>
import path from 'path'

export default {
  async asyncData({ store, params, app, redirect, from }) {
    if (!store.getters['user/getUserCanUpdate']) {
      return redirect('/?error=unauthorized')
    }
    var libraryItem = await app.$axios.$get(`/api/items/${params.id}?expanded=1`).catch((error) => {
      console.error('Failed', error)
      return false
    })
    if (!libraryItem) {
      console.error('Not found...', params.id)
      return redirect('/')
    }
    if (libraryItem.mediaType != 'book') {
      console.error('Invalid media type')
      return redirect('/')
    }

    // Fetch and set library if this items library does not match the current
    if (store.state.libraries.currentLibraryId !== libraryItem.libraryId || !store.state.libraries.filterData) {
      await store.dispatch('libraries/fetch', libraryItem.libraryId)
    }

    var previousRoute = from ? from.fullPath : null
    if (from && from.path === '/login') previousRoute = null
    return {
      libraryItem,
      previousRoute
    }
  },  data() {
    return {
      newChapters: [],
      selectedChapter: null,
      showShiftTimes: false,
      shiftAmount: 0,
      audioEl: null,
      isPlayingChapter: false,
      isLoadingChapter: false,
      currentTrackIndex: 0,
      saving: false,
      asinInput: null,
      regionInput: 'US',
      findingChapters: false,
      showFindChaptersModal: false,
      chapterData: null,
      asinError: null,
      removeBranding: false,
      showSecondInputs: false,
      audibleRegions: ['US', 'CA', 'UK', 'AU', 'FR', 'DE', 'JP', 'IT', 'IN', 'ES'],
      hasChanges: false,
      timeIncrementAmount: 1,
      elapsedTime: 0,
      playStartTime: null,
      elapsedTimeInterval: null,
      lockedChapters: new Set(),
      lastSelectedLockIndex: null,
      bulkChapterInput: '',
      showBulkChapterModal: false,
      bulkChapterCount: 1,
      detectedPattern: null,
      activeTab: 'tracks',
      cueFileInput: '',
      jsonInput: '',
      checkChaptersTimeout: null
    }
  },
  computed: {
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    userToken() {
      return this.$store.getters['user/getToken']
    },
    media() {
      return this.libraryItem.media || {}
    },
    mediaMetadata() {
      return this.media.metadata || {}
    },
    title() {
      return this.mediaMetadata.title
    },
    mediaDuration() {
      return this.media.duration
    },
    mediaDurationRounded() {
      return Math.round(this.mediaDuration)
    },
    chapters() {
      return this.media.chapters || []
    },
    tracks() {
      return this.media.tracks || []
    },
    audioFiles() {
      return this.media.audioFiles || []
    },
    audioTracks() {
      return this.audioFiles.filter((af) => !af.exclude)
    },
    selectedChapterId() {
      return this.selectedChapter ? this.selectedChapter.id : null
    },
    allChaptersLocked() {
      return this.newChapters.length > 0 && this.newChapters.every((chapter) => this.lockedChapters.has(chapter.id))
    },
    chapterSourceTabs() {
      return [
        { id: 'tracks', label: 'Audio Tracks' },
        { id: 'cue', label: 'Cue File' },
        { id: 'json', label: 'JSON' }
      ]
    }
  },  methods: {
    // Debounced version of checkChapters to improve performance
    debouncedCheckChapters() {
      // Clear any existing timeout
      if (this.checkChaptersTimeout) {
        clearTimeout(this.checkChaptersTimeout)
      }
      // Set a new timeout to call checkChapters after 300ms of no changes
      this.checkChaptersTimeout = setTimeout(() => {
        this.checkChapters()
      }, 300)
    },
    
    formatNumberWithPadding(number, pattern) {
      if (!pattern || !pattern.hasLeadingZeros || !pattern.originalPadding) {
        return number.toString()
      }
      return number.toString().padStart(pattern.originalPadding, '0')
    },

    setChaptersFromTracks() {
      let currentStartTime = 0
      let index = 0
      const chapters = []
      for (const track of this.audioTracks) {
        chapters.push({
          id: index++,
          title: path.basename(track.metadata.filename, path.extname(track.metadata.filename)),
          start: currentStartTime,
          end: currentStartTime + track.duration
        })
        currentStartTime += track.duration
      }
      this.newChapters = chapters

      this.checkChapters()
    },
    setChaptersFromCueFile() {
      if (!this.cueFileInput.trim()) {
        this.$toast.error('Please paste cue file content first')
        return
      }

      try {
        const chapters = this.parseCueFile(this.cueFileInput)
        if (chapters.length === 0) {
          this.$toast.error('No chapters found in cue file')
          return
        }

        this.newChapters = chapters
        this.checkChapters()
        this.$toast.success(`Successfully parsed ${chapters.length} chapters from cue file`)
      } catch (error) {
        console.error('Failed to parse cue file:', error)
        this.$toast.error('Failed to parse cue file. Please check the format.')
      }
    },
    setChaptersFromJson() {
      if (!this.jsonInput.trim()) {
        this.$toast.error('Please paste JSON content first')
        return
      }

      try {
        const chapters = this.parseJsonChapters(this.jsonInput)
        if (chapters.length === 0) {
          this.$toast.error('No chapters found in JSON')
          return
        }

        this.newChapters = chapters
        this.checkChapters()
        this.$toast.success(`Successfully parsed ${chapters.length} chapters from JSON`)
      } catch (error) {
        console.error('Failed to parse JSON:', error)
        this.$toast.error('Failed to parse JSON. Please check the format.')
      }
    },
    parseCueFile(cueContent) {
      const lines = cueContent.split('\n').map((line) => line.trim())
      const chapters = []
      let currentTrack = null
      let trackNumber = 0

      for (const line of lines) {
        // Parse TRACK lines
        const trackMatch = line.match(/TRACK\s+(\d+)\s+AUDIO/i)
        if (trackMatch) {
          if (currentTrack) {
            chapters.push(currentTrack)
          }
          trackNumber = parseInt(trackMatch[1])
          currentTrack = {
            id: chapters.length,
            title: `Track ${trackNumber.toString().padStart(2, '0')}`,
            start: 0,
            end: 0
          }
          continue
        }

        // Parse TITLE lines
        const titleMatch = line.match(/TITLE\s+"([^"]+)"/i)
        if (titleMatch && currentTrack) {
          currentTrack.title = titleMatch[1]
          continue
        }

        // Parse INDEX lines (typically INDEX 01 for track start)
        const indexMatch = line.match(/INDEX\s+01\s+(\d{2}):(\d{2}):(\d{2})/i)
        if (indexMatch && currentTrack) {
          const minutes = parseInt(indexMatch[1])
          const seconds = parseInt(indexMatch[2])
          const frames = parseInt(indexMatch[3])
          // Convert frames to seconds (75 frames per second for CD audio)
          const totalSeconds = minutes * 60 + seconds + frames / 75
          currentTrack.start = Math.floor(totalSeconds)
          continue
        }
      } // Add the last track
      if (currentTrack) {
        chapters.push(currentTrack)
      }

      // Sort by start time, then by title (with natural number ordering)
      chapters.sort((a, b) => {
        // Primary sort: by start time
        if (a.start !== b.start) {
          return a.start - b.start
        }

        // Secondary sort: by title with natural number ordering
        return this.naturalSort(a.title, b.title)
      })

      // Set end times based on next chapter start times
      for (let i = 0; i < chapters.length; i++) {
        if (i < chapters.length - 1) {
          chapters[i].end = chapters[i + 1].start
        } else {
          chapters[i].end = this.mediaDuration || chapters[i].start + 300
        }
        chapters[i].id = i
      }

      return chapters
    },
    parseJsonChapters(jsonContent) {
      let data
      try {
        data = JSON.parse(jsonContent)
      } catch (error) {
        throw new Error('Invalid JSON format')
      }

      if (!Array.isArray(data)) {
        throw new Error('JSON must be an array of chapter objects')
      }

      const chapters = []
      for (let i = 0; i < data.length; i++) {
        const item = data[i]

        if (!item || typeof item !== 'object') {
          throw new Error(`Chapter ${i + 1} must be an object`)
        }

        if (!item.title || typeof item.title !== 'string') {
          throw new Error(`Chapter ${i + 1} must have a title`)
        }

        if (typeof item.start !== 'number') {
          throw new Error(`Chapter ${i + 1} must have a numeric start time`)
        }

        const chapter = {
          id: i,
          title: item.title.trim(),
          start: Math.max(0, Math.floor(item.start)),
          end: item.end ? Math.floor(item.end) : 0
        }

        chapters.push(chapter)
      } // Sort by start time, then by title (with natural number ordering)
      chapters.sort((a, b) => {
        // Primary sort: by start time
        if (a.start !== b.start) {
          return a.start - b.start
        }

        // Secondary sort: by title with natural number ordering
        return this.naturalSort(a.title, b.title)
      })

      // Set end times if not provided
      for (let i = 0; i < chapters.length; i++) {
        chapters[i].id = i
        if (!chapters[i].end || chapters[i].end <= chapters[i].start) {
          if (i < chapters.length - 1) {
            chapters[i].end = chapters[i + 1].start
          } else {
            chapters[i].end = this.mediaDuration || chapters[i].start + 300
          }
        }
      }
      return chapters
    },
    loadCurrentChaptersAsCue() {
      if (!this.newChapters.length) {
        this.$toast.error('No chapters to convert')
        return
      }

      let cueContent = `REM Generated from Audiobookshelf chapters
FILE "${this.title || 'audiobook'}.wav" WAVE
`

      this.newChapters.forEach((chapter, index) => {
        const trackNumber = (index + 1).toString().padStart(2, '0')
        const startTime = this.secondsToCueTime(chapter.start)

        cueContent += `  TRACK ${trackNumber} AUDIO
    TITLE "${chapter.title || `Chapter ${index + 1}`}"
    INDEX 01 ${startTime}
`
      })

      this.cueFileInput = cueContent
      this.$toast.success('Current chapters loaded as CUE format')
    },
    loadCurrentChaptersAsJson() {
      if (!this.newChapters.length) {
        this.$toast.error('No chapters to convert')
        return
      }

      const chaptersData = this.newChapters.map((chapter) => ({
        title: chapter.title || '',
        start: chapter.start,
        end: chapter.end
      }))

      this.jsonInput = JSON.stringify(chaptersData, null, 2)
      this.$toast.success('Current chapters loaded as JSON format')
    },
    secondsToCueTime(seconds) {
      const totalSeconds = Math.floor(seconds)
      const minutes = Math.floor(totalSeconds / 60)
      const remainingSeconds = totalSeconds % 60
      const frames = Math.floor((seconds - totalSeconds) * 75) // 75 frames per second for CD audio      return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}:${frames.toString().padStart(2, '0')}`
    },
    naturalSort(a, b) {
      // Natural sort that handles numbers within strings correctly
      // e.g., "Chapter 2" comes before "Chapter 10"
      return a.localeCompare(b, undefined, {
        numeric: true,
        sensitivity: 'base'
      })
    },
    toggleRemoveBranding() {
      this.removeBranding = !this.removeBranding
    },
    shiftChapterTimes() {
      if (!this.shiftAmount || isNaN(this.shiftAmount) || this.newChapters.length <= 1) {
        return
      }

      const amount = Number(this.shiftAmount)

      // Check if any unlocked chapters would be affected negatively
      const unlockedChapters = this.newChapters.filter((chap) => !this.lockedChapters.has(chap.id))

      if (unlockedChapters.length === 0) {
        this.$toast.warning(this.$strings.ToastChaptersAllLocked)
        return
      }

      if (unlockedChapters[0].id === 0 && unlockedChapters[0].end + amount <= 0) {
        this.$toast.error(this.$strings.ToastChapterInvalidShiftAmount)
        return
      }

      for (let i = 0; i < this.newChapters.length; i++) {
        const chap = this.newChapters[i]

        // Skip locked chapters
        if (this.lockedChapters.has(chap.id)) {
          continue
        }

        chap.end = Math.min(chap.end + amount, this.mediaDuration)
        if (i > 0) {
          chap.start = Math.max(0, chap.start + amount)
        }
      }
      this.checkChapters()
    },
    incrementChapterTime(chapter, amount) {
      if (chapter.id === 0 && chapter.start + amount < 0) {
        return
      }
      if (chapter.start + amount >= this.mediaDuration) {
        return
      }      chapter.start = Math.max(0, chapter.start + amount)
      this.debouncedCheckChapters()
    },
    adjustChapterStartTime(chapter) {
      const newStartTime = chapter.start + this.elapsedTime
      chapter.start = newStartTime
      this.checkChapters()
      this.$toast.success(this.$strings.ToastChapterStartTimeAdjusted.replace('{0}', this.elapsedTime))

      this.destroyAudioEl()
    },
    startElapsedTimeTracking() {
      this.elapsedTime = 0
      this.playStartTime = Date.now()
      this.elapsedTimeInterval = setInterval(() => {
        this.elapsedTime = Math.floor((Date.now() - this.playStartTime) / 1000)
      }, 100)
    },
    stopElapsedTimeTracking() {
      if (this.elapsedTimeInterval) {
        clearInterval(this.elapsedTimeInterval)
        this.elapsedTimeInterval = null
      }
      this.elapsedTime = 0
      this.playStartTime = null
    },
    toggleChapterLock(chapter, event) {
      const chapterId = chapter.id

      if (event.shiftKey && this.lastSelectedLockIndex !== null) {
        const startIndex = Math.min(this.lastSelectedLockIndex, chapterId)
        const endIndex = Math.max(this.lastSelectedLockIndex, chapterId)
        const shouldLock = !this.lockedChapters.has(chapterId)

        for (let i = startIndex; i <= endIndex; i++) {
          if (shouldLock) {
            this.lockedChapters.add(i)
          } else {
            this.lockedChapters.delete(i)
          }
        }
      } else {
        if (this.lockedChapters.has(chapterId)) {
          this.lockedChapters.delete(chapterId)
        } else {
          this.lockedChapters.add(chapterId)
        }
      }

      this.lastSelectedLockIndex = chapterId
      this.lockedChapters = new Set(this.lockedChapters)
    },
    lockAllChapters() {
      this.newChapters.forEach((chapter) => {
        this.lockedChapters.add(chapter.id)
      })
      this.lockedChapters = new Set(this.lockedChapters)
    },
    unlockAllChapters() {
      this.lockedChapters.clear()
      this.lockedChapters = new Set(this.lockedChapters)
    },
    toggleAllChaptersLock() {
      if (this.allChaptersLocked) {
        this.unlockAllChapters()
      } else {
        this.lockAllChapters()
      }
    },
    editItem() {
      this.$store.commit('showEditModal', this.libraryItem)
    },
    addChapter(chapter) {
      const newChapter = {
        id: chapter.id + 1,
        start: chapter.start,
        end: chapter.end,
        title: ''
      }
      this.newChapters.splice(chapter.id + 1, 0, newChapter)
      this.checkChapters()
    },
    removeChapter(chapter) {
      if (this.lockedChapters.has(chapter.id)) {
        this.$toast.warning(this.$strings.ToastChapterLocked)
        return
      }
      this.newChapters = this.newChapters.filter((ch) => ch.id !== chapter.id)
      this.checkChapters()
    },    checkChapters() {
      let previousStart = 0
      let hasChanges = this.newChapters.length !== this.chapters.length

      // Use a more efficient approach - batch updates
      const updates = []
      
      for (let i = 0; i < this.newChapters.length; i++) {
        const chapter = this.newChapters[i]
        const newId = i
        const newStart = Number(chapter.start)
        const newTitle = (chapter.title || '').trim()
        
        // Only update if values actually changed
        if (chapter.id !== newId) {
          updates.push({ index: i, property: 'id', value: newId })
        }
        if (chapter.start !== newStart) {
          updates.push({ index: i, property: 'start', value: newStart })
        }
        if (chapter.title !== newTitle) {
          updates.push({ index: i, property: 'title', value: newTitle })
        }

        // Error checking with current values
        let error = null
        if (i === 0 && newStart !== 0) {
          error = this.$strings.MessageChapterErrorFirstNotZero
        } else if (newStart <= previousStart && i > 0) {
          error = this.$strings.MessageChapterErrorStartLtPrev
        } else if (newStart >= this.mediaDuration) {
          error = this.$strings.MessageChapterErrorStartGteDuration
        }
        
        if (chapter.error !== error) {
          updates.push({ index: i, property: 'error', value: error })
        }
        
        previousStart = newStart

        // Check for changes only if we haven't already detected them
        if (!hasChanges) {
          const existingChapter = this.chapters[i]
          if (existingChapter) {
            const { start, end, title } = chapter
            if (start !== existingChapter.start || end !== existingChapter.end || title !== existingChapter.title) {
              hasChanges = true
            }
          } else {
            hasChanges = true
          }
        }
      }

      // Apply all updates in batch
      updates.forEach(update => {
        this.newChapters[update.index][update.property] = update.value
      })

      this.hasChanges = hasChanges
    },
    playChapter(chapter) {
      console.log('Play Chapter', chapter.id)
      if (this.selectedChapterId === chapter.id) {
        console.log('Chapter already playing', this.isLoadingChapter, this.isPlayingChapter)
        if (this.isLoadingChapter) return
        if (this.isPlayingChapter) {
          this.destroyAudioEl()
          return
        }
      }
      if (this.selectedChapterId) {
        this.destroyAudioEl()
      }

      const audioTrack = this.tracks.find((at) => {
        return chapter.start >= at.startOffset && chapter.start < at.startOffset + at.duration
      })
      this.selectedChapter = chapter
      this.isLoadingChapter = true

      const trackOffset = chapter.start - audioTrack.startOffset
      this.playTrackAtTime(audioTrack, trackOffset)
    },
    playTrackAtTime(audioTrack, trackOffset) {
      this.currentTrackIndex = audioTrack.index

      const audioEl = this.audioEl || document.createElement('audio')
      var src = audioTrack.contentUrl + `?token=${this.userToken}`

      audioEl.src = `${process.env.serverUrl}${src}`
      audioEl.id = 'chapter-audio'
      document.body.appendChild(audioEl)

      audioEl.addEventListener('loadeddata', () => {
        console.log('Audio loaded data', audioEl.duration)
        audioEl.currentTime = trackOffset
        audioEl.play()
        console.log('Playing audio at current time', trackOffset)
      })
      audioEl.addEventListener('play', () => {
        console.log('Audio playing')
        this.isLoadingChapter = false
        this.isPlayingChapter = true
        this.startElapsedTimeTracking()
      })
      audioEl.addEventListener('ended', () => {
        console.log('Audio ended')
        const nextTrack = this.tracks.find((t) => t.index === this.currentTrackIndex + 1)
        if (nextTrack) {
          console.log('Playing next track', nextTrack.index)
          this.currentTrackIndex = nextTrack.index
          this.playTrackAtTime(nextTrack, 0)
        } else {
          console.log('No next track')
          this.destroyAudioEl()
        }
      })
      this.audioEl = audioEl
    },
    destroyAudioEl() {
      if (!this.audioEl) return
      this.audioEl.remove()
      this.audioEl = null
      this.selectedChapter = null
      this.isPlayingChapter = false
      this.isLoadingChapter = false
      this.stopElapsedTimeTracking()
    },
    saveChapters() {
      this.checkChapters()

      for (let i = 0; i < this.newChapters.length; i++) {
        if (this.newChapters[i].error) {
          this.$toast.error(this.$strings.ToastChaptersHaveErrors)
          return
        }
        if (!this.newChapters[i].title) {
          this.$toast.error(this.$strings.ToastChaptersMustHaveTitles)
          return
        }

        const nextChapter = this.newChapters[i + 1]
        if (nextChapter) {
          this.newChapters[i].end = nextChapter.start
        } else {
          this.newChapters[i].end = this.mediaDuration
        }
      }

      this.saving = true

      const payload = {
        chapters: this.newChapters
      }
      this.$axios
        .$post(`/api/items/${this.libraryItem.id}/chapters`, payload)
        .then((data) => {
          this.saving = false
          if (data.updated) {
            this.$toast.success(this.$strings.ToastChaptersUpdated)
            if (this.previousRoute) {
              this.$router.push(this.previousRoute)
            } else {
              this.$router.push(`/item/${this.libraryItem.id}`)
            }
          } else {
            this.$toast.info(this.$strings.MessageNoUpdatesWereNecessary)
          }
        })
        .catch((error) => {
          this.saving = false
          console.error('Failed to update chapters', error)
          this.$toast.error(this.$strings.ToastFailedToUpdate)
        })
    },
    applyChapterNamesOnly() {
      this.newChapters.forEach((chapter, index) => {
        if (this.chapterData.chapters[index]) {
          chapter.title = this.chapterData.chapters[index].title
        }
      })

      this.showFindChaptersModal = false
      this.chapterData = null

      this.checkChapters()
    },
    applyChapterData() {
      let index = 0
      this.newChapters = this.chapterData.chapters
        .filter((chap) => chap.startOffsetSec < this.mediaDuration)
        .map((chap) => {
          return {
            id: index++,
            start: chap.startOffsetMs / 1000,
            end: Math.min(this.mediaDuration, (chap.startOffsetMs + chap.lengthMs) / 1000),
            title: chap.title
          }
        })
      this.showFindChaptersModal = false
      this.chapterData = null

      this.checkChapters()
    },
    findChapters() {
      if (!this.asinInput) {
        this.$toast.error(this.$strings.ToastAsinRequired)
        return
      }

      // Update local storage region
      if (this.regionInput !== localStorage.getItem('audibleRegion')) {
        localStorage.setItem('audibleRegion', this.regionInput)
      }

      this.findingChapters = true
      this.chapterData = null
      this.asinError = null // used to show warning about audible vs amazon ASIN
      this.$axios
        .$get(`/api/search/chapters?asin=${this.asinInput}&region=${this.regionInput}`)
        .then((data) => {
          this.findingChapters = false

          if (data.error) {
            this.asinError = this.$getString(data.stringKey)
          } else {
            console.log('Chapter data', data)
            this.chapterData = this.removeBranding ? this.removeBrandingFromData(data) : data
          }
        })
        .catch((error) => {
          this.findingChapters = false
          console.error('Failed to get chapter data', error)
          this.$toast.error(this.$strings.ToastFailedToLoadData)
          this.showFindChaptersModal = false
        })
    },
    removeBrandingFromData(data) {
      if (!data) return data
      try {
        const introDuration = data.brandIntroDurationMs
        const outroDuration = data.brandOutroDurationMs

        for (let i = 0; i < data.chapters.length; i++) {
          const chapter = data.chapters[i]
          if (chapter.startOffsetMs < introDuration) {
            // This should never happen, as the intro is not longer than the first chapter
            // If this happens set to the next second
            // Will be 0 for the first chapter anayways
            chapter.startOffsetMs = i * 1000
            chapter.startOffsetSec = i
          } else {
            chapter.startOffsetMs -= introDuration
            chapter.startOffsetSec = Math.floor(chapter.startOffsetMs / 1000)
          }
        }

        const lastChapter = data.chapters[data.chapters.length - 1]
        // If there is an outro that's in the outro duration, remove it
        if (lastChapter && lastChapter.lengthMs <= outroDuration) {
          data.chapters.pop()
        }

        return data
      } catch {
        return data
      }
    },
    resetChapters() {
      const payload = {
        message: this.$strings.MessageResetChaptersConfirm,
        callback: (confirmed) => {
          if (confirmed) {
            this.initChapters()
          }
        },
        type: 'yesNo'
      }
      this.$store.commit('globals/setConfirmPrompt', payload)
    },
    initChapters() {
      this.newChapters = this.chapters.map((c) => ({ ...c }))
      if (!this.newChapters.length) {
        this.newChapters = [
          {
            id: 0,
            start: 0,
            end: this.mediaDuration,
            title: ''
          }
        ]
      }
      this.checkChapters()
    },
    removeAllChaptersClick() {
      const payload = {
        message: this.$strings.MessageConfirmRemoveAllChapters,
        callback: (confirmed) => {
          if (confirmed) {
            this.removeAllChapters()
          }
        },
        type: 'yesNo'
      }
      this.$store.commit('globals/setConfirmPrompt', payload)
    },
    removeAllChapters() {
      this.saving = true
      const payload = {
        chapters: []
      }
      this.$axios
        .$post(`/api/items/${this.libraryItem.id}/chapters`, payload)
        .then((data) => {
          if (data.updated) {
            this.$toast.success(this.$strings.ToastChaptersRemoved)
            if (this.previousRoute) {
              this.$router.push(this.previousRoute)
            } else {
              this.$router.push(`/item/${this.libraryItem.id}`)
            }
          } else {
            this.$toast.info(this.$strings.MessageNoUpdatesWereNecessary)
          }
        })
        .catch((error) => {
          console.error('Failed to remove chapters', error)
          this.$toast.error(this.$strings.ToastRemoveFailed)
        })
        .finally(() => {
          this.saving = false
        })
    },
    handleBulkChapterAdd() {
      const input = this.bulkChapterInput.trim()
      if (!input) return

      const numberMatch = input.match(/(\d+)/)

      if (numberMatch) {
        // Extract the base pattern and number, preserving zero-padding
        const originalNumberString = numberMatch[1]
        const foundNumber = parseInt(originalNumberString)
        const numberIndex = numberMatch.index
        const beforeNumber = input.substring(0, numberIndex)
        const afterNumber = input.substring(numberIndex + originalNumberString.length)

        this.detectedPattern = {
          before: beforeNumber,
          after: afterNumber,
          startingNumber: foundNumber,
          originalPadding: originalNumberString.length,
          hasLeadingZeros: originalNumberString.length > 1 && originalNumberString.startsWith('0')
        }

        this.bulkChapterCount = 1
        this.showBulkChapterModal = true
      } else {
        this.addSingleChapterFromInput(input)
      }
    },
    addSingleChapterFromInput(title) {
      // Find the last chapter to determine where to add the new one
      const lastChapter = this.newChapters[this.newChapters.length - 1]
      const newStart = lastChapter ? lastChapter.end : 0
      const newEnd = Math.min(newStart + 300, this.mediaDuration)

      const newChapter = {
        id: this.newChapters.length,
        start: newStart,
        end: newEnd,
        title: title
      }

      this.newChapters.push(newChapter)
      this.bulkChapterInput = ''
      this.checkChapters()
    },

    addBulkChapters() {
      const count = parseInt(this.bulkChapterCount)
      if (!count || count < 1 || count > 150) {
        this.$toast.error(this.$strings.ToastBulkChapterInvalidCount)
        return
      }

      const { before, after, startingNumber, originalPadding, hasLeadingZeros } = this.detectedPattern
      const lastChapter = this.newChapters[this.newChapters.length - 1]
      const baseStart = lastChapter ? lastChapter.start + 1 : 0

      // Add multiple chapters with the detected pattern
      for (let i = 0; i < count; i++) {
        const chapterNumber = startingNumber + i
        let formattedNumber = chapterNumber.toString()

        // Apply zero-padding if the original had leading zeros
        if (hasLeadingZeros && originalPadding > 1) {
          formattedNumber = chapterNumber.toString().padStart(originalPadding, '0')
        }

        const newStart = baseStart + i
        const newEnd = Math.min(newStart + i + i, this.mediaDuration)

        const newChapter = {
          id: this.newChapters.length,
          start: newStart,
          end: newEnd,
          title: `${before}${formattedNumber}${after}`
        }

        this.newChapters.push(newChapter)
      }

      this.bulkChapterInput = ''
      this.showBulkChapterModal = false
      this.detectedPattern = null
      this.checkChapters()
    },
    libraryItemUpdated(libraryItem) {
      if (libraryItem.id === this.libraryItem.id) {
        if (!!libraryItem.media.metadata.asin && this.mediaMetadata.asin !== libraryItem.media.metadata.asin) {
          this.asinInput = libraryItem.media.metadata.asin
        }
        this.libraryItem = libraryItem
      }
    }
  },
  mounted() {
    this.regionInput = localStorage.getItem('audibleRegion') || 'US'
    this.asinInput = this.mediaMetadata.asin || null
    this.initChapters()

    this.$eventBus.$on(`${this.libraryItem.id}_updated`, this.libraryItemUpdated)
  },  beforeDestroy() {
    this.destroyAudioEl()
    
    // Cancel any pending debounced calls
    if (this.checkChaptersTimeout) {
      clearTimeout(this.checkChaptersTimeout)
      this.checkChaptersTimeout = null
    }

    this.$eventBus.$off(`${this.libraryItem.id}_updated`, this.libraryItemUpdated)
  }
}
</script>
