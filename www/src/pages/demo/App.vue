<template>
  <div id="app">
    <div class="absolute top-0 bottom-0 left-0 right-0 bg-blue-400">
      <div class="absolute top-0 left-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute top-1 left-1">Webcam</div>
        <div class="absolute bottom-2 left-1">
          <RecordButton class="h-10 w-10" @click="record('canvas-in');" v-if="! recording" />
          <StopButton class="h-10 w-10" @click="stop();" v-else />
        </div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-in"
        ></canvas>
      </div>
      <div class="absolute top-0 right-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute top-1 right-1">Mask</div>
        <div class="absolute bottom-2 left-1">
          <RecordButton class="h-10 w-10" @click="record('canvas-mask');" v-if="! recording" />
          <StopButton class="h-10 w-10" @click="stop();" v-else />
        </div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-mask"
        ></canvas>
      </div>
      <div class="absolute bottom-0 left-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute bottom-1 left-1">Background</div>
        <div class="absolute bottom-2 left-1">
          <RecordButton class="h-10 w-10" @click="record('canvas-background');" v-if="! recording" />
          <StopButton class="h-10 w-10" @click="stop();" v-else />
        </div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-background"
        ></canvas>
      </div>
      <div class="absolute bottom-0 right-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute bottom-1 right-1">Foreground</div>
        <div class="absolute bottom-2 left-1">
          <RecordButton class="h-10 w-10" @click="record('canvas-foreground');" v-if="! recording" />
          <StopButton class="h-10 w-10" @click="stop();" v-else />
        </div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-foreground"
        ></canvas>
      </div>
    </div>
  </div>
</template>

<script>
import { SelfieSegmentation } from "@mediapipe/selfie_segmentation";
import RecordButton from "@/components/recordbutton";
import StopButton from "@/components/stopbutton";

