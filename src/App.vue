<template>
	<div id="app">
		<video id="video" playsinline></video>
		<div class="emoji-container" :style="{ transform: emojiTransform }">
			<div class="emoji">{{ emoji }}</div>
		</div>
		<select v-model="selectedCamera" @change="cameraChange">
			<option v-for="camera in cameras" :value="camera.deviceId" :key="camera.deviceId" select>
				{{ camera.label }}
			</option>
		</select>
	</div>
</template>

<script>
import * as bodyPix from '@tensorflow-models/body-pix';

const emojis = ['😀', '🤪', '🙈', '🤬', '🤯', '🥶', '💩'];

const state = {
	video: null,
	stream: null,
	net: null,
	videoConstraints: {},
	frame: 0,
};

async function getVideoInputs() {
	if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
		console.log('enumerateDevices() not supported.');
		return [];
	}

	const devices = await navigator.mediaDevices.enumerateDevices();

	const videoDevices = devices.filter((device) => device.kind === 'videoinput');

	return videoDevices;
}

function stopExistingVideoCapture() {
	if (state.video && state.video.srcObject) {
		state.video.srcObject.getTracks().forEach((track) => {
			track.stop();
		});
		state.video.srcObject = null;
	}
}

/**
 * Loads a the camera to be used in the demo
 *
 */
async function setupCamera(deviceId) {
	if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
		throw new Error('Browser API navigator.mediaDevices.getUserMedia not available');
	}

	const videoElement = document.getElementById('video');

	stopExistingVideoCapture();

	const stream = await navigator.mediaDevices.getUserMedia({ audio: false, video: { deviceId } });
	videoElement.srcObject = stream;

	return new Promise((resolve) => {
		videoElement.onloadedmetadata = () => {
			videoElement.width = videoElement.videoWidth;
			videoElement.height = videoElement.videoHeight;
			resolve(videoElement);
		};
	});
}

async function loadVideo(deviceId) {
	try {
		state.video = await setupCamera(deviceId);
	} catch (e) {
		let info = document.getElementById('info');
		info.textContent = 'this browser does not support video capture,' + 'or this device does not have a camera';
		info.style.display = 'block';
		throw e;
	}

	state.video.play();
}

function segmentPerson() {
	async function bodySegmentationFrame() {
		const personPart = await state.net.segmentPersonParts(state.video, {
			flipHorizontal: false,
			internalResolution: 'medium',
			segmentationThreshold: 0.7,
		});
		if (personPart.allPoses.length && personPart.allPoses[0].score > 0.4) {
			let part = personPart.allPoses[0].keypoints[0];
			this.$set(this, 'emojiX', Math.round((part.position.x / 640) * 100));
			this.$set(this, 'emojiY', Math.round((part.position.y / 480) * 100));
		}
		requestAnimationFrame(bodySegmentationFrame.bind(this));
	}

	bodySegmentationFrame.call(this);
}

async function bindPage() {
	state.net = await bodyPix.load();
	let cameras = await getVideoInputs();
	if (cameras.length) {
		this.$set(this, 'selectedCamera', cameras[0].deviceId);
	}
	this.$set(this, 'cameras', cameras);
	await loadVideo(this.selectedCamera);
	segmentPerson.call(this);
}

navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia;

export default {
	name: 'app',
	data: () => ({
		emojiX: 0,
		emojiY: 0,
		emoji: emojis[Math.round(Math.random() * emojis.length - 1)],
		selectedCamera: '',
		cameras: [],
	}),
	computed: {
		emojiTransform: function() {
			return `translate(${this.emojiX}%, ${this.emojiY}%)`;
		},
	},
	methods: {
		cameraChange: function() {
			loadVideo(this.selectedCamera);
		},
	},
	mounted: function() {
		bindPage.call(this);
	},
};
</script>

<style>
body {
	margin: 0;
	width: 100%;
}
.emoji {
	position: absolute;
	color: white;
	font-size: 50px;
	transform: translate(-50%, -50%);
}
.emoji-container {
	position: absolute;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	transition-property: transform;
	transition-duration: 0.1s;
	transition-timing-function: linear;
}
video {
	width: 100vw;
	height: 100vh;
	object-fit: cover;
}
select {
	position: absolute;
	right: 0;
}
</style>
