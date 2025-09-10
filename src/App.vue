<template>
  <div class="app">
    <!-- Auth Screen -->
    <div v-if="!isAuthenticated" class="auth-container">
      <div class="auth-form">
        <h2>Video Call App</h2>
        <div class="tabs">
          <button
            @click="authMode = 'login'"
            :class="{ active: authMode === 'login' }"
          >
            Login
          </button>
          <button
            @click="authMode = 'register'"
            :class="{ active: authMode === 'register' }"
          >
            Register
          </button>
        </div>

        <form @submit.prevent="authenticate">
          <input v-model="username" placeholder="Username" required />
          <button type="submit" :disabled="loading">
            {{ authMode === "login" ? "Login" : "Register" }}
          </button>
        </form>

        <div v-if="error" class="error">{{ error }}</div>
      </div>
    </div>

    <!-- Main App -->
    <div v-else class="main-container">
      <!-- Call Interface -->
      <div v-if="currentView === 'call-interface'" class="call-interface">
        <div class="user-info">
          <h3>Welcome, {{ currentUser.username }}</h3>
          <p>Your ID: {{ currentUser.userId }}</p>
        </div>

        <div class="call-section">
          <h4>Make a Call</h4>
          <div class="call-input">
            <input v-model="calleeId" placeholder="Enter user ID to call" />
            <button @click="initiateCall" :disabled="!calleeId || calling">
              Call
            </button>
          </div>
        </div>

        <div class="contacts-section">
          <h4>Contacts</h4>
          <input v-model="searchQuery" placeholder="Search contacts..." />
          <div class="contacts-list">
            <div
              v-for="user in filteredUsers"
              :key="user.id"
              class="contact-item"
              @click="callUser(user.id)"
            >
              <div class="contact-info">
                <div class="contact-name">{{ user.username }}</div>
                <div class="contact-id">{{ user.id }}</div>
              </div>
              <div :class="['status', user.online ? 'online' : 'offline']">
                {{ user.online ? "Online" : "Offline" }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Outgoing Call -->
      <div v-else-if="currentView === 'outgoing-call'" class="call-screen">
        <div class="call-info">
          <h3>Calling...</h3>
          <p>{{ calleeInfo.username || calleeId }}</p>
        </div>

        <!-- Show local video during outgoing call -->
        <div class="outgoing-video-container">
          <video
            ref="localVideo"
            autoplay
            muted
            playsinline
            webkit-playsinline
            class="outgoing-local-video"
          ></video>
        </div>

        <button @click="endCall" class="end-call-btn">📞 End Call</button>
      </div>

      <!-- Incoming Call -->
      <div v-else-if="currentView === 'incoming-call'" class="call-screen">
        <div class="call-info">
          <h3>Incoming Call</h3>
          <p>{{ incomingCall.callerName }}</p>
        </div>
        <div class="call-actions">
          <button @click="acceptCall" class="accept-btn">✅ Accept</button>
          <button @click="rejectCall" class="reject-btn">❌ Reject</button>
        </div>
      </div>

      <!-- Video Call -->
      <div v-else-if="currentView === 'video-call'" class="video-call">
        <div class="video-container">
          <!-- Remote video (big) -->
          <div class="remote-wrap">
            <video
              ref="remoteVideo"
              autoplay
              playsinline
              webkit-playsinline
              class="remote-video"
            ></video>

            <!-- Local preview (small) -->
            <video
              ref="localVideo"
              autoplay
              muted
              playsinline
              webkit-playsinline
              class="local-video"
            ></video>

            <!-- Play button overlay if browser blocked autoplay -->
            <div v-if="showRemotePlayButton" class="play-overlay">
              <button @click="playRemote" class="play-btn">
                Click to play
              </button>
            </div>
          </div>
        </div>

        <div class="call-actions-bottom">
          <button @click="endCall" class="end-call-btn">📞 End Call</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, watch } from "vue";
import { io } from "socket.io-client";

