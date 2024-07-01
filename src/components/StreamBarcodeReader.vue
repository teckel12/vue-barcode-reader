<template>
  <div class="scanner-container">
    <div v-show="!isLoading">
      <video ref="scanner" poster="data:image/gif,AAAA"></video>
      <div class="overlay-element"></div>
      <div class="laser">
        <div class="laser-inner"></div>
      </div>
    </div>
    <!--<pre>{{ cameraDetails }}</pre>-->
  </div>
</template>

<script>
import { BrowserMultiFormatReader, Exception } from '@zxing/library'

export default {
  name: 'stream-barcode-reader',
  emits: ['loaded', 'decode', 'result', 'update:hasTorch', 'update:hasZoom', 'update:hasAutofocus', 'update:hasFocusDistance', 'update:videoDevices', 'update:cameraDetails'],

  props: {
    torch: {
      type: Boolean,
      default: false,
    },
    zoom: {
      type: Number,
      default: 1,
    },
    autofocus: {
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
    deviceIndex: {
      type: Number,
      default: 0,
    },
    noFrontCameras: {
      type: Boolean,
      default: false,
    },
    msBetweenDecoding: {
      type: Number,
      default: 500,
    },
  },

  data() {
    return {
      idealDevice: {},
      cameraDetails: {},
      videoDevices: [],
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
    hasAutofocus() {
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
    autofocus() {
      if (this.hasAutofocus) {
        this.applyCameraConstraints()
      }
    },
    focusDistance() {
      if (!this.autofocus) {
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
    deviceIndex() {
      if (this.deviceIndex != this.videoDevices.selectedIndex && this.deviceIndex == Number(this.deviceIndex) && this.videoDevices?.devices?.length > 1) {
        this.isLoading = true
        this.codeReader.reset()
        this.cameraDetails = {}
        this.cameraDetails.previousDevice = this.idealDevice
        const deviceId = this.videoDevices?.devices[this.deviceIndex]?.deviceId
        navigator.mediaDevices.enumerateDevices().then(devices => {
          this.findIdealDevice(devices, deviceId).then(() => {
            this.selectCamera()
          })
        })
      }
    },
  },

  mounted() {
    if (!this.isMediaStreamAPISupported) {
      throw new Exception('Media Stream API is not supported')
      return
    }

    this.idealDevice = JSON.parse(localStorage.getItem('vue-barcode-reader-ideal') || '{}')
    this.cameraDetails.previousDevice = this.idealDevice

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
    async findIdealDevice(devices, deviceId = false) {
      let deviceOptions = []
      let cameras = []
      if (deviceId) {
        // Specific camera device specified, use this camera if it exists
        cameras = devices.filter((device) => device.kind === 'videoinput' && device.deviceId === deviceId)
      }
      if (cameras?.length !== 1) {
        // Filter for the ideal camera
        cameras = devices.filter((device) => device.kind === 'videoinput' && device.label.toLowerCase().indexOf('front') === -1)
        if (cameras?.length === 0) {
          cameras = devices.filter((device) => device.kind === 'videoinput')
        }
      }
      this.cameraDetails.requestedDeviceId = deviceId
      this.cameraDetails.cameras = devices.filter(device => device.kind === 'videoinput')
      this.cameraDetails.filteredCameras = cameras
      this.cameraDetails.constraints = []
      for (let index = 0; index < cameras.length; index++) {
        const constraints = { video: true }
        if (deviceId) {
          constraints.video = {
            deviceId: {
              exact: cameras[index].deviceId,
            }
          }
        } else {
          constraints.video = { facingMode: 'environment' }
          if (cameras[index].deviceId) {
            constraints.video.deviceId = {
              exact: cameras[index].deviceId,
            }
          }
        }
        this.cameraDetails.constraints.push(constraints)
        await navigator.mediaDevices.getUserMedia(constraints)
          .then(stream => {
            const track = stream.getVideoTracks()[0]
            const trackCapabilities = {
              deviceId: cameras[index].deviceId,
              ...track.getCapabilities?.() ?? {}
            }
            deviceOptions.push(trackCapabilities)
            stream.getTracks().forEach(track => { track.stop() })
            return true
          })
          .catch(() => { return false })
      }

      this.cameraDetails.deviceOptions = deviceOptions

      if (deviceOptions.length > 0) {
        // If rear facing (environment) camera(s), use only those
        const environmentFacing = deviceOptions.filter(device => device.facingMode?.includes('environment'))
        if (environmentFacing.length > 0) {
          deviceOptions = environmentFacing
        }

        // Find ideal device (hopefully includes torch and continuous focus)
        let idealIndex = deviceOptions.length - 1
        for (let index = 0; index < deviceOptions.length; index++) {
          if (deviceOptions[index].torch && deviceOptions[index].focusMode?.includes('continuous')) {
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
      // Make sure the ideal device we found is available with the code reader (if not, do a search)
      this.codeReader.listVideoInputDevices().then((videoInputDevices) => {
        if (this.noFrontCameras) {
          videoInputDevices = videoInputDevices.filter(device => device.label.toLowerCase().indexOf('front') === -1)
        }
        this.videoDevices = { devices: videoInputDevices }
        if (videoInputDevices.findIndex(device => device.deviceId === this.idealDevice.deviceId) === -1) {
          this.idealDevice = {}
          localStorage.removeItem('vue-barcode-reader-ideal')
          navigator.mediaDevices.enumerateDevices().then(devices => {
            this.findIdealDevice(devices).then(() => {
              this.selectCamera()
            })
          })
          return
        }

        // Now start the barcode reader
        this.startCodeReader(this.idealDevice.deviceId)
        this.$refs.scanner.oncanplay = (event) => {
          this.isLoading = false
          this.videoDevices.selectedId = this.idealDevice.deviceId
          this.videoDevices.selectedIndex = this.videoDevices?.devices?.findIndex(device => device.deviceId === this.idealDevice.deviceId)
          this.cameraDetails.videoDevices = this.videoDevices.devices
          this.cameraDetails.selectedIndex = this.videoDevices.selectedIndex
          this.cameraDetails.selectedDeviceId = this.idealDevice.deviceId
          this.cameraDetails.selectedDevice = this.idealDevice
          this.$emit('loaded')
          this.$emit('update:hasTorch', this.hasTorch)
          this.$emit('update:hasZoom', this.hasZoom)
          this.$emit('update:hasAutofocus', this.hasAutofocus)
          this.$emit('update:hasFocusDistance', this.hasFocusDistance)
          this.$emit('update:videoDevices', this.videoDevices)

          this.applyCameraConstraints()
        }
      })
    },

    applyCameraConstraints() {
      const advanced = {}
      if (this.hasTorch) advanced.torch = this.torch
      if (this.hasZoom) advanced.zoom = Math.min(Math.max(this.idealDevice.zoom.min, this.zoom), this.idealDevice.zoom.max)
      if (this.hasAutofocus || (!this.hasAutofocus && !this.autofocus && this.hasFocusDistance)) advanced.focusMode = this.autofocus ? 'continuous' : 'manual'
      if (!this.autofocus && this.hasFocusDistance) advanced.focusDistance = Math.min(Math.max(this.idealDevice.focusDistance.min, this.focusDistance), this.idealDevice.focusDistance.max)
      this.cameraDetails.applyConstraints = advanced
      this.$emit('update:cameraDetails', this.cameraDetails)
      this.$refs.scanner?.srcObject?.getVideoTracks()[0]?.applyConstraints({
        advanced: [advanced]
      }).catch(() => {})
    },

    startCodeReader(deviceId) {
      this.codeReader.timeBetweenDecodingAttempts = this.msBetweenDecoding
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
  max-width: fit-content;
  margin: 0 auto;
}

.scanner-container pre {
  position: fixed;
  top: 50px;
  left: 0;
  bottom: 0;
  right: 0;
  margin: 0;
  color: #ccc;
  background-color: rgba(0, 0, 0, 0.25);
  text-align: left;
  overflow-y: scroll;
}

video {
  display: block;
  margin: auto;
}

.overlay-element {
  position: absolute;
  top: 0;
  width: 100%;
  height: 100%;
  background: rgba(30, 30, 30, 0.5);
  clip-path: polygon(0% 0%, 0% 100%, 20% 100%, 20% 20%, 80% 20%, 80% 80%, 20% 80%, 20% 100%, 100% 100%, 100% 0%);
}

@media screen and (orientation:portrait) {
  video {
    width: 100%;
    max-width: 100vh;
    height: auto;
  }
}
@media screen and (orientation:landscape) {
  video {
    height: 100%;
    max-height: 100vh;
    width: auto;
  }
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
