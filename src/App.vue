<template>
  <div class="flex">
    <div class="relative">
      <video ref="videoElement" width="800" height="600" autoplay></video>

      <div
        v-for="keypoint in keypoints"
        :key="keypoint.part"
        class="absolute flex items-center justify-center w-4 h-4 bg-red-500 rounded-full"
        :style="{
          top: keypoint.position.y + 'px',
          left: keypoint.position.x + 'px',
        }"
      ></div>
    </div>
    <div v-if="isOk" ref="successLottie" class="w-1/4 h-1/4"></div>
    <div v-else ref="warningLottie" class="w-1/4 h-1/4"></div>
  </div>
  <h1 class="font-bold text-9xl">حالة المريض : {{ patientStatus }}</h1>
</template>

<script setup>
import { ref, onMounted, computed } from "vue";
import lottie from "lottie-web";
const successLottie = ref(null);
const warningLottie = ref(null);
const videoElement = ref(null);
const keypoints = ref([]);
const patientStatus = ref("");
let poseNetModel = null;
let videoWidth = 0;
let videoHeight = 0;

onMounted(async () => {
  const animation = lottie.loadAnimation({
    container: successLottie.value,
    renderer: "svg", // Set the renderer type ('svg', 'canvas', 'html') based on your preference
    loop: true,
    autoplay: true,
    path: "/success.json", // Path to your Lottie animation JSON file
  });
  const animation2 = lottie.loadAnimation({
    container: warningLottie.value,
    renderer: "svg", // Set the renderer type ('svg', 'canvas', 'html') based on your preference
    loop: true,
    width: 100,
    height: 100,
    autoplay: true,

    path: "/warning.json", // Path to your Lottie animation JSON file
  });
  poseNetModel = await posenet.load();

  if (navigator.mediaDevices.getUserMedia) {
    navigator.mediaDevices
      .getUserMedia({ video: true })
      .then((stream) => {
        videoElement.value.srcObject = stream;
      })
      .catch((error) => {
        console.error("Error accessing the camera: ", error);
      });
  }

  videoElement.value.addEventListener("loadedmetadata", () => {
    videoWidth = videoElement.value.videoWidth;
    videoHeight = videoElement.value.videoHeight;
    detectPose();
  });
});

const detectPose = async () => {
  const video = videoElement.value;

  const pose = await poseNetModel.estimateSinglePose(video, {
    flipHorizontal: false,
  });

  keypoints.value = pose.keypoints;

  const hipJoint = pose.keypoints[11];
  const kneeJoint = pose.keypoints[13];

  if (
    hipJoint.score >= 0.5 &&
    kneeJoint.score >= 0.5 &&
    hipJoint.position.y < kneeJoint.position.y
  ) {
    patientStatus.value = "مستلقي";
  } else if (
    hipJoint.score >= 0.5 &&
    kneeJoint.score >= 0.5 &&
    hipJoint.position.y >= kneeJoint.position.y
  ) {
    patientStatus.value = "جالس";
  } else {
    patientStatus.value = "واقف";
  }

  requestAnimationFrame(detectPose);
};

const isOk = computed(() => {
  return poseNetModel === "مستلقي";
});
</script>