/* ---------- Reactive state ---------- */
const isAuthenticated = ref(false);
const authMode = ref("login");
const username = ref("");
const loading = ref(false);
const error = ref("");
const currentUser = reactive({ username: "", userId: "" });
const currentView = ref("call-interface");
const calleeId = ref("");
const searchQuery = ref("");
const users = ref([]);
const calling = ref(false);
const incomingCall = ref(null);
const calleeInfo = ref({});
const currentCallId = ref(null);

/* ---------- DOM refs ---------- */
const localVideo = ref(null);
const remoteVideo = ref(null);
const showRemotePlayButton = ref(false);

/* ---------- WebRTC / Socket state ---------- */
let localStream = null;
let peerConnection = null;
let socket = null;
let pendingRemoteCandidates = []; // queue ICE until remoteDescription exists
let remoteMediaStream = null; // persist remote stream even if element not mounted yet
let callTimeout = null; // timeout for call establishment

/* ---------- helpers ---------- */
const filteredUsers = computed(() =>
  users.value.filter(
    (u) =>
      u.id !== currentUser.userId &&
      u.username.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
);

/* ---------- Create peer connection and handlers ---------- */
const createPeerConnectionIfNeeded = () => {
  if (peerConnection) return;

  peerConnection = new RTCPeerConnection({
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
  });

  peerConnection.onicecandidate = (ev) => {
    if (ev.candidate && socket) {
      console.log("Sending ICE candidate", ev.candidate);
      socket.emit("ice-candidate", {
        callId: currentCallId.value,
        candidate: ev.candidate,
      });
    }
  };

  peerConnection.ontrack = (ev) => {
    console.log("ontrack event streams:", ev.streams);
    console.log("ontrack event track:", ev.track);
    const stream = ev.streams && ev.streams[0] ? ev.streams[0] : null;
    if (stream) {
      // persist and attach when possible
      remoteMediaStream = stream;
      console.log("Remote stream received:", stream);
      if (remoteVideo.value && remoteVideo.value.srcObject !== stream) {
        remoteVideo.value.srcObject = stream;
        remoteVideo.value.playsInline = true;
        remoteVideo.value.play().catch((err) => {
          console.warn("remote.play() blocked:", err);
          showRemotePlayButton.value = true;
        });
        console.log("Remote video attached and playing");
      } else {
        console.log(
          "Remote video element not ready, stream will be attached via watcher"
        );
      }
    } else {
      // fallback: create MediaStream from track
      const fallbackStream = new MediaStream([ev.track]);
      remoteMediaStream = fallbackStream;
      console.log("Fallback remote stream created:", fallbackStream);
      if (remoteVideo.value && remoteVideo.value.srcObject !== fallbackStream) {
        remoteVideo.value.srcObject = fallbackStream;
        remoteVideo.value.playsInline = true;
        remoteVideo.value
          .play()
          .catch(() => (showRemotePlayButton.value = true));
        console.log("Fallback remote video attached and playing");
      } else {
        console.log(
          "Remote video element not ready, fallback stream will be attached via watcher"
        );
      }
    }
  };

  peerConnection.onconnectionstatechange = () => {
    console.log("PC state:", peerConnection.connectionState);
    if (
      ["disconnected", "failed", "closed"].includes(
        peerConnection.connectionState || ""
      )
    ) {
      console.log("Connection lost, cleaning up call");
      cleanupCall();
    }
  };

  // Monitor ICE connection state for better debugging
  peerConnection.oniceconnectionstatechange = () => {
    console.log("ICE connection state:", peerConnection.iceConnectionState);
    if (peerConnection.iceConnectionState === "failed") {
      console.log("ICE connection failed, attempting to restart ICE");
      // Try to restart ICE
      peerConnection.restartIce();
    }
  };

  // Monitor ICE gathering state
  peerConnection.onicegatheringstatechange = () => {
    console.log("ICE gathering state:", peerConnection.iceGatheringState);
  };
};

/* ---------- getUserMedia + attach to local video ---------- */
const startLocalStream = async () => {
  if (localStream) return localStream;
  try {
    // Mobile-optimized constraints
    const constraints = {
      video: {
        width: { ideal: 640, max: 1280 },
        height: { ideal: 480, max: 720 },
        facingMode: "user", // Front camera for mobile
        frameRate: { ideal: 15, max: 30 }, // Lower frame rate for mobile
      },
      audio: {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true,
      },
    };

    localStream = await navigator.mediaDevices.getUserMedia(constraints);
    console.log("Got local stream", localStream);
    console.log("Local stream tracks:", localStream.getTracks());

    // Attach to local video element if it exists
    if (localVideo.value) {
      console.log("Attaching local stream to video element");
      localVideo.value.srcObject = localStream;
      // local is muted to avoid echo
      localVideo.value.muted = true;
      localVideo.value.playsInline = true; // Important for iOS
      localVideo.value
        .play()
        .catch((e) => console.warn("localVideo.play() blocked:", e));
    } else {
      console.log(
        "Local video element not available yet, will attach via watcher"
      );
    }

    return localStream;
  } catch (err) {
    console.error("getUserMedia error:", err);
    // Show user-friendly error for mobile
    if (err.name === "NotAllowedError") {
      alert(
        "Camera and microphone access is required. Please allow permissions and try again."
      );
    } else if (err.name === "NotFoundError") {
      alert("No camera or microphone found. Please check your device.");
    } else {
      alert("Failed to access camera/microphone. Please try again.");
    }
    throw err;
  }
};

/* ---------- Drain queued ICE candidates ---------- */
const drainPendingCandidates = async () => {
  if (!peerConnection || !peerConnection.remoteDescription) return;
  for (const c of pendingRemoteCandidates) {
    try {
      await peerConnection.addIceCandidate(new RTCIceCandidate(c));
    } catch (err) {
      console.warn("failed to add queued candidate", err);
    }
  }
  pendingRemoteCandidates = [];
};

/* ---------- Socket initialization ---------- */
const initializeSocket = () => {
  socket = io("https://5d1f56feeee4.ngrok-free.app");

  socket.on("connect", () => {
    console.log("socket connected", socket.id);
    if (currentUser.userId) socket.emit("join", currentUser.userId);
  });

  socket.on("user-online", () => loadUsers());
  socket.on("user-offline", () => loadUsers());

  socket.on("call-initiated", ({ callId }) => {
    console.log("call-initiated -> callId", callId);
    currentCallId.value = callId;
  });

  socket.on("incoming-call", (data) => {
    console.log("incoming-call", data);
    incomingCall.value = data;
    currentCallId.value = data.callId;
    currentView.value = "incoming-call";
    // stow callee info for UI
    calleeInfo.value = { username: data.callerName, id: data.callerId };
  });

  socket.on("call-accepted", async (data) => {
    console.log("call-accepted", data);
    currentCallId.value = data.callId;

    // Clear call timeout
    if (callTimeout) {
      clearTimeout(callTimeout);
      callTimeout = null;
    }

    // On caller side: set remote description (answer) and drain queued ICE
    try {
      createPeerConnectionIfNeeded();
      await peerConnection.setRemoteDescription(
        new RTCSessionDescription(data.answer)
      );
      await drainPendingCandidates();

      // Switch to video call view
      currentView.value = "video-call";

      // Ensure videos play after view change (important for mobile)
      setTimeout(() => {
        if (localVideo.value && localVideo.value.srcObject) {
          localVideo.value.play().catch(console.warn);
        }
        if (remoteVideo.value && remoteVideo.value.srcObject) {
          remoteVideo.value.play().catch(console.warn);
        }
      }, 100);

      console.log("Call accepted, switched to video call view");
    } catch (err) {
      console.error("Error applying answer on caller:", err);
      alert("Failed to establish video call. Please try again.");
      cleanupCall();
    }
  });

  socket.on("call-rejected", () => {
    console.log("call rejected");
    calling.value = false;
    currentView.value = "call-interface";
    alert("Call was rejected or callee unavailable");
  });

  socket.on("call-ended", () => {
    console.log("call-ended received");
    cleanupCall();
  });

  socket.on("ice-candidate", async (data) => {
    // data: { callId, candidate } or candidate
    const candidate = data?.candidate ?? data;
    console.log("received ice-candidate", candidate);
    if (!peerConnection || !peerConnection.remoteDescription) {
      pendingRemoteCandidates.push(candidate);
      return;
    }
    try {
      await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
    } catch (err) {
      console.warn("Error adding ICE candidate:", err);
      // still okay to queue
      pendingRemoteCandidates.push(candidate);
    }
  });
};

/* ---------- Call flow: caller initiates ---------- */
const initiateCall = async () => {
  if (!calleeId.value) return;

  try {
    calling.value = true;
    currentView.value = "outgoing-call";

    // Start local stream first (important for mobile)
    const stream = await startLocalStream();

    // Create peer connection after getting stream
    createPeerConnectionIfNeeded();

    // Add tracks to peer connection
    stream.getTracks().forEach((t) => peerConnection.addTrack(t, stream));

    // Create offer with proper constraints for mobile
    const offer = await peerConnection.createOffer({
      offerToReceiveAudio: true,
      offerToReceiveVideo: true,
      voiceActivityDetection: true,
    });

    await peerConnection.setLocalDescription(offer);

    // Send offer to server (server will forward to callee)
    socket.emit("initiate-call", {
      callerId: currentUser.userId,
      calleeId: calleeId.value,
      offer: peerConnection.localDescription,
    });

    // Set timeout for call establishment (30 seconds)
    callTimeout = setTimeout(() => {
      if (currentView.value === "outgoing-call") {
        console.log("Call timeout - no answer received");
        alert("Call timed out. The other person may not be available.");
        cleanupCall();
      }
    }, 30000);

    console.log("Call initiated successfully");
  } catch (err) {
    console.error("initiateCall error:", err);
    alert("Failed to start call. Please try again.");
    cleanupCall();
  }
};

/* ---------- When user clicks contact ---------- */
const callUser = (userId) => {
  calleeId.value = userId;
  initiateCall();
};

/* ---------- Callee accepts the call ---------- */
const acceptCall = async () => {
  if (!incomingCall.value) return;

  // mark current call id
  currentCallId.value = incomingCall.value.callId;

  // IMPORTANT ORDER:
  // 1. create pc (to receive tracks)
  // 2. setRemoteDescription(offer)
  // 3. getUserMedia and add local tracks
  // 4. createAnswer & setLocalDescription
  // 5. send answer

  try {
    createPeerConnectionIfNeeded();

    // Step 2 - set remote (the caller's offer)
    await peerConnection.setRemoteDescription(
      new RTCSessionDescription(incomingCall.value.offer)
    );

    // Step 3 - get local stream and add tracks
    // For mobile, we need to ensure this happens after user interaction
    const stream = await startLocalStream();
    stream.getTracks().forEach((t) => peerConnection.addTrack(t, stream));

    // Step 4 - create answer and set local
    const answer = await peerConnection.createAnswer();
    await peerConnection.setLocalDescription(answer);

    // Step 5 - send answer back to caller
    socket.emit("accept-call", {
      callId: incomingCall.value.callId,
      answer: peerConnection.localDescription,
    });

    // Drain queued candidates (if any)
    await drainPendingCandidates();

    // Switch to video call view FIRST
    currentView.value = "video-call";

    // Wait for DOM update, then ensure videos play
    await new Promise((resolve) => setTimeout(resolve, 200));

    // Ensure local video is attached and playing
    if (localVideo.value && localStream) {
      localVideo.value.srcObject = localStream;
      localVideo.value.muted = true;
      localVideo.value.playsInline = true;
      try {
        await localVideo.value.play();
        console.log("Local video playing successfully");
      } catch (err) {
        console.warn("Local video play failed:", err);
      }
    }

    // Ensure remote video plays if available
    if (remoteVideo.value && remoteVideo.value.srcObject) {
      try {
        await remoteVideo.value.play();
        console.log("Remote video playing successfully");
      } catch (err) {
        console.warn("Remote video play failed:", err);
        showRemotePlayButton.value = true;
      }
    }
  } catch (err) {
    console.error("acceptCall error", err);
    // Show user-friendly error
    alert("Failed to accept call. Please try again.");
    cleanupCall();
  }
};

/* ---------- Reject call ---------- */
const rejectCall = () => {
  if (!incomingCall.value) return;
  socket.emit("reject-call", { callId: incomingCall.value.callId });
  cleanupCall();
};

/* ---------- End/cleanup ---------- */
const cleanupCall = () => {
  try {
    if (localStream) {
      localStream.getTracks().forEach((t) => t.stop());
      localStream = null;
    }
  } catch (e) {}

  try {
    if (peerConnection) {
      peerConnection.close();
      peerConnection = null;
    }
  } catch (e) {}

  // Clear call timeout
  if (callTimeout) {
    clearTimeout(callTimeout);
    callTimeout = null;
  }

  pendingRemoteCandidates = [];
  calling.value = false;
  currentCallId.value = null;
  calleeId.value = "";
  calleeInfo.value = {};
  incomingCall.value = null;
  currentView.value = "call-interface";
  showRemotePlayButton.value = false;
  if (remoteVideo.value) remoteVideo.value.srcObject = null;
  if (localVideo.value) localVideo.value.srcObject = null;
  remoteMediaStream = null;
};

const endCall = () => {
  if (currentCallId.value && socket) {
    socket.emit("end-call", { callId: currentCallId.value });
  }
  cleanupCall();
};

/* ---------- Play remote (user click if autoplay blocked) ---------- */
const playRemote = async () => {
  if (!remoteVideo.value) return;
  try {
    await remoteVideo.value.play();
    showRemotePlayButton.value = false;
  } catch (err) {
    console.warn("playRemote failed", err);
  }
};

/* ---------- Auth + users ---------- */
const authenticate = async () => {
  loading.value = true;
  error.value = "";
  try {
    const res = await fetch(
      `https://5d1f56feeee4.ngrok-free.app/api/auth/${authMode.value}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ username: username.value }),
      }
    );
    const data = await res.json();
    if (!res.ok) {
      error.value = data.error || "Auth failed";
      loading.value = false;
      return;
    }

    currentUser.username = data.username;
    currentUser.userId = data.userId;
    isAuthenticated.value = true;
    initializeSocket();
    loadUsers();
  } catch (err) {
    console.error(err);
    error.value = "Network error";
  } finally {
    loading.value = false;
  }
};

const loadUsers = async () => {
  try {
    const res = await fetch("https://5d1f56feeee4.ngrok-free.app/api/users");
    const data = await res.json();
    users.value = data;
  } catch (err) {
    console.error("loadUsers error", err);
  }
};

/* ---------- Lifecycle ---------- */
onUnmounted(() => {
  if (socket) socket.disconnect();
  cleanupCall();
});

/* ---------- For convenience during dev: auto open socket if already logged in ---------- */
onMounted(() => {
  // nothing to do. user clicks login/register.
});

// Ensure remote stream attaches once the video element exists and view is active
watch([currentView, remoteVideo], async () => {
  if (
    currentView.value === "video-call" &&
    remoteVideo.value &&
    remoteMediaStream
  ) {
    console.log("Watcher: Attaching remote stream to video element");
    if (remoteVideo.value.srcObject !== remoteMediaStream) {
      remoteVideo.value.srcObject = remoteMediaStream;
      remoteVideo.value.playsInline = true;
    }
    try {
      await remoteVideo.value.play();
      showRemotePlayButton.value = false;
      console.log("Watcher: Remote video playing successfully");
    } catch (err) {
      console.warn("Watcher: remoteVideo play after mount blocked:", err);
      showRemotePlayButton.value = true;
    }
  }
});

// Ensure local stream attaches once the video element exists and view is active
watch([currentView, localVideo], async () => {
  if (currentView.value === "video-call" && localVideo.value && localStream) {
    console.log("Watcher: Attaching local stream to video element");
    if (localVideo.value.srcObject !== localStream) {
      localVideo.value.srcObject = localStream;
      localVideo.value.muted = true; // Ensure muted to avoid echo
      localVideo.value.playsInline = true;
    }
    try {
      await localVideo.value.play();
      console.log("Watcher: Local video playing successfully");
    } catch (err) {
      console.warn("Watcher: localVideo play after mount blocked:", err);
    }
  }
});
</script>

<style scoped>
.app {
  min-height: 100vh;
  background: #f5f5f5;
  padding: 20px;
  box-sizing: border-box;
}
.auth-container,
.main-container {
  display: flex;
  justify-content: center;
  align-items: center;
}
.auth-form {
  background: white;
  padding: 20px;
  border-radius: 8px;
  width: 320px;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.06);
}
.tabs {
  display: flex;
  gap: 6px;
  margin-bottom: 12px;
}
.tabs button {
  flex: 1;
  padding: 8px;
  border: none;
  background: #eee;
  cursor: pointer;
}
.tabs button.active {
  background: #007bff;
  color: #fff;
}
.call-interface {
  width: 100%;
  max-width: 900px;
  margin: 0 auto;
}
.contacts-list {
  max-height: 220px;
  overflow: auto;
  margin-top: 10px;
}
.contact-item {
  display: flex;
  justify-content: space-between;
  padding: 8px;
  border-radius: 6px;
  border: 1px solid #eee;
  margin-bottom: 8px;
  cursor: pointer;
  background: #fff;
}
.status.online {
  color: green;
}
.status.offline {
  color: gray;
}
.video-call {
  display: flex;
  flex-direction: column;
  gap: 12px;
  align-items: center;
  width: 100%;
}
.video-container {
  position: relative;
  width: 100%;
  max-width: 900px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 12px;
}
.remote-wrap {
  position: relative;
  width: 100%;
  max-width: 640px;
  height: 480px;
  background: #000;
  border-radius: 8px;
  overflow: hidden;
}
.remote-video {
  width: 100%;
  height: 100%;
  background: #000;
  border-radius: 8px;
  object-fit: cover;
  transform: scaleX(-1); /* Mirror for natural feel */
}
.local-video {
  position: absolute;
  bottom: 12px;
  right: 12px;
  width: 200px;
  height: 150px;
  border: 2px solid #fff;
  border-radius: 8px;
  object-fit: cover;
  background: #000;
  z-index: 10;
  transform: scaleX(-1); /* Mirror for natural feel */
}

/* Mobile responsive adjustments */
@media (max-width: 768px) {
  .remote-wrap {
    width: 100%;
    height: 60vh;
    max-height: 480px;
  }
  .local-video {
    width: 150px;
    height: 112px;
    bottom: 8px;
    right: 8px;
  }
  .video-container {
    padding: 0 10px;
  }
}
.play-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0, 0, 0, 0.25);
  border-radius: 8px;
}
.play-btn {
  padding: 12px 20px;
  border-radius: 8px;
  background: #007bff;
  color: #fff;
  border: none;
  cursor: pointer;
}
.call-actions-bottom {
  margin-top: 20px;
}
.call-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 20px;
}

.outgoing-video-container {
  width: 100%;
  max-width: 400px;
  height: 300px;
  background: #000;
  border-radius: 12px;
  overflow: hidden;
  margin: 20px 0;
}

.outgoing-local-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(-1); /* Mirror for natural feel */
}

.call-actions {
  display: flex;
  gap: 20px;
  margin-top: 20px;
}

.accept-btn,
.reject-btn {
  padding: 15px 30px;
  border: none;
  border-radius: 50px;
  font-size: 18px;
  font-weight: bold;
  cursor: pointer;
  min-width: 120px;
  transition: all 0.3s ease;
}

.accept-btn {
  background: #28a745;
  color: white;
}

.accept-btn:hover {
  background: #218838;
  transform: scale(1.05);
}

.reject-btn {
  background: #dc3545;
  color: white;
}

.reject-btn:hover {
  background: #c82333;
  transform: scale(1.05);
}

.end-call-btn {
  padding: 15px 30px;
  border-radius: 50px;
  background: #dc3545;
  color: #fff;
  border: none;
  cursor: pointer;
  font-size: 18px;
  font-weight: bold;
  min-width: 150px;
  transition: all 0.3s ease;
}

.end-call-btn:hover {
  background: #c82333;
  transform: scale(1.05);
}

/* Mobile button improvements */
@media (max-width: 768px) {
  .call-actions {
    flex-direction: column;
    gap: 15px;
    width: 100%;
    max-width: 300px;
  }

  .accept-btn,
  .reject-btn,
  .end-call-btn {
    width: 100%;
    padding: 20px;
    font-size: 20px;
  }
}
</style>
