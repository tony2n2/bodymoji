<template>
	<div id="app" :style="{ transform: scale }">
		<video id="video" playsinline></video>
		<div class="emoji-container">
			<div class="emoji" :style="{ left: emojiX, top: emojiY }">{{ emoji }}</div>
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
	cameraChanged: false,
};

async function getVideoInputs() {
	if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
		console.log('enumerateDevices() not supported.');
		return [];
	}
	// call getUserMedia before enumerateDevices
	await navigator.mediaDevices.getUserMedia({ audio: false, video: true });
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
		throw e;
	}

	state.video.play();
}

function segmentPerson() {
	async function bodySegmentationFrame() {
		if (state.cameraChanged) {
			state.net = await bodyPix.load();
			state.cameraChanged = false;
		}
		const personPart = await state.net.segmentPersonParts(state.video, {
			flipHorizontal: false,
			internalResolution: 'medium',
			segmentationThreshold: 0.7,
		});
		if (personPart.allPoses.length && personPart.allPoses[0].score > 0.4) {
			let part = personPart.allPoses[0].keypoints[0];
			this.$set(this, 'emojiX', Math.round(part.position.x) + 'px');
			this.$set(this, 'emojiY', Math.round(part.position.y) + 'px');
		}
		// if (state.frame < 100) {
		// state.frame++;
		requestAnimationFrame(bodySegmentationFrame.bind(this));
		// }
	}

	bodySegmentationFrame.call(this);
}

function rescale() {
	let wScale = document.documentElement.clientWidth / 640;
	let hScale = document.documentElement.clientHeight / 480;
	let scale = wScale > hScale ? hScale : wScale;
	this.$set(this, 'scale', `scale(${scale})`);
}

async function bindPage() {
	state.net = await bodyPix.load();
	let cameras = await getVideoInputs();
	if (!cameras) return;
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
		scale: 1,
	}),
	computed: {
		emojiTransform: function() {
			return `translate(${this.emojiX}%, ${this.emojiY}%)`;
		},
	},
	methods: {
		cameraChange: function() {
			loadVideo(this.selectedCamera);
			state.cameraChanged = true;
		},
	},
	mounted: function() {
		bindPage.call(this);
		rescale.call(this);
		window.addEventListener('resize', rescale.bind(this));
	},
};
</script>

<style>
body {
	margin: 0;
	width: 100%;
}
.emoji-container {
	position: absolute;
	width: 640px;
	height: 480px;
}
.emoji {
	position: absolute;
	color: white;
	font-size: 50px;
	transform: translate(-50%, -50%);
	transition-property: top, left;
	transition-duration: 0.1s;
	transition-timing-function: linear;
}
select {
	position: absolute;
	top: 0;
}
#app {
	transform-origin: top;
	display: flex;
	justify-content: center;
	align-items: center;
}
</style>
