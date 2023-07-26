<template>
  <div class="scanner-container">
    <div v-show="!isLoading">
      <video ref="scanner" poster="data:image/gif,AAAA"></video>
      <div class="overlay-element"></div>
      <div class="laser">
        <div class="laser-inner"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { BrowserMultiFormatReader, Exception } from '@zxing/library'

export default {
  name: 'stream-barcode-reader',
  emits: ['loaded', 'decode', 'result', 'update:hasTorch', 'update:hasZoom', 'update:hasAutoFocus', 'update:hasFocusDistance'],

  props: {
    torch: {
      type: Boolean,
      default: false,
    },
    zoom: {
      type: Number,
      default: 1,
    },
    autoFocus: {
      type: Boolean,
      default: true,
    },
    focusDistance: {
      type: Number,
      default: 0,
    },
    landscape: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      idealDevice: {},
      isLoading: true,
      codeReader: new BrowserMultiFormatReader(),
      isMediaStreamAPISupported: navigator && navigator.mediaDevices && 'enumerateDevices' in navigator.mediaDevices,
     };
  },

  computed: {
    hasTorch() {
      return !!this.idealDevice?.torch
    },
    hasZoom() {
      return this.idealDevice?.zoom ? this.idealDevice.zoom : false
    },
    hasAutoFocus() {
      return !!this.idealDevice?.focusMode?.includes('continuous')
    },
    hasFocusDistance() {
      return this.idealDevice?.focusDistance ? this.idealDevice.focusDistance : false
    },
  },

  watch: {
    torch() {
      if (this.hasTorch) {
        this.applyCameraConstraints()
      }
    },
    zoom() {
      if (this.hasZoom) {
        this.applyCameraConstraints()
      }
    },
    autoFocus() {
      if (this.hasAutoFocus) {
        this.applyCameraConstraints()
      }
    },
    focusDistance() {
      if (!this.autoFocus) {
        this.applyCameraConstraints()
      }
    },
    landscape() {
      if (this.landscape) {
        this.fullscreenLandscape()
      } else {
        this.exitFullscreenLandscape()
      }
    },
  },

  mounted() {
    if (!this.isMediaStreamAPISupported) {
      throw new Exception('Media Stream API is not supported')
      return
    }

    this.idealDevice = JSON.parse(localStorage.getItem('vue-barcode-reader-ideal') || '{}')

    if (this.landscape) {
      this.fullscreenLandscape()
    }

    // If ideal device is already set, use it, if not, find the ideal device
    if (typeof this.idealDevice === 'object' && Object.keys(this.idealDevice).length > 0) {
      this.selectCamera()
    } else {
      navigator.mediaDevices.enumerateDevices().then(devices => {
        this.findIdealDevice(devices).then(() => {
          this.selectCamera()
        })
      })
    }
  },

  beforeUnmount() {
    this.isLoading = true
    this.codeReader.reset()
    if (this.landscape) {
      this.exitFullscreenLandscape()
    }
  },

  methods: {
    async findIdealDevice(devices) {
      let deviceOptions = []
      const cameras = devices.filter((device) => device.kind === 'videoinput' && device.label.toLowerCase().indexOf('front') === -1)
      for (let index = 0; index < cameras.length; index++) {
        await navigator.mediaDevices.getUserMedia(
          {
            video: {
              deviceId: {
                exact: cameras[index].deviceId,
              },
            }
          })
          .then(stream => {
            const track = stream.getVideoTracks()[0]
            const trackCapabilities = track.getCapabilities()
            deviceOptions.push(trackCapabilities)
            stream.getTracks().forEach(track => { track.stop() })
            return true
          })
          .catch(() => { return false })
      }

      if (deviceOptions.length > 0) {
        // If rear facing (environment) camera(s), use only those
        const environmentFacing = deviceOptions.filter(device => device.facingMode.includes('environment'))
        if (environmentFacing.length > 0) {
          deviceOptions = environmentFacing
        }

        // Find ideal device (hopefully includes torch and continuous focus)
        let idealIndex = deviceOptions.length -1
        for (let index = 0; index < deviceOptions.length; index++) {
          if (deviceOptions[index].torch && deviceOptions[index].focusMode.includes('continuous')) {
            idealIndex = index
            break
          } else if (deviceOptions[index].torch) {
            idealIndex = index
          }
        }

        this.idealDevice = deviceOptions[idealIndex]
        localStorage.setItem('vue-barcode-reader-ideal', JSON.stringify(deviceOptions[idealIndex]))
        return true
      } else {
        localStorage.removeItem('vue-barcode-reader-ideal')
        return false
      }
    },

    async fullscreenLandscape() {
      await document.documentElement.requestFullscreen()
        .then(() => {
          screen.orientation.lock('landscape').catch(() => {})
          return true
        })
        .catch(() => { return false })
    },

    async exitFullscreenLandscape() {
      screen.orientation.lock('portrait-primary').catch(() => {})
      await document.exitFullscreen()
        .then(() => { return true })
        .catch(() => { return false })
    },

    selectCamera() {
      // Make sure the ideal device we found is available with the code reader (it should always be, but just in case...)
      this.codeReader.listVideoInputDevices().then((videoInputDevices) => {
        if (videoInputDevices.findIndex(device => device.deviceId === this.idealDevice.deviceId) === -1) {
          this.idealDevice = {}
          localStorage.removeItem('vue-barcode-reader-ideal')
        }

        // Now start the barcode reader
        this.startCodeReader(this.idealDevice.deviceId)
        this.$refs.scanner.oncanplay = (event) => {
          this.isLoading = false
          this.$emit('loaded')
          this.$emit('update:hasTorch', this.hasTorch)
          this.$emit('update:hasZoom', this.hasZoom)
          this.$emit('update:hasAutoFocus', this.hasAutoFocus)
          this.$emit('update:hasFocusDistance', this.hasFocusDistance)

          // Finally, turn on features as available and requested
          this.applyCameraConstraints()
        }

      })
    },

    applyCameraConstraints() {
      const advanced = {}
      if (this.hasTorch) advanced.torch = this.torch
      if (this.hasZoom) advanced.zoom = Math.min(Math.max(this.idealDevice.zoom.min, this.zoom), this.idealDevice.zoom.max)
      if (this.hasAutoFocus) advanced.focusMode = this.autoFocus ? 'continuous' : 'manual'
      if (!this.autoFocus && this.hasFocusDistance) advanced.focusDistance = Math.min(Math.max(this.idealDevice.focusDistance.min, this.focusDistance), this.idealDevice.focusDistance.max)
      this.$refs.scanner?.srcObject?.getVideoTracks()[0]?.applyConstraints({
        advanced: [advanced]
      }).catch(() => {})
    },

    startCodeReader(deviceId) {
      this.codeReader.decodeFromVideoDevice(deviceId, this.$refs.scanner, (result, err) => {
        if (result) {
          this.$emit('decode', result.text)
          this.$emit('result', result)
        }
      })
    },
  },
};
</script>

