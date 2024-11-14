<script setup lang="ts">
import { ref, onUnmounted } from "vue";
import { Mic, MicOff, Phone, PhoneOff, Video, VideoOff } from "lucide-vue-next";

const configuration = {
  iceServers: [
    { urls: "stun:stun.l.google.com:19302" },
    { urls: "stun:stun1.l.google.com:19302" }
  ]
};

const ENABLE_VIDEO = true;
const isHost = ref(false);
const isMuted = ref(false);
const isVideoEnabled = ref(true);
const isCallActive = ref(false);
const connectionCode = ref("");
const inputCode = ref("");
const connectionStatus = ref("");
const waitingForAnswer = ref(false);
const localVideoRef = ref<HTMLVideoElement | null>(null);
const remoteVideoRef = ref<HTMLVideoElement | null>(null);

const localStream = ref<MediaStream | null>(null);
const peerConnection = ref<RTCPeerConnection | null>(null);
const iceCandidates = ref<RTCIceCandidate[]>([]);
const disconnectionTimeout = ref<number | null>(null);

const setupPeerConnectionListeners = (pc: RTCPeerConnection) => {
  pc.onicecandidate = (event) => {
    if (event.candidate) {
      iceCandidates.value.push(event.candidate);
      updateConnectionCode();
    }
  };

  pc.onconnectionstatechange = () => {
    if (disconnectionTimeout.value) {
      window.clearTimeout(disconnectionTimeout.value);
      disconnectionTimeout.value = null;
    }

    switch (pc.connectionState) {
      case "connected":
        connectionStatus.value = "Call connected successfully!";
        waitingForAnswer.value = false;
        break;
      case "disconnected":
        connectionStatus.value =
          "Peer disconnected. Waiting for reconnection...";
        disconnectionTimeout.value = window.setTimeout(() => {
          connectionStatus.value = "Call ended - peer disconnected";
          endCall();
        }, 10000);
        break;
      case "failed":
        connectionStatus.value = "Connection failed";
        endCall();
        break;
      case "closed":
        connectionStatus.value = "Call ended";
        endCall();
        break;
    }
  };

  pc.oniceconnectionstatechange = () => {
    if (pc.iceConnectionState === "disconnected") {
      connectionStatus.value = "Connection lost. Attempting to reconnect...";
    }
  };

  pc.ontrack = (event) => {
    if (ENABLE_VIDEO && remoteVideoRef.value) {
      remoteVideoRef.value.srcObject = event.streams[0];
      return;
    }

    const remoteStream = new MediaStream();
    event.streams[0].getTracks().forEach((track) => {
      remoteStream.addTrack(track);
    });
    const audioElement = new Audio();
    audioElement.srcObject = remoteStream;
    audioElement.play().catch(console.error);
  };
};

const updateConnectionCode = () => {
  const data = {
    sdp: peerConnection.value?.localDescription?.sdp,
    type: peerConnection.value?.localDescription?.type,
    candidates: iceCandidates.value
  };
  connectionCode.value = btoa(JSON.stringify(data));
};

const startCall = async () => {
  try {
    connectionStatus.value = "Initializing call...";
    const stream = await navigator.mediaDevices.getUserMedia({
      audio: true,
      video: ENABLE_VIDEO
    });
    localStream.value = stream;

    if (ENABLE_VIDEO && localVideoRef.value) {
      localVideoRef.value.srcObject = stream;
    }

    const pc = new RTCPeerConnection(configuration);
    peerConnection.value = pc;
    setupPeerConnectionListeners(pc);

    stream.getTracks().forEach((track) => {
      pc.addTrack(track, stream);
    });

    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);

    updateConnectionCode();
    isHost.value = true;
    isCallActive.value = true;
    waitingForAnswer.value = true;
    connectionStatus.value = "Waiting for peer to join...";
  } catch (err) {
    console.error("Error starting call:", err);
    connectionStatus.value = "Failed to start call: " + (err as Error).message;
    endCall();
  }
};

