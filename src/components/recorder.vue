<style lang="scss">
  .ar {
    width: 420px;
    font-family: 'Roboto', sans-serif;
    border-radius: 16px;
    background-color: #FAFAFA;
    box-shadow: 0 4px 18px 0 rgba(0,0,0,0.17);
    position: relative;
    box-sizing: content-box;

    &-content {
      padding: 16px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }

    &-records {
      height: 138px;
      padding-top: 1px;
      overflow-y: auto;
      margin-bottom: 20px;

      &__record {
        width: 320px;
        height: 45px;
        padding: 0 10px;
        margin: 0 auto;
        line-height: 45px;
        display: flex;
        justify-content: space-between;
        border-bottom: 1px solid #E8E8E8;
        position: relative;

        &--selected {
          border: 1px solid #E8E8E8;
          border-radius: 24px;
          background-color: #FFFFFF;
          margin-top: -1px;
          padding: 0 34px;
        }
      }
    }

    &-recorder {
      position: relative;
      display: flex;
      flex-direction: column;
      align-items: center;

      &__duration {
        color: #AEAEAE;
        font-size: 32px;
        font-weight: 500;
        margin-top: 20px;
        margin-bottom: 16px;
      }

      &__stop {
        position: absolute;
        top: 10px;
        right: -52px;
      }

      &__time-limit {
        position: absolute;
        color: #AEAEAE;
        font-size: 12px;
        top: 128px;
      }

      &__records-limit {
        position: absolute;
        color: #AEAEAE;
        font-size: 13px;
        top: 78px;
      }
    }

    &-spinner {
      display: flex;
      height: 30px;
      position: absolute;
      left: 0;
      right: 0;
      top: 0;
      bottom: 0;
      margin: auto;
      width: 144px;
      z-index: 10;

      &__dot {
        display: block;
        margin: 0 8px;
        border-radius: 50%;
        width: 30px;
        height: 30px;
        background: #05CBCD;
        animation-name: blink;
        animation-duration: 1.4s;
        animation-iteration-count: infinite;
        animation-fill-mode: both;

        &:nth-child(2) { animation-delay: .2s; }

        &:nth-child(3) { animation-delay: .4s; }

        @keyframes blink {
          0%    { opacity: .2; }
          20%   { opacity: 1;  }
          100%  { opacity: .2; }
        }
      }
    }

    &__text {
      color: rgba(84,84,84,0.5);
      font-size: 16px;
    }

    &__blur {
      filter: blur(2px);
      opacity: 0.7;
    }

    &__overlay {
      position: absolute;
      width: 100%;
      height: 100%;
      z-index: 10;
    }

    &__upload-status {
      text-align: center;
      font-size: 10px;
      padding: 2px;
      letter-spacing: 1px;
      position: absolute;
      bottom: 0;

      &--success {
        color: green;
      }

      &--fail {
        color: red;
      }
    }

    &__rm {
      cursor: pointer;
      position: absolute;
      width: 6px;
      height: 6px;
      padding: 6px;
      line-height: 6px;
      margin: auto;
      left: 10px;
      bottom: 0;
      top: 0;
      color: rgb(244, 120, 90);
    }

    &__downloader,
    &__uploader {
      position: absolute;
      top: 0;
      bottom: 0;
      margin: auto;
    }

    &__downloader {
      right: 115px;
    }

    &__uploader {
      right: 85px;
    }
  }

  @import '../scss/icons';
</style>

