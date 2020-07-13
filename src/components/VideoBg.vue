<template>
  <div>
    <div class="flex justify-center mb-10">
      <video hidden id="video" ref="video" playsinline autoplay></video>
      <canvas id="output" ref="output" />
    </div>
    <div v-if="loading" class="loader"></div>
    <div class="flex justify-center">
      <button
        @click="start()"
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
      >
        Start
      </button>
      <button
        @click="chageMode('blur')"
        class="bg-transparent hover:bg-blue-500 text-blue-700 font-semibold hover:text-white py-2 px-4 border border-blue-500 hover:border-transparent rounded mr-4 ml-4"
      >
        Blur
      </button>
      <button
        @click="chageMode('mask')"
        class="bg-transparent hover:bg-blue-500 text-blue-700 font-semibold hover:text-white py-2 px-4 border border-blue-500 hover:border-transparent rounded"
      >
        Mask
      </button>
    </div>
    <Footer />
  </div>
</template>

<script>
import * as bodyPix from "@tensorflow-models/body-pix";
import "@tensorflow/tfjs-backend-webgl";
import * as dat from "dat.gui";
import Footer from "./Footer";

export default {
  name: "LiveBg",
  components: {
    Footer
  },
  data() {
    return {
      net: {},
      video: {},
      loading: null,
      mode: "blur",
      params: {
        flipHorizontal: false
      },
      paramsBlur: {
        backgroundBlurAmount: 3,
        edgeBlurAmount: 3
      },
      paramsMask: {
        foregroundColor: { r: 0, g: 0, b: 0, a: 0 },
        backgroundColor: { r: 80, g: 255, b: 255, a: 255 },
        drawContour: false,
        opacity: 0.8,
        maskBlurAmount: 0.5,
        maskForeground: false,
        maskBackground: true
      },
      gui: null
    };
  },
  mounted() {
    this.setupGui();
  },
  beforeDestroy() {
    this.gui.destroy();
  },
  methods: {
    setupGui() {
      this.gui = new dat.GUI({ width: 340 });
      this.gui
        .add(this.params, "flipHorizontal")
        .onChange(
          value => (this.params = { ...this.params, flipHorizontal: value })
        );
      let blur = this.gui.addFolder("Blur");
      blur.add(this.paramsBlur, "backgroundBlurAmount", 0, 10).onChange(
        value =>
          (this.paramsBlur = {
            ...this.paramsBlur,
            backgroundBlurAmount: value
          })
      );
      blur
        .add(this.paramsBlur, "edgeBlurAmount", 0, 10)
        .onChange(
          value =>
            (this.paramsBlur = { ...this.paramsBlur, edgeBlurAmount: value })
        );
      let mask = this.gui.addFolder("Mask");
      mask
        .add(this.paramsMask, "opacity", 0.0, 1.0)
        .onChange(
          value => (this.paramsMask = { ...this.paramsMask, opacity: value })
        );
      mask
        .add(this.paramsMask, "maskBlurAmount", 0.0, 1.0)
        .onChange(
          value =>
            (this.paramsMask = { ...this.paramsMask, maskBlurAmount: value })
        );
      mask
        .addColor(this.paramsMask, "foregroundColor", {
          r: 0,
          g: 0,
          b: 0,
          a: 0
        })
        .onChange(
          value =>
            (this.paramsMask = { ...this.paramsMask, foregroundColor: value })
        );
      mask
        .addColor(this.paramsMask, "backgroundColor", {
          r: 80,
          g: 255,
          b: 255,
          a: 255
        })
        .onChange(
          value =>
            (this.paramsMask = { ...this.paramsMask, backgroundColor: value })
        );

      mask.add(this.paramsMask, "maskForeground").onChange(value =>
        value
          ? (this.paramsMask = {
              ...this.paramsMask,
              ...{
                foregroundColor: {
                  ...this.paramsMask.foregroundColor,
                  a: 255
                }
              }
            })
          : (this.paramsMask = {
              ...this.paramsMask,
              ...{
                foregroundColor: {
                  ...this.paramsMask.foregroundColor,
                  a: 0
                }
              }
            })
      );
      mask.add(this.paramsMask, "maskBackground").onChange(value =>
        value
          ? (this.paramsMask = {
              ...this.paramsMask,
              ...{
                backgroundColor: {
                  ...this.paramsMask.backgroundColor,
                  a: 255
                }
              }
            })
          : (this.paramsMask = {
              ...this.paramsMask,
              ...{
                backgroundColor: {
                  ...this.paramsMask.backgroundColor,
                  a: 0
                }
              }
            })
      );
    },
    chageMode(mode) {
      this.mode = mode;
    },
    async start() {
      this.stopVideo();
      this.loading = true;
      await this.loadBodyPix();
      await this.loadVideo();
      this.loading = false;
      this.segmentBodyInRealTime();
    },
    async loadBodyPix() {
      this.net = await bodyPix.load();
    },
    async loadVideo() {
      this.video = await this.setupCamera();
      this.video.play();
    },
    async setupCamera() {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        throw new Error("getUserMedia is not supported");
      }

      const videoElement = this.$refs.video;
      this.stopVideo();

      const stream = await navigator.mediaDevices.getUserMedia({
        audio: false,
        video: {
          width: 480,
          height: 360
        }
      });

      videoElement.srcObject = stream;
      return new Promise(resolve => {
        videoElement.onloadedmetadata = () => {
          let width = videoElement.videoWidth;
          let height = videoElement.videoHeight;
          videoElement.width = width;
          videoElement.height = height;
          resolve(videoElement);
        };
      });
    },
    stopVideo() {
      if (this.video && this.video.srcObject) {
        this.video.srcObject.getTracks().forEach(track => {
          track.stop();
        });
        this.video.srcObject = null;
      }
    },
    segmentBodyInRealTime() {
      const canvas = this.$refs.output;
      const _this = this;

      async function updateFrame() {
        try {
          if (_this.mode === "blur") {
            const segmentation = await _this.net.segmentPerson(_this.video);
            const backgroundBlurAmount = _this.paramsBlur.backgroundBlurAmount;
            const edgeBlurAmount = _this.paramsBlur.edgeBlurAmount;
            const flipHorizontal = _this.params.flipHorizontal;
            bodyPix.drawBokehEffect(
              canvas,
              _this.video,
              segmentation,
              backgroundBlurAmount,
              edgeBlurAmount,
              flipHorizontal
            );
          } else if (_this.mode === "mask") {
            const segmentation = await _this.estimateSegmentation();
            const backgroundDarkeningMask = _this.toMask(segmentation);
            _this.drawMask(canvas, _this.video, backgroundDarkeningMask);
          }
        } catch (error) {
          // eslint-disable-next-line no-console
          console.error(error);
        } finally {
          requestAnimationFrame(updateFrame);
        }
      }
      updateFrame();
    },
    toMask(personSegmentation) {
      const foregroundColor = this.paramsMask.foregroundColor;
      const backgroundColor = this.paramsMask.backgroundColor;
      const drawContour = this.paramsMask.drawContour;
      const backgroundDarkeningMask = bodyPix.toMask(
        personSegmentation,
        foregroundColor,
        backgroundColor,
        drawContour
      );
      return backgroundDarkeningMask;
    },
    drawMask(canvas, video, backgroundDarkeningMask) {
      const opacity = this.paramsMask.opacity;
      const maskBlurAmount = this.paramsMask.maskBlurAmount;
      const flipHorizontal = this.params.flipHorizontal;
      bodyPix.drawMask(
        canvas,
        video,
        backgroundDarkeningMask,
        opacity,
        maskBlurAmount,
        flipHorizontal
      );
    },
    async estimateSegmentation() {
      return await this.net.segmentPerson(this.video, {
        internalResolution: "medium",
        segmentationThreshold: 0.7,
        maxDetections: 5,
        scoreThreshold: 0.2,
        nmsRadius: 20,
        numKeypointForMatching: 17,
        refineSteps: 10
      });
    }
  }
};
</script>