const processAnswerCode = async () => {
  try {
    if (!peerConnection.value || !inputCode.value) return;

    const { sdp, type, candidates } = JSON.parse(atob(inputCode.value));

    if (!sdp || !type) {
      throw new Error("Invalid answer format");
    }

    await peerConnection.value.setRemoteDescription({ sdp, type });

    for (const candidate of candidates) {
      await peerConnection.value.addIceCandidate(
        new RTCIceCandidate(candidate)
      );
    }

    connectionStatus.value = "Processing answer...";
    inputCode.value = "";
  } catch (err) {
    console.error("Error processing answer:", err);
    connectionStatus.value =
      "Failed to process answer: " + (err as Error).message;
    endCall();
  }
};

const joinCall = async () => {
  try {
    connectionStatus.value = "Joining call...";
    const stream = await navigator.mediaDevices.getUserMedia({
      audio: true,
      video: ENABLE_VIDEO
    });
    localStream.value = stream;

    if (ENABLE_VIDEO && localVideoRef.value) {
      localVideoRef.value.srcObject = stream;
    }

    const pc = new RTCPeerConnection(configuration);
    peerConnection.value = pc;
    setupPeerConnectionListeners(pc);

    stream.getTracks().forEach((track) => {
      pc.addTrack(track, stream);
    });

    const { sdp, type, candidates } = JSON.parse(atob(inputCode.value));

    if (!sdp || !type) {
      throw new Error("Invalid offer format");
    }

    await pc.setRemoteDescription({ sdp, type });

    for (const candidate of candidates) {
      await pc.addIceCandidate(new RTCIceCandidate(candidate));
    }

    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);

    const answerData = {
      sdp: pc.localDescription?.sdp,
      type: pc.localDescription?.type,
      candidates: iceCandidates.value
    };
    connectionCode.value = btoa(JSON.stringify(answerData));
    isCallActive.value = true;
    connectionStatus.value = "Connected to call";
  } catch (err) {
    console.error("Error joining call:", err);
    connectionStatus.value = "Failed to join call: " + (err as Error).message;
    endCall();
  }
};

const endCall = () => {
  if (disconnectionTimeout.value) {
    window.clearTimeout(disconnectionTimeout.value);
    disconnectionTimeout.value = null;
  }

  localStream.value?.getTracks().forEach((track) => {
    track.stop();
    track.enabled = false;
  });

  if (peerConnection.value) {
    peerConnection.value.getSenders().forEach((sender) => {
      if (sender.track) {
        sender.track.stop();
      }
    });
    peerConnection.value.close();
    peerConnection.value = null;
  }

  if (ENABLE_VIDEO) {
    if (localVideoRef.value) {
      localVideoRef.value.srcObject = null;
    }
    if (remoteVideoRef.value) {
      remoteVideoRef.value.srcObject = null;
    }
  }

  isCallActive.value = false;
  isHost.value = false;
  waitingForAnswer.value = false;
  connectionCode.value = "";
  inputCode.value = "";
  iceCandidates.value = [];

  if (!connectionStatus.value.includes("ended")) {
    connectionStatus.value = "Call ended";
  }
};

const toggleMute = () => {
  if (localStream.value) {
    localStream.value.getAudioTracks().forEach((track) => {
      track.enabled = !track.enabled;
    });
    isMuted.value = !isMuted.value;
  }
};

const toggleVideo = () => {
  if (localStream.value) {
    localStream.value.getVideoTracks().forEach((track) => {
      track.enabled = !track.enabled;
    });
    isVideoEnabled.value = !isVideoEnabled.value;
  }
};

onUnmounted(() => {
  endCall();
});
</script>

