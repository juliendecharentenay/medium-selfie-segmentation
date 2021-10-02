<template>
  <div id="app">
    <div class="absolute top-0 bottom-0 left-0 right-0 bg-blue-400">
      <div class="absolute top-0 left-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute top-1 left-1">Webcam</div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-in"
        ></canvas>
      </div>
      <div class="absolute top-0 right-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute top-1 right-1">Mask</div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-mask"
        ></canvas>
      </div>
      <div class="absolute bottom-0 left-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute bottom-1 left-1">Background</div>
        <canvas
          class="object-none object-center bg-gray-100 border-gray-800"
          id="canvas-background"
        ></canvas>
      </div>
      <div class="absolute bottom-0 right-0 h-1/2 w-1/2 border-gray-800">
        <div class="absolute bottom-1 right-1">Foreground</div>
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

export default {
  name: "App",
  components: {},
  data: function () {
    return {
      error: null,
      webcam_video: null,
      selfie_segmentation: null,
      selfie_segmentation_ready: false,
    };
  },
  mounted: function () {
    this.resize();
    window.onresize = () => {
      this.resize();
    };
    this.init();
  },
  methods: {
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
      navigator.mediaDevices
        .getUserMedia({ audio: false, video: { facingMode: "user" } })
        .then((media_stream) => {
          this.webcam_video = document.createElement("video");
          this.webcam_video.srcObject = media_stream;
          this.webcam_video.style.display = "none";
          document.body.appendChild(this.webcam_video);
          this.webcam_video.onplay = this.playing;
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