<style scoped>
.loader {
  margin: 100px auto;
  font-size: 16px;
  width: 1em;
  height: 1em;
  border-radius: 50%;
  position: relative;
  text-indent: -9999em;
  -webkit-animation: load5 1.1s infinite ease;
  animation: load5 1.1s infinite ease;
  -webkit-transform: translateZ(0);
  -ms-transform: translateZ(0);
  transform: translateZ(0);
}
@-webkit-keyframes load5 {
  0%,
  100% {
    box-shadow: 0em -2.6em 0em 0em #2375e1,
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.5),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.7);
  }
  12.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.7),
      1.8em -1.8em 0 0em #2375e1, 2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.5);
  }
  25% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.5),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.7), 2.5em 0em 0 0em #2375e1,
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  37.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.5),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.7), 1.75em 1.75em 0 0em #2375e1,
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  50% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.5),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.7), 0em 2.5em 0 0em #2375e1,
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  62.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.5),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.7), -1.8em 1.8em 0 0em #2375e1,
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  75% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.5),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.7), -2.6em 0em 0 0em #2375e1,
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  87.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.5),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.7), -1.8em -1.8em 0 0em #2375e1;
  }
}
@keyframes load5 {
  0%,
  100% {
    box-shadow: 0em -2.6em 0em 0em #2375e1,
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.5),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.7);
  }
  12.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.7),
      1.8em -1.8em 0 0em #2375e1, 2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.5);
  }
  25% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.5),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.7), 2.5em 0em 0 0em #2375e1,
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  37.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.5),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.7), 1.75em 1.75em 0 0em #2375e1,
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  50% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.5),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.7), 0em 2.5em 0 0em #2375e1,
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.2),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  62.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.5),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.7), -1.8em 1.8em 0 0em #2375e1,
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  75% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.5),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.7), -2.6em 0em 0 0em #2375e1,
      -1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2);
  }
  87.5% {
    box-shadow: 0em -2.6em 0em 0em rgba(35, 117, 225, 0.2),
      1.8em -1.8em 0 0em rgba(35, 117, 225, 0.2),
      2.5em 0em 0 0em rgba(35, 117, 225, 0.2),
      1.75em 1.75em 0 0em rgba(35, 117, 225, 0.2),
      0em 2.5em 0 0em rgba(35, 117, 225, 0.2),
      -1.8em 1.8em 0 0em rgba(35, 117, 225, 0.5),
      -2.6em 0em 0 0em rgba(35, 117, 225, 0.7), -1.8em -1.8em 0 0em #2375e1;
  }
}
</style>