<template>
  <div
    class="min-h-screen bg-gradient-to-br from-indigo-100 to-purple-100 flex items-center justify-center p-4"
  >
    <div class="bg-white rounded-2xl shadow-xl p-8 w-full max-w-4xl">
      <h1 class="text-3xl font-bold text-center mb-8 text-indigo-900">
        {{ ENABLE_VIDEO ? "Video" : "Audio" }} Call
      </h1>

      <div v-show="!isCallActive" class="space-y-6">
        <button
          @click="startCall"
          class="w-full py-3 px-4 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 transition-colors flex items-center justify-center gap-2 font-medium"
        >
          <Phone :size="20" />
          Start New Call
        </button>

        <div class="relative">
          <div class="absolute inset-0 flex items-center">
            <div class="w-full border-t border-gray-300" />
          </div>
          <div class="relative flex justify-center text-sm">
            <span class="px-2 bg-white text-gray-500">Or join a call</span>
          </div>
        </div>

        <div class="space-y-4">
          <input
            type="text"
            v-model="inputCode"
            placeholder="Enter connection code"
            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
          />
          <button
            @click="joinCall"
            :disabled="!inputCode"
            class="w-full py-3 px-4 bg-indigo-100 text-indigo-700 rounded-lg hover:bg-indigo-200 transition-colors flex items-center justify-center gap-2 font-medium disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Join Call
          </button>
        </div>
      </div>

      <div v-show="isCallActive" class="space-y-6">
        <div
          v-if="connectionStatus"
          :class="[
            'p-4 rounded-lg text-sm',
            connectionStatus.includes('Failed') ||
            connectionStatus.includes('ended')
              ? 'bg-red-50 text-red-700'
              : connectionStatus.includes('success')
              ? 'bg-green-50 text-green-700'
              : 'bg-blue-50 text-blue-700'
          ]"
        >
          {{ connectionStatus }}
        </div>

        <div v-if="ENABLE_VIDEO" class="grid grid-cols-2 gap-4">
          <div class="relative">
            <video
              ref="localVideoRef"
              autoplay
              muted
              playsinline
              class="w-full rounded-lg bg-gray-900"
            ></video>
            <span
              class="absolute bottom-2 left-2 text-white text-sm bg-black/50 px-2 py-1 rounded"
            >
              You
            </span>
          </div>
          <div class="relative">
            <video
              ref="remoteVideoRef"
              autoplay
              playsinline
              class="w-full rounded-lg bg-gray-900"
            ></video>
            <span
              class="absolute bottom-2 left-2 text-white text-sm bg-black/50 px-2 py-1 rounded"
            >
              Remote
            </span>
          </div>
        </div>

        <div v-if="connectionCode" class="p-4 bg-gray-50 rounded-lg">
          <p class="text-sm text-gray-600 mb-2">
            {{
              isHost
                ? "Share this code to let others join:"
                : "Share this answer code with the host:"
            }}
          </p>
          <p
            class="text-xs bg-white p-3 rounded border border-gray-200 break-all"
          >
            {{ connectionCode }}
          </p>
        </div>

        <div v-if="isHost && waitingForAnswer" class="space-y-4">
          <input
            type="text"
            v-model="inputCode"
            placeholder="Paste answer code here"
            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
          />
          <button
            @click="processAnswerCode"
            :disabled="!inputCode"
            class="w-full py-3 px-4 bg-indigo-100 text-indigo-700 rounded-lg hover:bg-indigo-200 transition-colors flex items-center justify-center gap-2 font-medium disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Complete Connection
          </button>
        </div>

        <div class="flex justify-center gap-4">
          <button
            @click="toggleMute"
            :class="[
              'p-4 rounded-full transition-opacity hover:opacity-90',
              isMuted ? 'bg-red-100 text-red-600' : 'bg-gray-100 text-gray-600'
            ]"
          >
            <component :is="isMuted ? MicOff : Mic" :size="24" />
          </button>

          <button
            v-if="ENABLE_VIDEO"
            @click="toggleVideo"
            :class="[
              'p-4 rounded-full transition-opacity hover:opacity-90',
              !isVideoEnabled
                ? 'bg-red-100 text-red-600'
                : 'bg-gray-100 text-gray-600'
            ]"
          >
            <component :is="!isVideoEnabled ? VideoOff : Video" :size="24" />
          </button>

          <button
            @click="endCall"
            class="p-4 rounded-full bg-red-500 text-white hover:bg-red-600 transition-colors"
          >
            <PhoneOff :size="24" />
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
