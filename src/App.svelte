<script lang="ts">
  import { initializeApp } from "firebase/app";
  import {
    getFirestore,
    addDoc,
    setDoc,
    updateDoc,
    doc,
    getDoc,
    onSnapshot,
    collection,
  } from "firebase/firestore";

  const firebaseConfig = {
    apiKey: "AIzaSyDLoxtIPrM0YwBybxru3kgJZCTlE1OWW4w",
    authDomain: "live-yunusemre-dev.firebaseapp.com",
    databaseURL:
      "https://live-yunusemre-dev-default-rtdb.europe-west1.firebasedatabase.app",
    projectId: "live-yunusemre-dev",
    storageBucket: "live-yunusemre-dev.appspot.com",
    messagingSenderId: "535277851221",
    appId: "1:535277851221:web:fb16bccf124a2ae3a3fadc",
  };

  const servers = {
    iceServers: [
      {
        urls: [
          "stun:stun1.l.google.com:19302",
          "stun:stun2.l.google.com:19302",
        ],
      },
    ],
    iceCandidatePoolSize: 10,
  };

  const app = initializeApp(firebaseConfig);
  const firestore = getFirestore(app);

  const pc = new RTCPeerConnection(servers);

  let localStream: MediaStream;
  let remoteStream: MediaStream;
  let webcamVideo: HTMLVideoElement;
  let remoteVideo: HTMLVideoElement;
  let callButton: HTMLButtonElement;
  let callInput: HTMLInputElement;
  let answerButton: HTMLButtonElement;
  let webcamButton: HTMLButtonElement;

  const startLocalWebcam = async () => {
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true,
    });
    remoteStream = new MediaStream();

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

    webcamVideo.srcObject = localStream;
    remoteVideo.srcObject = remoteStream;

    callButton.disabled = false;
    answerButton.disabled = false;
    webcamButton.disabled = true;
  };

  const createOffer = async () => {
    // Reference Firestore collections for signaling
    const callsCollection = collection(firestore, "calls");
    const callsDoc = await addDoc(callsCollection, {});
    const offerCandidates = collection(callsDoc, "offerCandidates");
    const answerCandidates = collection(callsDoc, "answerCandidates");

    callInput.value = callsDoc.id;

    // Get candidates for caller, save to db
    pc.onicecandidate = (event) => {
      event.candidate && addDoc(offerCandidates, event.candidate.toJSON());
    };

    // Create offer
    const offerDescription = await pc.createOffer();
    await pc.setLocalDescription(offerDescription);

    const offer = {
      sdp: offerDescription.sdp,
      type: offerDescription.type,
    };

    await setDoc(callsDoc, { offer });

    // Listen for remote answer
    onSnapshot(callsDoc, (snapshot) => {
      const data = snapshot.data();
      if (!pc.currentRemoteDescription && data?.answer) {
        const answerDescription = new RTCSessionDescription(data.answer);
        pc.setRemoteDescription(answerDescription);
      }
    });

    // When answered, add candidate to peer connection
    onSnapshot(answerCandidates, (snapshot) => {
      snapshot.docChanges().forEach((change) => {
        if (change.type === "added") {
          const candidate = new RTCIceCandidate(change.doc.data());
          pc.addIceCandidate(candidate);
        }
      });
    });
  };

  const answerOffer = async () => {
    const callId = callInput.value;
    const callsCollection = collection(firestore, "calls");
    const callDoc = doc(callsCollection, callId);
    const docSnap = await getDoc(callDoc);
    const offerCandidates = collection(callDoc, "offerCandidates");
    const answerCandidates = collection(callDoc, "answerCandidates");

    pc.onicecandidate = (event) => {
      event.candidate && addDoc(answerCandidates, event.candidate.toJSON());
    };

    const callData = docSnap.data();

    const offerDescription = callData?.offer;
    await pc.setRemoteDescription(new RTCSessionDescription(offerDescription));

    const answerDescription = await pc.createAnswer();
    await pc.setLocalDescription(answerDescription);

    const answer = {
      type: answerDescription.type,
      sdp: answerDescription.sdp,
    };

    await updateDoc(callDoc, { answer });

    onSnapshot(offerCandidates, (snapshot) => {
      snapshot.docChanges().forEach((change) => {
        console.log(change);
        if (change.type === "added") {
          let data = change.doc.data();
          pc.addIceCandidate(new RTCIceCandidate(data));
        }
      });
    });
  };
</script>

<main>
  <div class="videos">
    <video
      id="webcamVideo"
      muted
      autoplay
      playsinline
      bind:this={webcamVideo}
    />
    <video id="remoteVideo" autoplay playsinline bind:this={remoteVideo} />
  </div>

  <div class="buttons">
    <button
      id="webcamButton"
      bind:this={webcamButton}
      on:click={startLocalWebcam}
    >
      Start Webcam
    </button>
    <button
      id="callButton"
      disabled
      bind:this={callButton}
      on:click={createOffer}
    >
      Create Call
    </button>

    <div class="join">
      <input id="callInput" bind:this={callInput} />
      <button
        id="answerButton"
        disabled
        bind:this={answerButton}
        on:click={answerOffer}
      >
        Join Call
      </button>
    </div>
  </div>
</main>

<style>
  main {
    font-family: Arial, sans-serif;
    padding: 0 24px;
  }

  .videos {
    display: flex;
    justify-content: center;
    width: 100%;
    gap: 2px;
  }

  .videos video {
    background-color: #333;
    transition: height 0.5s;
  }

  .buttons {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 1rem;
    gap: 1rem;
  }

  #webcamVideo {
    border-radius: 2rem 0 0 2rem;
    overflow: hidden;
  }

  #remoteVideo {
    border-radius: 0 2rem 2rem 0;
    overflow: hidden;
  }

  #answerButton {
    margin: 0;
    border-radius: 0 8px 8px 0;
  }

  #callInput {
    padding: 0.5rem;
    font-size: 1rem;
    height: 18px;
    border-color: transparent;
    border-radius: 8px 0 0 8px;
  }

  .join {
    height: fit-content;
    display: flex;
  }

  video {
    width: 50%;
    aspect-ratio: 4 / 3;
  }

  button {
    margin: 1rem;
    padding: 0.5rem 1rem;
    font-size: 1rem;
    cursor: pointer;
  }
</style>
