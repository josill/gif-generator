<template>
  <div class="flex flex-col gap-y-8 h-screen w-screen justify-center items-center bg-neutral-900 font-montserrat">
    <div class="flex flex-col gap-y-8 items-center justify-center w-1/2" v-if="!videoSrc">
      <h1 class="text-xl text-white">Animate a random GIF from your video</h1>
      <label for="dropzone-file"
        class="flex flex-col items-center justify-center w-full h-64 border-2 border-gray-300 border-dashed rounded-lg cursor-pointer dark:hover:bg-bray-800 hover:bg-gray-100 dark:border-gray-600 dark:hover:border-gray-500 dark:hover:bg-gray-600">
        <div class="flex flex-col items-center justify-center pt-5 pb-6 gap-y-4">
          <Icon icon="ic:outline-file-upload" class="text-gray-500 dark:text-gray-400 text-4xl" />
          <p class="mb-2 text-sm text-gray-500 dark:text-gray-400"><span class="font-semibold">Choose a video to
              upload</span> or drag and drop</p>
        </div>
        <input ref="fileInput" id="dropzone-file" type="file" class="hidden" @change="handleFileUpload"
          accept="video/*" />
      </label>
    </div>
    <div class="flex justify-center items-center gap-x-2" v-else>
      <video 
      id="video-player"
      controls 
      width="640" 
      :src="videoSrc"></video>
    </div>
    <div class="flex flex-col gap-y-8 justify-center items-center w-1/2" v-if="videoSrc && ready">
      <button
        class="border border-dashed rounded-lg w-1/2 h-[50px] text-gray-500 dark:text-gray-400 dark:hover:bg-bray-800 hover:bg-gray-100 dark:border-gray-600 dark:hover:border-gray-500 dark:hover:bg-gray-600"
        @click="createGif()"
        v-if="!loading && !gifSrc">Convert</button>
        <Icon icon="line-md:loading-twotone-loop" class="text-4xl" v-if="loading" />
      <h2 class="text-red-700 text-lg" v-if="error">Midagi läks valesti</h2>
      <div class="flex flex-col gap-y-8" v-if="gifSrc">
        <img
        :src="gifSrc" 
        width="480" 
        height="480" 
        alt="Generated GIF" />
        <div class="flex items-center justify-around">
          <button class="border border-dashed rounded-lg w-1/3 h-[50px] text-gray-500 dark:text-gray-400 dark:hover:bg-bray-800 hover:bg-gray-100 dark:border-gray-600 dark:hover:border-gray-500 dark:hover:bg-gray-600" 
            @click="resetPage">Create another GIF</button>
          <button class="border border-dashed rounded-lg w-1/3 h-[50px] text-gray-500 dark:text-gray-400 dark:hover:bg-bray-800 hover:bg-gray-100 dark:border-gray-600 dark:hover:border-gray-500 dark:hover:bg-gray-600" 
            @click="downloadGif">Download GIF</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import { Icon } from '@iconify/vue';
import { createFFmpeg, fetchFile } from '@ffmpeg/ffmpeg';


export default {
  name: 'HomeView',
  components: {
    Icon,
  },
  setup() {
    let file = ''
    const fileInput = ref(null);
    const videoSrc = ref('');
    const ready = ref(false);
    const gifSrc = ref('');
    const loading = ref(false);
    const error = ref(null);
    const ffmpeg = createFFmpeg({log: true});

    onMounted(async () => {
      await ffmpeg.load();
      ready.value = true;
    });

    const handleFileUpload = () => {
      file = fileInput.value.files[0];
      const videoURL = URL.createObjectURL(file);
      videoSrc.value = videoURL;
    };

    const downloadGif = () => {
      const link = document.createElement('a');
      link.href = gifSrc.value;
      link.download = 'generated_gif.gif';
      link.click();
    };

    const resetPage = () => {
      // Reset variables to their initial state
      videoSrc.value = '';
      gifSrc.value = '';
      error.value = null;
    };

    const createGif = async () => {
      try {
        loading.value = true;
        // Write the file to memory
        ffmpeg.FS('writeFile', 'input.mp4', await fetchFile(file));

       // Create a new video element to get the video duration
        const videoElement = document.createElement('video');
        videoElement.src = URL.createObjectURL(file);

        // Wait for the video to load its metadata (including duration)
        await new Promise((resolve) => {
          videoElement.addEventListener('loadedmetadata', resolve);
        });

        const videoDuration = videoElement.duration;
        const randomStartTime = Math.floor(Math.random() * videoDuration);

        // Run the FFMpeg command
        await ffmpeg.run('-i', 'input.mp4', '-t', '5.0', '-ss', randomStartTime.toString(), '-f', 'gif', 'out.gif');

        // Read the result
        const data = ffmpeg.FS('readFile', 'out.gif');

        // Create a URL
        const url = URL.createObjectURL(new Blob([data.buffer]), { type: 'image/gif' });
        gifSrc.value = url;;
        
        // Cleanup the video element after use
        videoElement.src = '';
        URL.revokeObjectURL(videoElement.src);
      } catch (e) {
        console.log(e)
        error.value = true;
      } finally {
        loading.value = false;
      }
    }

    return {
      fileInput,
      videoSrc,
      gifSrc,
      ready,
      error,
      loading,
      handleFileUpload,
      downloadGif,
      createGif,
      resetPage,
    };
  }
};
</script>