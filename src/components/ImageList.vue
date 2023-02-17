
<script setup lang="ts">









import JSZip from 'jszip';
import saveAs from 'file-saver';

import '@mediapipe/face_detection';
import '@tensorflow/tfjs-core';
import '@tensorflow/tfjs-backend-webgl';
import * as faceDetection from '@tensorflow-models/face-detection';
import { ref, onMounted, onUpdated } from 'vue'

const URL = window.URL || window.webkitURL;
const imgList = ref(['']);
let size = ref("96");
const lg = (msg: any) => { console.log(msg) }
let imgTags: HTMLCollection;
let canvasArea: HTMLElement;
let detector: faceDetection.FaceDetector;

onMounted(async () => {
  canvasArea = document.getElementById("canvasArea")!;
  const model = faceDetection.SupportedModels.MediaPipeFaceDetector;
  detector = await faceDetection.createDetector(model, { runtime: 'tfjs' });
  lg('faceDetector ready')
})

const onDrop = (e: DragEvent) => {
  const files = e.dataTransfer?.files;
  if (!files) return;
  resetImage();
  for (let i = 0; i < files.length; i++) {
    const url = URL.createObjectURL(files[i]);
    imgList.value.push(url);
  }
}

onUpdated(() => { for (let url in imgList) URL.revokeObjectURL(url); })

const resetImage = () => {
  imgList.value = [];
  resetCanvas();
}

const resetCanvas = () => {
  canvasArea = document.getElementById("canvasArea")!;
  while (canvasArea?.firstChild)
    canvasArea.removeChild(canvasArea.firstChild);
}

interface Box { x: number; y: number; w: number; h: number; }
const trimType = { face: 'face', mouth: 'mouth', ear: 'ear' }

const trimImage = async (type: string) => {
  resetCanvas();

  imgTags = document.getElementsByTagName("img");

  const drawCanvas = (img: HTMLImageElement, box: Box) => {
    const { x, y, w, h } = box;
    const canvas = document.createElement("canvas");
    const selected = parseInt(size.value);
    const [tw, th] = selected == 0 ? [w, h] : [selected, selected];
    lg(tw);
    [canvas.width, canvas.height] = [tw, th];
    [canvas.style.maxWidth, canvas.style.maxHeight] = ['96px', '96px'];
    canvas.getContext("2d")!.drawImage(img, x, y, w, h, 0, 0, tw, th);
    canvasArea.appendChild(canvas);
  }

  for (let i = 0; i < imgTags.length; i++) {
    const img = <HTMLImageElement>imgTags[i];

    const cfg = { flipHorizontal: false };
    const faces = await detector.estimateFaces(img, cfg);
    if (faces?.length > 0 && faces[0].box) {

      const { xMin, yMin, width, height } = faces[0].box;
      const r = img.naturalWidth / img.width;
      let [x, y, w, h] = [xMin, yMin, width, height].map(v => v * r);
      const faceBox: Box = { x, y, w, h };

      const mouth = faces[0].keypoints[3];
      const [mx, my] = [mouth.x, mouth.y].map(v => v * r);
      w = faceBox.w / 2;
      const mouthBox: Box = { x: mx - w / 2, y: my - w / 2, w, h: w };
      switch (type) {
        case trimType.face: drawCanvas(img, faceBox); break;
        case trimType.mouth: drawCanvas(img, mouthBox); break;
      }
    }

  }

}

const download = async () => {
  if (!imgList) return;

  const allCanvas = <HTMLCollection>canvasArea.getElementsByTagName("canvas");
  let promises = new Array();
  const zip = new JSZip();
  const cfg = { base64: true };
  for (let i = 0; i < allCanvas.length; i++) {
    const canvas = <HTMLCanvasElement>allCanvas[i];
    promises.push(new Promise(res => {
      canvas.toBlob(blob => { zip.file(`${i}.jpg`, blob, cfg); res(''); })
    }))
  }
  await Promise.all(promises);
  saveAs(await zip.generateAsync({ type: "blob" }));

}

const openTM = () => {
  const url = "https://teachablemachine.withgoogle.com/train/tiny_image";
  window.open(url, '_blank');
}

const openTry = () => {
  const url = "";
  window.open(url, '_blank');
}


</script>

<template>
  <div class="dropArea" @dragover.prevent @drop.prevent="onDrop">
    <div class="header">
      <div class="titleText">Image Trimmer</div>
      <button @click="resetImage">Reset</button>
      <button @click="trimImage('face')">TrimFace</button>
      <button @click="trimImage('mouth')">TrimMouth</button>
      Size:
      <select v-model="size">
        <option value="96">96x96</option>
        <option value="224">224x224</option>
        <option value="0">Original</option>
      </select>
      <button @click="download">Download</button>
    </div>
    <br />
    <div class="imageText">=== Target images ===</div>
    <img class="droppedImg" v-for="imgURL in imgList" :src="imgURL" />

    <div class="imageText">=== Trimmed images ===</div>
    <div id="canvasArea"></div>
  </div>
</template>

<style scoped>
.dropArea {
  width: 100vw;
  height: auto;
  min-height: 100vh;
  background-color: black;
}

.header {
  background: linear-gradient(darkgray, black);
  width: 100vw;
  padding: 5px 0px 40px 10px;
  vertical-align: top;
}

.titleText {
  display: inline-flex;
  vertical-align: top;
  font-size: 150%;
  /* font-style: italic; */
  margin-right: 30px;
  margin-top: 3px;
}

select,
button {
  margin-right: 15px;
  border-color: black;
  border-width: 2px;
  border-radius: 15px;
  padding: 5px;
  font-size: 120%;
  text-align: center;
  width: 130px;
  height: 35px;
}

button:hover {
  background-color: bisque;
}

.imageText {
  margin-left: 10px;
  color: white;
  font-size: 150%;
}

.droppedImg {
  margin: 3px;
  max-width: 120px;
  max-height: 120px;
  box-shadow: 3px 3px 5px rgb(100, 100, 100, 0.6);
}

#canvasArea :deep(canvas) {
  margin: 3px;
  max-width: 120px;
  max-height: 120px;
  box-shadow: 3px 3px 5px rgb(100, 100, 100, 0.6);
}
</style>
