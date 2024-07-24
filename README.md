# Vue 3 Barcode and QR Code Scanner

A Vue.js set of components to scan barcodes and QR codes (or upload images).

## Enhancements

> The following bug fixes, features & improvements over the abandoned package by [olefirenko](https://github.com/olefirenko/vue-barcode-reader)

* Fixes ZXing library version issue which greatly increases scanning speed.
* On startup, the library searches all available rear-facing cameras to find the most ideal camera for barcode scanning, preferably one with torch (flash) and autofocus.  It also saves this ideal camera to local storage for faster startup on repeat scans.
* Adds option to activate the torch (camera flash), which can yield higher barcode scanning speed and accuracy (Android only).
* Adds option to cycle through the available cameras (if more than one camera is available).
* Adds option to set orientation to landscape mode, this can also increase the scanning speed and accuracy as there's more horizontal pixels in the landscape orientation.
* Adds option to control the camera zoom (if camera device reports the user is allowed to set the zoom).
* Adds option to switch between autofocus and manual focus mode (defaults to autofocus mode if available).
* Adds option to set the focus distance (if in manual focus mode and camera device supports the feature).
* Adds option to only select from rear-facing cameras (you probably only want rear-facing cameras when scanning barcodes).
* Adds option to control the time between decode scans (defaults to 500ms).

## Benefits

* Can scan both barcodes and QR codes
* Uses [ZXing](https://github.com/zxing-js/library) ("zebra crossing"), an open-source, multi-format 1D/2D barcode image processing library implemented in Java, with ports to other languages.

## Demo

[Demo](https://barcode.leethost.com) | [Demo repository](https://github.com/teckel12/vue-barcode-reader-example)

## Installation

The easiest way to use Vue Barcode Reader is to install it with **npm** or **yarn**.

```sh
npm install @teckel/vue-barcode-reader --save
```

Or

```sh
yarn add @teckel/vue-barcode-reader
```

## Usage

The Vue Barcode Reader works out of the box by just including it.

### Scanning from Video Camera

Once a stream from the users camera is loaded, it's displayed and continuously scanned for barcodes. Results are indicated by the decode event.

Composition API example:

```js
<script setup>
import { ref } from 'vue'
import { StreamBarcodeReader } from '@teckel/vue-barcode-reader'

const decodedText = ref('')

const onDecode = (result) => {
  decodedText = result
}

const onLoaded = () => {
  console.log('loaded')
}
</script>

<template>
  <StreamBarcodeReader
    torch
    no-front-cameras
    @decode="onDecode"
    @loaded="onLoaded"
  />
  <h2>Decoded value: {{ decodedText }}</h2>
</template>
```

Options API example:

```js
<template>
  <StreamBarcodeReader
    torch
    no-front-cameras
    @decode="onDecode"
    @loaded="onLoaded"
  />
  <h2>Decoded value: {{ decodedText }}</h2>
</template>

<script>
import { StreamBarcodeReader } from '@teckel/vue-barcode-reader'

export default {
  components: { StreamBarcodeReader },
  data() {
    return {
      decodedText: '',
    }
  },
  methods: {
    onDecode(result) {
      this.decodedText = result
    },
    onLoaded() {
      console.log('loaded')
    },
  }
}
</script>
```

### Scanning from Image

The component renders to a simple file picker input element. Clicking opens a file dialog. On supporting mobile devices the camera is started to take a picture. The selected images are directly scanned and positive results are indicated by the `decode` event.

```js
import { ImageBarcodeReader } from '@teckel/vue-barcode-reader'
```

In your template you can use this syntax:

```html
<ImageBarcodeReader
  @decode="onDecode"
  @error="onError"
/>
```

## Props

> Props will only work if the camera reports that the feature is supported.  Some camera devices and some platforms either don't allow setting constraints or don't report the feature exists.  Chrome on Android work quite well, while (as expected) iOS and Safari don't support most/all features.

### `torch`

Activate the torch (flash). Can be set with simply `torch` or controlled via `:torch="torch"`.

### `zoom`

Set the zoom value (min/max/step available in `hasZoom` emitted value).

### `landscape`

Set the browser to landscape orientation.  In order to set landscape mode, the browser will first switch to fullscreen mode (this is required to force landscape mode).

### `autofocus`

Defaults to `true`, but setting `:autofocus="false"` turns off autofocus (manual focus).

### `focus-distance`

> Must have `:autofocus="false"` (turning off autofocus and turning on manual focus) for focus-distance to work.

Set the focus distance (min/max/step available in `hasFocusDistance` emitted value).

### `no-front-cameras`

Only selects from rear-facing cameras.  This only works if the device reports the camera's orientation.

### `device-index`

Select the index of the camera device to use (get the camera device array from the `videoDevices` emitted value).

### `msBetweenDecoding`

Set the time between decode scans (defaults to 500ms).  This is useful if you want to limit the number of scans per second (for example, if you're scanning a barcode on a moving object, you may want to limit the number of scans per second).

## Emitted values

### `hasTorch`

Returns `true` or `false` if camera device reports it's capable of activating the torch (flash).

### `hasAutofocus`

Returns `true` or `false` if camera device reports it's capable of autofocus mode.

### `hasZoom`

Returns `false` or object containing `min`, `max`, `step` set from the supported camera device.

### `hasFocusDistance`

Returns `false` or object containing `min`, `max`, `step` set from the supported camera device.

### `videoDevices`

Returns an array of camera devices available to the browser (can be used to select the desired camera device via `device-index` prop).

### `cameraDetails`

Object dump of the library processing to select the ideal camera, switch cameras, and apply constraints (useful for debugging or could be used for enhanced features).

## Events

### `loaded`

When the library is loaded and the camera is ready to scan

### `decode`

When a barcode or QR code is scanned. The result is passed as a parameter to the event handler. The result is the text encoded in the barcode or QR code.

### `result`

When a barcode or QR code is scanned. The result is passed as a parameter to the event handler. Below is an example result object from the UPC code of a box of Kellogg's Frosted Mini-Wheats:

```json
{
    "text": "038000199349",
    "rawBytes": null,
    "numBits": 0,
    "resultPoints": [
        {
            "x": 189,
            "y": 240
        },
        {
            "x": 445.5,
            "y": 240
        }
    ],
    "format": 14,
    "timestamp": 1690401753332,
    "resultMetadata": null
}
```

Barcode formats:

* 0 AZTEC,
* 1 CODABAR,
* 2 CODE_39,
* 3 CODE_93,
* 4 CODE_128,
* 5 DATA_MATRIX,
* 6 EAN_8,
* 7 EAN_13,
* 8 ITF,
* 9 MAXICODE,
* 10 PDF_417,
* 11 QR_CODE,
* 12 RSS_14,
* 13 RSS_EXPANDED,
* 14 UPC_A,
* 15 UPC_E,
* 16 UPC_EAN_EXTENSION
