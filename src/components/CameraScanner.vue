<template>
  <div id="app">
    <header>
      <img
        src="https://cdn.prod.website-files.com/65363d385688281ce5b016ec/65363d385688281ce5b016f5_Vectors-Wrapper.svg"
        alt="Celfocus Logo"
      />
    </header>
    <h1>Camera Test</h1>
    <!-- Toggle for camera mode. Disabled if scanner is running -->
    <div class="toggle-mode">
      <label>
        <input
          type="radio"
          v-model="cameraMode"
          value="inline"
          :disabled="isScannerRunning"
        />
        Inline
      </label>
      <label>
        <input
          type="radio"
          v-model="cameraMode"
          value="modal"
          :disabled="isScannerRunning"
        />
        Modal
      </label>
    </div>

    <!-- Inline Mode: Render the camera container on the page -->
    <div
      v-if="cameraMode === 'inline'"
      id="camera-container"
      @click="toggleScanner"
      :key="scannerKey"
    >
      <div id="scanner" ref="scannerContainer"></div>
    </div>

    <!-- Modal Mode: Render a modal overlay when scanner is active -->
    <div
      v-if="cameraMode === 'modal' && showModal"
      class="modal-overlay"
      @click.self="stopScanner"
    >
      <div class="modal-content" :key="scannerKey">
        <div id="scanner" ref="scannerContainer"></div>
        <button class="close-btn" @click="stopScanner">Close</button>
      </div>
    </div>

    <div id="output">
      Scanned Result: <span>{{ scannedResult }}</span>
    </div>
    <!-- Display error message if any -->
    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>
    <!-- Buttons to manually start/stop the scanner -->
    <button @click="startScanner" :disabled="isScannerRunning">
      Start Scanner
    </button>
    <button @click="stopScanner" :disabled="!isScannerRunning">
      Stop Scanner
    </button>

    <!-- Log output -->
    <div id="log">
      <h2>Application Log</h2>
      <ul>
        <li v-for="(log, index) in logs" :key="index">{{ log }}</li>
      </ul>
    </div>

    <!-- New Test Camera Section (without QuaggaJS) -->
    <div id="test-camera-section">
      <h2>Test Camera Section</h2>
      <video
        ref="testCameraVideo"
        autoplay
        muted
        playsinline
        style="width: 100%; max-width: 600px; height: auto; border: 1px solid #ccc;"
      ></video>
      <br />
      <button @click="startTestCamera" :disabled="testCameraRunning">
        Start Test Camera
      </button>
      <button @click="stopTestCamera" :disabled="!testCameraRunning">
        Stop Test Camera
      </button>
    </div>
  </div>
</template>

<script>
import Quagga from 'quagga';

