# Vue Barcode and QR code scanner

[![npm version](https://badgen.net/npm/v/vue-barcode-reader)](https://www.npmjs.com/package/vue-barcode-reader)

A Vue.js set of components to scan (or upload images) barcodes and QR codes.

## Changes in this fork

Fixes a library version issue and adds the ability to activate the torch (camera flash).  Both yield much better barcode scanning speed and accuracy.

## Benefits

- Can scan both barcodes and QR codes
- Uses [ZXing](https://github.com/zxing-js/library) ("zebra crossing"), an open-source, multi-format 1D/2D barcode image processing library implemented in Java, with ports to other languages.

## Demo

TODO: Update demos

[Demo repository](https://github.com/olefirenko/vue-barcode-reader-example) | [Codesandbox](https://codesandbox.io/s/vue-barcode-reader-demo-guj3f) | [Vercel](https://vue-barcode-reader-example-2iiz1fhbf-olefirenko.vercel.app/) | [Netlify](https://stupefied-meitner-58f299.netlify.app/)

## Installation

The easiest way to use Vue Barcode Reader is to install it from **npm** or **yarn**.

```sh
npm install @teckel/vue-barcode-reader --save
```

Or

```sh
yarn add @teckel/vue-barcode-reader
```

### Vue 2.0 support

For Vue 2.0 compatible version please use the `vue-barcode-reader@0.0.3`?

## Usage

The Vue Barcode Reader works out of the box by just including it.

### Using Video Camera

Once a stream from the users camera is loaded, it's displayed and continuously scanned for barcodes. Results are indicated by the decode event.

```js
import { StreamBarcodeReader } from "@teckel/vue-barcode-reader";
```

In your template you can use this syntax:

```html
<StreamBarcodeReader @decode="onDecode" @loaded="onLoaded" torch></StreamBarcodeReader>
```

### Scanning from Image

The component renders to a simple file picker input element. Clicking opens a file dialog. On supporting mobile devices the camera is started to take a picture. The selected images are directly scanned and positive results are indicated by the `decode` event.

```js
import { ImageBarcodeReader } from "vue-barcode-reader";
```

In your template you can use this syntax:

```html
<ImageBarcodeReader @decode="onDecode" @error="onError"></ImageBarcodeReader>
```

```html
methods: { onDecode (result) { console.log(result) } }
```

## Props

The library supports the following props:

### torch (if supported by camera)

Activate the torch (flash), can be set with simply `torch` or controlled via `:torch="torch"`

### zoom (if supported by camera)

Set the zoom value (min/max/step available in `hasZoom` emitted value)

### autoFocus (if supported by camera)

Defaults to `true`, but setting `:auto-focus="false"` turns off auto focus

### focusDistance (if supported by camera)

Set the focus distance (min/max/step available in `hasFocusDistance` emited value)


## Emitted values

The library supports the following emitted values:

### hasTorch

Returns `true` or `false` if camera is capable of activating the torch (flash)

### hasAutoFocus

Returns `true` or `false` if camera is capable of auto focus

### hasZoom

Returns `false` or object containing `min`, `max`, `step`

### hasFocusDistance

Returns `false` or object containing `min`, `max`, `step`


## Events

The library emits the following events:

### loaded

When the libraty is loaded and the camera is ready to scan

### decode

When a barcode or QR code is scanned. The result is passed as a parameter to the event handler. The result is the text encoded in the barcode or QR code.

### result

When a barcode or QR code is scanned. The result is passed as a parameter to the event handler. Below is an example result object from the UPC code of a box of Kellogg's Frosted Mini-Wheats:

```
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
