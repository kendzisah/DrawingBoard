<script>
  import { onMount } from "svelte";
  let localStream;
  let remoteStream;
  let peerConnection;
  let socket;
  let isCaller = false;

  const config = {
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
  };

  onMount(() => {
    socket = new WebSocket("ws://localhost:8080");

    socket.onmessage = async (event) => {
      const data = JSON.parse(event.data);

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

        socket.onopen = () => {
          isCaller = true;
          startWebRTC();
        };
      });
  }

  async function startWebRTC() {
    peerConnection = new RTCPeerConnection(config);
    peerConnection.addStream(localStream);

    peerConnection.onaddstream = (event) => {
      remoteStream = event.stream;
      document.getElementById("remoteVideo").srcObject = remoteStream;
    };

    peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        socket.send(JSON.stringify({ candidate: event.candidate }));
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
  }

  async function handleCandidate(candidate) {
    await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
  }

  function animateConnection() {
    const connectionStatus = document.getElementById("connectionStatus");
    connectionStatus.animate(
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
  }
</script>

<div>
  <h2>WebRTC Video Chat</h2>
  <div
    id="connectionStatus"
    on:click={animateConnection}
    on:keydown={animateConnection}
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
