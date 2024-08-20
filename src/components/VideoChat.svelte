<script>
  import { onMount } from "svelte";
  let localStream;
  let remoteStream;
  let peerConnection;
  let socket;
  let isCaller = false;
  let connectionStatus = false;

  const config = {
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
  };

  onMount(() => {
    socket = new WebSocket(
      "wss://whiteboard-server-production-4ec9.up.railway.app"
    );

    socket.onmessage = async (event) => {
      const data = JSON.parse(event.data);
      console.log("Received message via WebSocket:", data);

      if (data.offer) {
        await handleOffer(data.offer);
      } else if (data.answer) {
        await handleAnswer(data.answer);
      } else if (data.candidate) {
        await handleCandidate(data.candidate);
      }
    };

    startLocalVideo();
  });

  function startLocalVideo() {
    navigator.mediaDevices
      .getUserMedia({ video: true, audio: true })
      .then((stream) => {
        localStream = stream;
        document.getElementById("localVideo").srcObject = stream;
      });
  }

  function initiateCall() {
    isCaller = true;
    startWebRTC();
    animateConnection();
  }

  async function startWebRTC() {
    if (!peerConnection) {
      peerConnection = new RTCPeerConnection(config);
    }

    peerConnection.addStream(localStream);

    peerConnection.onaddstream = (event) => {
      console.log("Remote stream added:", event.stream);
      remoteStream = event.stream;
      document.getElementById("remoteVideo").srcObject = remoteStream;
    };

    peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        console.log("Sending ICE candidate:", event.candidate);
        socket.send(JSON.stringify({ candidate: event.candidate }));
      }
    };

    peerConnection.onconnectionstatechange = () => {
      if (peerConnection.connectionState === "connected") {
        updateConnectionStatus(true);
      } else if (
        peerConnection.connectionState === "disconnected" ||
        peerConnection.connectionState === "failed"
      ) {
        updateConnectionStatus(false);
      } else if (peerConnection.connectionState === "connecting") {
        updateConnectionStatus(true);
      }
    };

    if (isCaller) {
      const offer = await peerConnection.createOffer();
      await peerConnection.setLocalDescription(offer);
      socket.send(JSON.stringify({ offer: offer }));
    }
  }

  async function handleOffer(offer) {
    if (!peerConnection) startWebRTC();

    await peerConnection.setRemoteDescription(new RTCSessionDescription(offer));
    const answer = await peerConnection.createAnswer();
    await peerConnection.setLocalDescription(answer);

    socket.send(JSON.stringify({ answer: answer }));
  }

  async function handleAnswer(answer) {
    await peerConnection.setRemoteDescription(
      new RTCSessionDescription(answer)
    );

    updateConnectionStatus(true);
  }

  async function handleCandidate(candidate) {
    console.log("Received ICE candidate:", candidate);
    await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
  }

  function animateConnection() {
    const connectionElement = document.getElementById("connectionStatus");

    if (connectionStatus) {
      // Start the animation
      connectionElement.animate(
        [
          { transform: "scale(1)", opacity: 1 },
          { transform: "scale(1.2)", opacity: 0.7 },
          { transform: "scale(1)", opacity: 1 },
        ],
        {
          duration: 1000,
          iterations: Infinity,
        }
      );
    } else {
      stopAnimation();
    }
  }

  function stopAnimation() {
    const connectionElement = document.getElementById("connectionStatus");
    const animations = connectionElement.getAnimations();
    animations.forEach((animation) => animation.cancel());
  }

  function updateConnectionStatus(isConnected) {
    connectionStatus = isConnected;
    const connectionElement = document.getElementById("connectionStatus");

    if (isConnected) {
      // Set the background color to green and start the animation
      connectionElement.style.backgroundColor = "green";
      animateConnection();
    } else {
      // Set the background color to red and stop the animation
      connectionElement.style.backgroundColor = "red";
      stopAnimation();
    }
  }
</script>

<div>
  <h2>WebRTC Video Chat</h2>
  <div
    id="connectionStatus"
    on:click={initiateCall}
    on:keydown={initiateCall}
    aria-label="Connection Status"
    role="button"
    tabindex="0"
  ></div>
  <video id="localVideo" autoplay>
    <track kind="captions" />
  </video>
  <video id="remoteVideo" autoplay>
    <track kind="captions" />
  </video>
</div>

<style>
  #connectionStatus {
    width: 100px;
    height: 100px;
    background-color: green;
    border-radius: 50%;
    margin: 10px auto;
  }

  video {
    width: 100%;
    max-width: 300px;
  }

  h2 {
    color: white;
  }
</style>