<style scoped>
.scanner-container {
  position: relative;
}

video {
  display: block;
  width: 100%;
  height: auto;
  max-width: 100vw;
  max-height: 100vh;
}

.overlay-element {
  position: absolute;
  top: 0;
  width: 100%;
  height: 100%;
  background: rgba(30, 30, 30, 0.5);
  clip-path: polygon(0% 0%, 0% 100%, 20% 100%, 20% 20%, 80% 20%, 80% 80%, 20% 80%, 20% 100%, 100% 100%, 100% 0%);
}

.laser {
  width: 60%;
  margin-left: 20%;
  background-color: rgba(255, 99, 71, 0.75);
  height: 2px;
  position: absolute;
  top: 50%;
  z-index: 2;
  animation: scanning-laser 1.5s infinite;
}

.laser-inner {
  position: absolute;
  width: 35%;
  left: 0%;
  height: 2px;
  background-color: #ff2c06;
  box-shadow: 0 0 4px #ff2c06;
  animation: scanning-laser-inner 0.23s infinite;
}

@keyframes scanning-laser {
  0% {
    transform: translateY(3vh);
  }
  50% {
    transform: translateY(-3vh);
  }
  100% {
    transform: translateY(3vh);
  }
}

@keyframes scanning-laser-inner {
  0% {
    left: 0%;
  }
  50% {
    left: 65%;
  }
  100% {
    left: 0%;
  }
}
</style>