export default {
  name: 'CameraScanner',
  data() {
    return {
      isScannerRunning: false,
      scannedResult: 'None',
      cameraStream: null,
      errorMessage: '',
      errorTimer: null,
      // Camera mode can be 'inline' or 'modal'
      cameraMode: 'inline',
      // For modal mode: controls whether the modal is shown
      showModal: false,
      // Dynamic key to force re-render of the scanner container
      scannerKey: 0,
      logs: [], // Array to hold log messages

      // Properties for the test camera (without QuaggaJS)
      testCameraRunning: false,
      testCameraStream: null
    };
  },
  watch: {
    cameraMode() {
      this.addLog(`Camera mode changed to: ${this.cameraMode}`);
      this.scannerKey++;
    }
  },
  methods: {
    addLog(message) {
      const timestamp = new Date().toLocaleTimeString();
      this.logs.push(`${timestamp}: ${message}`);
    },
    // Helper to show an error message for 5 seconds, with logging.
    showError(message) {
      if (this.errorTimer) {
        clearTimeout(this.errorTimer);
      }
      this.errorMessage = message;
      this.addLog(`Error: ${message}`);
      this.errorTimer = setTimeout(() => {
        this.errorMessage = '';
        this.errorTimer = null;
      }, 5000);
    },
    // QuaggaJS scanner initialization
    initScanner() {
      if (this.errorTimer) {
        clearTimeout(this.errorTimer);
        this.errorTimer = null;
      }
      this.errorMessage = '';
      this.addLog("Initializing scanner...");

      Quagga.init(
        {
          inputStream: {
            type: 'LiveStream',
            target: this.$refs.scannerContainer,
            constraints: {
              facingMode: 'environment'
            }
          },
          decoder: {
            readers: [
              'code_128_reader',
              'ean_reader',
              'ean_8_reader',
              'code_39_reader'
            ]
          }
        },
        (err) => {
          if (err) {
            this.addLog(`QuaggaJS Initialization Error: ${err.name} - ${err.message}`);
            console.error('QuaggaJS Initialization Error:', err);
            if (
              err.name === 'NotAllowedError' ||
              err.name === 'PermissionDeniedError'
            ) {
              this.showError('Camera access was denied. Please allow camera permissions.');
            } else if (
              err.name === 'NotReadableError' ||
              err.name === 'TrackStartError'
            ) {
              this.showError('The camera is currently in use by another application. Please close it and try again.');
            } else {
              this.showError('Unable to access the camera. Please try again later.');
            }
            if (this.cameraMode === 'modal') {
              this.showModal = false;
            }
            return;
          }
          // On successful initialization, start the scanner.
          this.addLog("Initializing scanner Complete... Calling startScannerAfterInit");
          this.startScannerAfterInit();
        }
      );

      Quagga.onDetected((data) => {
        this.scannedResult = data.codeResult.code;
        this.addLog(`Detected code: ${data.codeResult.code}`);
      });
    },
    startScannerAfterInit() {
      this.addLog("Calling Quagga.start()...");
      Quagga.start();
      this.isScannerRunning = true;
      this.addLog("Scanner started successfully.");

      this.cameraStream = (this.$refs.scannerContainer.querySelector('video')) 
        ? this.$refs.scannerContainer.querySelector('video').srcObject 
        : null;
    },
    startScanner() {
      if (this.isScannerRunning) return;
      this.addLog("Attempting to start scanner...");
      // Force a fresh render of the scanner container
      this.scannerKey++;
      this.$nextTick(() => {
        if (this.cameraMode === 'modal') {
          this.showModal = true;
          this.addLog("Showing modal...");
          this.$nextTick(() => {
            this.initScanner();
          });
        } else {
          this.initScanner();
        }
      });
    },
    stopScanner() {
      if (!this.isScannerRunning) {
        if (this.cameraMode === 'modal' && this.showModal) {
          this.showModal = false;
        }
        return;
      }
      this.addLog("Stopping scanner...");
      Quagga.stop();

      if (this.$refs.scannerContainer) {
        const videoEl = this.$refs.scannerContainer.querySelector('video');
        if (videoEl) {
          videoEl.pause();
          videoEl.srcObject = null;
          this.addLog("Video element paused and srcObject cleared.");
        }
      }
      this.scannerKey++;
      if (this.cameraMode === 'inline' && this.$refs.scannerContainer) {
        this.$refs.scannerContainer.innerHTML = '';
        this.addLog("Scanner container innerHTML cleared.");
      }
      if (this.cameraStream) {
        this.cameraStream.getTracks().forEach((track, index) => {
          track.stop();
          this.addLog(`Track ${index} (${track.kind}) stopped.`);
        });
        this.cameraStream = null;
      }
      Quagga.offDetected();
      this.isScannerRunning = false;
      this.addLog("Scanner stopped and all references cleared.");
      if (this.errorTimer) {
        clearTimeout(this.errorTimer);
        this.errorTimer = null;
      }
      this.errorMessage = '';
      
      if (this.cameraMode === 'modal') {
        this.showModal = false;
      }
    },
    toggleScanner() {
      if (this.cameraMode === 'inline') {
        this.isScannerRunning ? this.stopScanner() : this.startScanner();
      }
    },
    // New methods for testing the camera without QuaggaJS.
    startTestCamera() {
      if (this.testCameraRunning) return;
      navigator.mediaDevices.getUserMedia({ video: true })
        .then((stream) => {
          this.$refs.testCameraVideo.srcObject = stream;
          this.testCameraStream = stream;
          this.testCameraRunning = true;
          this.addLog('Test camera started.');
        })
        .catch((err) => {
          this.addLog(`Error starting test camera: ${err}`);
          this.showError(`Error starting test camera: ${err.message}`);
        });
    },
    stopTestCamera() {
      if (!this.testCameraRunning) return;
      if (this.testCameraStream) {
        this.testCameraStream.getTracks().forEach((track, index) => {
          track.stop();
          this.addLog(`Test camera track ${index} (${track.kind}) stopped.`);
        });
      }
      // Clear the video element source
      if (this.$refs.testCameraVideo) {
        this.$refs.testCameraVideo.srcObject = null;
      }
      this.testCameraRunning = false;
      this.testCameraStream = null;
      this.addLog('Test camera stopped.');
    }
  },
  beforeUnmount() {
    if (this.isScannerRunning) {
      this.stopScanner();
    }
    if (this.testCameraRunning) {
      this.stopTestCamera();
    }
    if (this.errorTimer) {
      clearTimeout(this.errorTimer);
      this.errorTimer = null;
    }
  }
};
</script>

