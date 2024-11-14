<template>
  <div>
    <h1>WebRTC Audio Call</h1>
    <button @click="startCall">Start Call</button>
    <button @click="joinCall">Join Call</button>
    <audio ref="remoteAudio" autoplay></audio>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  setup() {
    const remoteAudio = ref(null);
    const servers = { iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] };
    let pc = new RTCPeerConnection(servers);

    pc.ontrack = (event) => {
      remoteAudio.value.srcObject = event.streams[0];
    };

    pc.onicecandidate = (event) => {
      if (event.candidate) {
        console.log('New ICE candidate:', JSON.stringify(event.candidate));
      }
    };

    const startCall = async () => {
      const localStream = await navigator.mediaDevices.getUserMedia({
        audio: true,
      });
      localStream
        .getTracks()
        .forEach((track) => pc.addTrack(track, localStream));

      const offerDescription = await pc.createOffer();
      await pc.setLocalDescription(offerDescription);

      console.log('Offer SDP:', JSON.stringify(pc.localDescription));
    };

    const joinCall = async () => {
      const offer = prompt('Paste the offer SDP:');
      if (!offer) return;

      const offerDescription = new RTCSessionDescription(JSON.parse(offer));
      await pc.setRemoteDescription(offerDescription);

      const localStream = await navigator.mediaDevices.getUserMedia({
        audio: true,
      });
      localStream
        .getTracks()
        .forEach((track) => pc.addTrack(track, localStream));

      const answerDescription = await pc.createAnswer();
      await pc.setLocalDescription(answerDescription);

      console.log('Answer SDP:', JSON.stringify(pc.localDescription));
    };

    // Function to add ICE candidates received manually
    const addIceCandidate = async () => {
      const iceCandidate = prompt('Paste the ICE candidate:');
      if (!iceCandidate) return;

      try {
        await pc.addIceCandidate(new RTCIceCandidate(JSON.parse(iceCandidate)));
        console.log('Added ICE candidate');
      } catch (error) {
        console.error('Error adding received ice candidate', error);
      }
    };

    return {
      startCall,
      joinCall,
      remoteAudio,
      addIceCandidate,
    };
  },
};
</script>