export default {
  name: "App",
  components: {
    RecordButton,
    StopButton
  },
  data: function () {
    return {
      error: null,
      webcam_video: null,
      selfie_segmentation: null,
      selfie_segmentation_ready: false,
      media_recorder: null,
      audio_track: null
    };
  },
  mounted: function () {
    this.resize();
    window.onresize = () => {
      this.resize();
    };
    this.init();
  },
  computed: {
    recording: function() { return this.media_recorder !== null; }
  },
  methods: {
    record: function(id) {
      console.log("Record canvas", id);
      const canvas = document.getElementById(id);
      if (canvas === null) { alert("Incorrect element id provided: ", id); return; }

      var chunks = [];
      var canvas_stream = canvas.captureStream(30); // fps

      // Add audio track
      if (this.audio_track) {canvas_stream.addTrack(this.audio_track);}

      // Create media recorder from canvas stream
      this.media_recorder = new MediaRecorder(canvas_stream, { mimeType: "video/webm; codecs=vp9" });

      // Record data in chunks array when data is available
      this.media_recorder.ondataavailable = (evt) => { chunks.push(evt.data); };

      // Provide recorded data when recording stops
      this.media_recorder.onstop = () => {this.on_media_recorder_stop(chunks);}

      // Start recording using a 1s timeslice [ie data is made available every 1s)
      this.media_recorder.start(1000);

    },
    stop: function() {
      if (this.media_recorder !== null) { this.media_recorder.stop(); }
    },
    on_media_recorder_stop: function(chunks) {
      this.media_recorder = null;

      // Gather chunks of video data into a blob and create an object URL
      var blob = new Blob(chunks, {type: "video/webm" });
      const recording_url = URL.createObjectURL(blob);

      // Attach the object URL to an <a> element, setting the download file name
      const a = document.createElement('a');
      a.style = "display: none;";
      a.href = recording_url;
      a.download = "video.webm";
      document.body.appendChild(a);

      // Trigger the file download
      a.click();

      setTimeout(() => {
        // Clean up - see https://stackoverflow.com/a/48968694 for why it is in a timeout
        URL.revokeObjectURL(recording_url);
        document.body.removeChild(a);
      }, 0);
    },
    canvas_in: function () {
      return document.getElementById("canvas-in");
    },
    canvas_mask: function () {
      return document.getElementById("canvas-mask");
    },
    canvas_background: function () {
      return document.getElementById("canvas-background");
    },
    canvas_foreground: function () {
      return document.getElementById("canvas-foreground");
    },

    /*
     * Resize canvas to match parent size
     */
    resize: function () {
      console.log("resize");
      [
        this.canvas_in(),
        this.canvas_mask(),
        this.canvas_background(),
        this.canvas_foreground(),
      ].forEach((c) => {
        c.width = c.parentNode.clientWidth;
        c.height = c.parentNode.clientHeight;
      });
    },

    /*
     * Initialisation
     */
    init: function () {
      this.init_webcam();
      this.init_selfie_segmentation();
    },

    /*
     * Initialize the webcam
     */
    init_webcam: function () {
      // Request webcam with audio and user facing camera
      navigator.mediaDevices
        .getUserMedia({ audio: true, video: { facingMode: "user" } })
        .then((media_stream) => {
          // Retrieve audio track
          this.audio_track = media_stream.getAudioTracks()[0];

          // Assign media stream to video element - with audio muted
          this.webcam_video = document.createElement("video");
          this.webcam_video.srcObject = media_stream;
          this.webcam_video.muted = true;
          this.webcam_video.style.display = "none";
          document.body.appendChild(this.webcam_video);

          this.webcam_video.onplay = this.playing;

          // And start playing
          this.webcam_video.play();
        })
        .catch((e) => {
          this.on_error("Unable to start webcam", e);
        });
    },

    /*
     * Playing
     */
    playing: async function () {
      if (this.selfie_segmentation_ready) {
        await this.selfie_segmentation.send({ image: this.webcam_video });
      }
      window.requestAnimationFrame(this.playing);
    },

    /*
     * Initialize Selfie Segmentation
     */
    init_selfie_segmentation: function () {
      this.selfie_segmentation = new SelfieSegmentation({
        locateFile: (file) => {
          return `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`;
        },
      });
      this.selfie_segmentation.setOptions({ modelSelection: 1 });
      this.selfie_segmentation.onResults(this.on_results);
      this.selfie_segmentation_ready = true;
    },

    /*
     * handle results
     */
    on_results: function (results) {
      {
        // Draw the original image
        const c = this.canvas_in();
        const ctx = c.getContext("2d");
        ctx.clearRect(0, 0, c.width, c.height);
        ctx.drawImage(results.image, 0, 0, c.width, c.height);
      }
      {
        // Draw the segmentation mask
        const c = this.canvas_mask();
        const ctx = c.getContext("2d");
        ctx.clearRect(0, 0, c.width, c.height);
        ctx.drawImage(results.segmentationMask, 0, 0, c.width, c.height);
      }
      {
        // Draw the background
        const c = this.canvas_background();
        const ctx = c.getContext("2d");
        ctx.save();
        ctx.clearRect(0, 0, c.width, c.height);
        ctx.drawImage(results.image, 0, 0, c.width, c.height);
        ctx.globalCompositeOperation = "destination-out";
        ctx.drawImage(results.segmentationMask, 0, 0, c.width, c.height);
        ctx.restore();
      }
      {
        // Draw the foreground
        const c = this.canvas_foreground();
        const ctx = c.getContext("2d");
        ctx.save();
        ctx.clearRect(0, 0, c.width, c.height);
        ctx.drawImage(results.image, 0, 0, c.width, c.height);
        ctx.globalCompositeOperation = "destination-in";
        ctx.drawImage(results.segmentationMask, 0, 0, c.width, c.height);
        ctx.restore();
      }
    },

    /*
     * on_error: handle error message
     */
    on_error: function (msg, e) {
      console.log(msg, e);
      this.error = msg;
    },
  },
};
</script>