<style scoped>
/* General Reset */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Arial', sans-serif;
  text-align: center;
  background-color: #ffffff;
  color: #333333;
}

header {
  margin: 20px 0;
}

header img {
  width: 150px;
  height: auto;
}

h1 {
  font-size: 36px;
  margin: 20px 0;
  color: #1e1e1e;
  font-weight: bold;
}

/* Toggle controls */
.toggle-mode {
  margin-bottom: 20px;
}
.toggle-mode label {
  margin-right: 15px;
  font-weight: bold;
}

/* Inline mode container */
#camera-container {
  position: relative;
  width: 100%;
  max-width: 600px;
  height: 350px;
  margin: 20px auto;
  border: 1px solid #e0e0e0;
  background-color: #f4f4f4;
  overflow: hidden;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  cursor: pointer;
}

/* Modal styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal-content {
  position: relative;
  width: 600px;
  max-width: 90%;
  height: 350px;
  background: #f4f4f4;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}
.close-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  background: #1e1e1e;
  color: white;
  border: none;
  border-radius: 50px;
  padding: 5px 10px;
  cursor: pointer;
}

/* Scanner video styling */
#scanner video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  background-color: #000;
}

#output {
  margin-top: 20px;
  font-size: 1.2em;
  color: #1e1e1e;
  font-weight: bold;
}

.error-message {
  color: red;
  margin: 10px 0;
  font-weight: bold;
}

button {
  padding: 10px 20px;
  margin: 15px 10px;
  font-size: 1em;
  font-weight: bold;
  color: #ffffff;
  background-color: #1e1e1e;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: all 0.3s ease;
}

button:hover {
  background-color: #ff5a5f;
}

button:disabled {
  background-color: #b3b3b3;
  cursor: not-allowed;
}

/* Log styles */
#log {
  margin: 20px auto;
  max-width: 600px;
  text-align: left;
  background: #f9f9f9;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
}
#log h2 {
  font-size: 18px;
  margin-bottom: 10px;
}
#log ul {
  list-style: none;
  padding: 0;
  margin: 0;
}
#log li {
  font-size: 14px;
  margin-bottom: 5px;
  border-bottom: 1px solid #eee;
  padding-bottom: 3px;
}

/* Test camera section styling */
#test-camera-section {
  margin: 20px auto;
  max-width: 600px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #f9f9f9;
}
</style>
