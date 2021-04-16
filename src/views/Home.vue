<template>
  <div class="home">
    <div>
      <video
        id="mainVideo"
        autoplay
        :muted="main.muted"
        :srcObject="main.srcObject"
      ></video>
    </div>
    <div>
      <button v-on:click="initVideo">init video</button>
      <button v-on:click="handleMute">unmute</button>
    </div>
    <div>
      <button v-on:click="handleRemoveVideo">remove video</button>
      <button v-on:click="handleMute">Mute</button>
    </div>
    <div>
      <button v-on:click="makeCall">Make Call</button>
    </div>
    <div>
      <input v-model="callCode" />
      <button v-on:click="handleAnswer">Answer Call</button>
    </div>
    <div>
      <video id="remoteVideo" autoplay :srcObject="remote.srcObject"></video>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import firebase from "firebase/app";
import "firebase/firestore";
const firebaseConfig = {
  apiKey: "AIzaSyBqPemaSudJ5bq9IFJ-JlOxoVxgeEMeaYY",
  authDomain: "test-webrtc-eb4bc.firebaseapp.com",
  projectId: "test-webrtc-eb4bc",
  storageBucket: "test-webrtc-eb4bc.appspot.com",
  messagingSenderId: "134950353841",
  appId: "1:134950353841:web:bc3f9f6b6f37b0f1f85b90",
  measurementId: "G-9SHG1WM6NJ",
};

if (!firebase.apps.length) {
  firebase.initializeApp(firebaseConfig);
}
const firestore = firebase.firestore();
const servers = {
  iceServers: [
    {
      urls: ["stun:stun1.l.google.com:19302", "stun:stun2.l.google.com:19302"],
    },
  ],
  iceCandidatePoolSize: 10,
};
const pc = new RTCPeerConnection(servers);

export default defineComponent({
  name: "Home",
  data() {
    return {
      video: null as HTMLVideoElement | null,
      main: {
        srcObject: null as MediaStream | null,
        muted: false,
      },
      remote: {
        srcObject: null as MediaStream | null,
        muted: false,
      },
      stream: "",
      remoteVideo: "" as any,
      localStream: null as any,
      remoteStream: null as any,
      muted: false,
      peerConn: "",
      connecteduser: null,
      callCode: "",
    };
  },
  mounted() {
    this.initVideo();
  },
  methods: {
    async handleRemoveVideo() {
      if (this.main.srcObject) {
        let stream: any = this.main.srcObject;
        stream.getVideoTracks().forEach((track: MediaStreamTrack) => {
          track.enabled = false;
          track.stop();
        });
      }
    },
    async initVideo() {
      const localStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true,
      });
      const remoteStream = new MediaStream();

      // Push tracks from local stream to peer connection
      localStream.getTracks().forEach((track) => {
        pc.addTrack(track, localStream);
      });

      // Pull tracks from remote stream, add to video stream
      pc.ontrack = (event) => {
        event.streams[0].getTracks().forEach((track) => {
          remoteStream.addTrack(track);
        });
      };

      this.main.srcObject = localStream;
      this.remote.srcObject = remoteStream;
    },
    handleMute() {
      this.main.muted = !this.main.muted;
    },
    async makeCall() {
      const remoteStream = new MediaStream();
      pc.ontrack = (event: any) => {
        event.streams[0].getTracks().forEach((track: any) => {
          remoteStream.addTrack(track);
        });
      };

      this.remote.srcObject = remoteStream;
      const callDoc = firestore.collection("calls").doc();
      const offerCandidates = callDoc.collection("offerCandidates");
      const answerCandidates = callDoc.collection("answerCandidates");

      this.callCode = callDoc.id;

      // Get candidates for caller, save to db
      pc.onicecandidate = (event) => {
        event.candidate && offerCandidates.add(event.candidate.toJSON());
      };

      // Create offer
      const offerDescription = await pc.createOffer();
      await pc.setLocalDescription(offerDescription);

      const offer = {
        sdp: offerDescription.sdp,
        type: offerDescription.type,
      };

      await callDoc.set({ offer });

      // Listen for remote answer
      callDoc.onSnapshot((snapshot) => {
        const data = snapshot.data();
        if (!pc.currentRemoteDescription && data?.answer) {
          const answerDescription = new RTCSessionDescription(data.answer);
          pc.setRemoteDescription(answerDescription);
        }
      });

      // When answered, add candidate to peer connection
      answerCandidates.onSnapshot((snapshot) => {
        snapshot.docChanges().forEach((change) => {
          if (change.type === "added") {
            const candidate = new RTCIceCandidate(change.doc.data());
            pc.addIceCandidate(candidate);
          }
        });
      });
    },
    async handleAnswer() {
      const callId = this.callCode;
      const callDoc = firestore.collection("calls").doc(callId);
      const answerCandidates = callDoc.collection("answerCandidates");
      const offerCandidates = callDoc.collection("offerCandidates");

      pc.onicecandidate = (event) => {
        event.candidate && answerCandidates.add(event.candidate.toJSON());
      };

      const callData = (await callDoc.get()).data();

      const offerDescription = callData!.offer;
      await pc.setRemoteDescription(
        new RTCSessionDescription(offerDescription)
      );

      const answerDescription = await pc.createAnswer();
      await pc.setLocalDescription(answerDescription);

      const answer = {
        type: answerDescription.type,
        sdp: answerDescription.sdp,
      };

      await callDoc.update({ answer });

      offerCandidates.onSnapshot((snapshot: any) => {
        console.log("snapshot", snapshot.docChanges());
        snapshot.docChanges().forEach((change: any) => {
          console.log("cahnge", change);
          if (change.type === "added") {
            let data = change.doc.data();
            pc.addIceCandidate(new RTCIceCandidate(data));
          }
        });
      });
    },
  },
});
</script>