<template>
  <div class="ar">
    <div class="ar__overlay" v-if="isUploading"></div>
    <div class="ar-spinner" v-if="isUploading">
      <div class="ar-spinner__dot"></div>
      <div class="ar-spinner__dot"></div>
      <div class="ar-spinner__dot"></div>
    </div>

    <div class="ar-content" :class="{'ar__blur': isUploading}">
      <div class="ar-recorder">
        <icon-button
          class="ar-icon ar-icon__lg"
          :name="iconButtonType"
          :class="{
            'ar-icon--rec': isRecording,
            'ar-icon--pulse': isRecording && volume > 0.02
          }"
          @click.native="toggleRecorder"/>
        <icon-button
          class="ar-icon ar-icon__sm ar-recorder__stop"
          name="stop"
          @click.native="stopRecorder"/>
      </div>

      <div class="ar-recorder__records-limit" v-if="attempts">Attempts: {{attemptsLeft}}/{{attempts}}</div>
      <div class="ar-recorder__duration">{{recordedTime}}</div>
      <div class="ar-recorder__time-limit" v-if="time">Record duration is limited: {{time}}m</div>

      <div class="ar-records">
        <div
          class="ar-records__record"
          :class="{'ar-records__record--selected': record.id === selected.id}"
          :key="record.id"
          v-for="(record, idx) in recordList"
          @click="choiceRecord(record)">
            <div
              class="ar__rm"
              v-if="record.id === selected.id"
              @click="removeRecord(idx)">&times;</div>
            <div class="ar__text">Record {{idx + 1}}</div>
            <div class="ar__text">{{record.duration}}</div>

            <downloader
              v-if="record.id === selected.id && showDownloadButton"
              class="ar__downloader"
              :record="record"
              :filename="filename"/>

            <uploader
              v-if="record.id === selected.id && showUploadButton"
              class="ar__uploader"
              :record="record"
              :filename="filename"
              :headers="headers"
              :upload-url="uploadUrl"/>
        </div>
      </div>

      <audio-player :record="selected"/>
    </div>
  </div>
</template>


<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue';
import AudioPlayer from './player';
import Downloader from './downloader';
import IconButton from './icon-button';
import Recorder from '@/lib/recorder';
import Uploader from './uploader';
import UploaderPropsMixin from '@/mixins/uploader-props';
import { convertTimeMMSS } from '@/lib/utils';

// Props
const props = defineProps({
  attempts: Number,
  time: Number,
  bitRate: { type: Number, default: 128 },
  sampleRate: { type: Number, default: 44100 },
  showDownloadButton: { type: Boolean, default: true },
  showUploadButton: { type: Boolean, default: true },
  micFailed: Function,
  beforeRecording: Function,
  pauseRecording: Function,
  afterRecording: Function,
  failedUpload: Function,
  beforeUpload: Function,
  successfulUpload: Function,
  selectRecord: Function,
});

// Reactive data
const isUploading = ref(false);
const recorder = ref(_initRecorder());
const recordList = ref([]);
const selected = ref({});
const uploadStatus = ref(null);

// Computed properties
const attemptsLeft = computed(() => props.attempts - recordList.value.length);
const iconButtonType = computed(() =>
  recorder.value.isRecording && recorder.value.isPause ? 'mic' : recorder.value.isRecording ? 'pause' : 'mic'
);
const isPause = computed(() => recorder.value.isPause);
const isRecording = computed(() => recorder.value.isRecording);
const recordedTime = computed(() => {
  if (props.time && recorder.value.duration >= props.time * 60) {
    stopRecorder();
  }
  return convertTimeMMSS(recorder.value.duration);
});
const volume = computed(() => parseFloat(recorder.value.volume));

// Lifecycle hooks
onMounted(() => {
  // Replace $eventBus with your event bus approach or custom events
});

onBeforeUnmount(() => {
  stopRecorder();
});

// Methods
function toggleRecorder() {
  if (props.attempts && recorder.value.records.length >= props.attempts) {
    return;
  }
  if (!isRecording.value || (isRecording.value && isPause.value)) {
    recorder.value.start();
  } else {
    recorder.value.pause();
  }
}

function stopRecorder() {
  if (!isRecording.value) {
    return;
  }
  recorder.value.stop();
  recordList.value = recorder.value.recordList();
}

function removeRecord(idx) {
  recordList.value.splice(idx, 1);
  selected.value.url = null;
  // Emit custom event for record removal
}

function choiceRecord(record) {
  if (selected.value === record) {
    return;
  }
  selected.value = record;
  props.selectRecord && props.selectRecord(record);
}

function _initRecorder() {
  return new Recorder({
    beforeRecording: props.beforeRecording,
    afterRecording: props.afterRecording,
    pauseRecording: props.pauseRecording,
    micFailed: props.micFailed,
    bitRate: props.bitRate,
    sampleRate: props.sampleRate,
    format: props.format,
  });
}
</script>


