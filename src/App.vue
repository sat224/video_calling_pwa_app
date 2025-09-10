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
        <button @click="endCall" class="end-call-btn">End Call</button>
      </div>

      <!-- Incoming Call -->
      <div v-else-if="currentView === 'incoming-call'" class="call-screen">
        <div class="call-info">
          <h3>Incoming Call</h3>
          <p>{{ incomingCall.callerName }}</p>
        </div>
        <div class="call-actions">
          <button @click="acceptCall" class="accept-btn">Accept</button>
          <button @click="rejectCall" class="reject-btn">Reject</button>
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
              class="remote-video"
            ></video>

            <!-- Local preview (small) -->
            <video
              ref="localVideo"
              autoplay
              muted
              playsinline
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
          <button @click="endCall" class="end-call-btn">End Call</button>
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
    const stream = ev.streams && ev.streams[0] ? ev.streams[0] : null;
    if (stream) {
      // persist and attach when possible
      remoteMediaStream = stream;
      if (remoteVideo.value && remoteVideo.value.srcObject !== stream) {
        remoteVideo.value.srcObject = stream;
        remoteVideo.value.play().catch((err) => {
          console.warn("remote.play() blocked:", err);
          showRemotePlayButton.value = true;
        });
      }
    } else {
      // fallback: create MediaStream from track
      const fallbackStream = new MediaStream([ev.track]);
      remoteMediaStream = fallbackStream;
      if (remoteVideo.value && remoteVideo.value.srcObject !== fallbackStream) {
        remoteVideo.value.srcObject = fallbackStream;
        remoteVideo.value
          .play()
          .catch(() => (showRemotePlayButton.value = true));
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
      cleanupCall();
    }
  };
};

/* ---------- getUserMedia + attach to local video ---------- */
const startLocalStream = async () => {
  if (localStream) return localStream;
  try {
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true,
    });
    console.log("Got local stream", localStream);
    console.log("Local stream tracks:", localStream.getTracks());

    // Attach to local video element if it exists
    if (localVideo.value) {
      console.log("Attaching local stream to video element");
      localVideo.value.srcObject = localStream;
      // local is muted to avoid echo
      localVideo.value.muted = true;
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

    // On caller side: set remote description (answer) and drain queued ICE
    try {
      createPeerConnectionIfNeeded();
      await peerConnection.setRemoteDescription(
        new RTCSessionDescription(data.answer)
      );
      await drainPendingCandidates();
      currentView.value = "video-call";
    } catch (err) {
      console.error("Error applying answer on caller:", err);
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
  calling.value = true;
  currentView.value = "outgoing-call";

  createPeerConnectionIfNeeded();

  // get local media and add tracks
  const stream = await startLocalStream();
  stream.getTracks().forEach((t) => peerConnection.addTrack(t, stream));

  // create offer
  const offer = await peerConnection.createOffer();
  await peerConnection.setLocalDescription(offer);

  // send offer to server (server will forward to callee)
  socket.emit("initiate-call", {
    callerId: currentUser.userId,
    calleeId: calleeId.value,
    offer: peerConnection.localDescription,
  });
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

    currentView.value = "video-call";
  } catch (err) {
    console.error("acceptCall error", err);
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
    if (remoteVideo.value.srcObject !== remoteMediaStream) {
      remoteVideo.value.srcObject = remoteMediaStream;
    }
    try {
      await remoteVideo.value.play();
      showRemotePlayButton.value = false;
    } catch (err) {
      console.warn("remoteVideo play after mount blocked:", err);
      showRemotePlayButton.value = true;
    }
  }
});

// Ensure local stream attaches once the video element exists and view is active
watch([currentView, localVideo], async () => {
  if (currentView.value === "video-call" && localVideo.value && localStream) {
    if (localVideo.value.srcObject !== localStream) {
      localVideo.value.srcObject = localStream;
      localVideo.value.muted = true; // Ensure muted to avoid echo
    }
    try {
      await localVideo.value.play();
    } catch (err) {
      console.warn("localVideo play after mount blocked:", err);
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
  width: 640px;
  height: 480px;
}
.remote-video {
  width: 100%;
  height: 100%;
  background: #000;
  border-radius: 8px;
  object-fit: cover;
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
  margin-top: 8px;
}
.end-call-btn {
  padding: 10px 18px;
  border-radius: 8px;
  background: #dc3545;
  color: #fff;
  border: none;
  cursor: pointer;
}
.call-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}
</style>
