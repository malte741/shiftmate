<template>
  <div id="app">
    <header>
      <h1>ShiftMate BLE</h1>
    </header>
    <main>
      <section class="connection-controls">
        <button @click="connectToDevice" :disabled="bleState === 'Connected'" class="btn btn-primary">Connect to BLE Device</button>
        <button @click="disconnectDevice" :disabled="bleState !== 'Connected'" class="btn btn-secondary">Disconnect BLE Device</button>
        <p class="ble-state">BLE state: <strong :class="bleStateClass">{{ bleState }}</strong></p>
      </section>y

      <section class="sensor-readings">
        <h2>Sensor Readings</h2>
        <div class="reading-grid">
          <div class="reading-item">
            <h3>Speed</h3>
            <span class="reading-value">{{ speed }}</span>
          </div>
          <div class="reading-item">
            <h3>Current Gear</h3>
            <span class="reading-value">{{ currentGear }}</span>
          </div>
          <div class="reading-item">
            <h3>ADXL</h3>
            <span class="reading-value">{{ adxl }}</span>
          </div>
        </div>
        <p class="timestamp">Last reading: {{ timestamp }}</p>
      </section>

      <section class="control-setup">
        <h2>Control Setup</h2>
        <!-- Eingabefelder für Float-Werte -->
        <div>
          <label for="floatValue1">Float Value 1 (0-100):</label>
          <input type="number" id="floatValue1" v-model.number="floatValue1" min="0" max="100" step="0.1">
        </div>

        <div>
          <label for="floatValue2">Float Value 2 (0-100):</label>
          <input type="number" id="floatValue2" v-model.number="floatValue2" min="0" max="100" step="0.1">
        </div>

        <!-- Button zum Senden der Float-Werte -->
        <button @click="sendFloatValues(floatValue1, floatValue2)">Send Float Values</button>

        <!-- Button zum Umschalten des Setup-Modus -->
        <button @click="toggleSetupMode">{{ setupMode ? 'Disable' : 'Enable' }} Setup Mode</button>
        <p>Last value sent: <span class="sent-value">{{ lastValueSent }}</span></p>
      </section>
    </main>
  </div>
</template>

<script>
export default  {
  name: 'App',
  data() {
    return {
      // BLE device specifications
      deviceName: 'ESP32',
      bleService: '19b10000-e8f2-537e-4f6c-d104768a1214',
      dataChar: '19b10001-e8f2-537e-4f6c-d104768a1214',
      setupChar: '06f4fdef-0beb-4d95-8669-d89830dcd0ce',

      // BLE connection variables
      bleServer: null,
      dataCharacteristic: null,
      setupCharacteristic: null,

      // UI state variables
      bleState: 'Disconnected',
      bleStateColor: 'red',
      speed: 'N/A',
      currentGear: 'N/A',
      adxl: 'N/A',
      timestamp: 'N/A',
      lastValueSent: 'N/A',
      floatValue1: 0,
      floatValue2: 0,
      setupMode: false
    }
  },
  computed: {
    bleStateClass() {
      switch(this.bleState) {
        case 'Connected':
          return 'state-connected';
        case 'Disconnected':
          return 'state-disconnected';
        default:
          return '';
      }
    }
  },
  methods: {
    connectToDevice() {
      navigator.bluetooth.requestDevice({
        filters: [{ name: this.deviceName }],
        optionalServices: [this.bleService]
      })
          .then(device => {
            console.log('Device selected:', device.name);
            this.updateBleState('Connecting...', 'blue');
            return device.gatt.connect();
          })
          .then(server => {
            console.log('Connected to GATT server');
            this.bleServer = server;
            return server.getPrimaryService(this.bleService);
          })
          .then(service => {
            console.log('Service found:', service.uuid);
            return Promise.all([
              service.getCharacteristic(this.dataChar),
              service.getCharacteristic(this.setupChar)
            ]);
          })
          .then(([data, setup]) => {
            this.dataCharacteristic = data;
            this.setupCharacteristic = setup;

            this.dataCharacteristic.addEventListener('characteristicvaluechanged', this.handleDataChange);

            return this.dataCharacteristic.startNotifications();
          })
          .then(() => {
            console.log('Notifications started');
            this.updateBleState('Connected', 'green');
          })
          .catch(error => {
            console.error('Error:', error);
            this.updateBleState('Error: ' + error, 'red');
          });
    },

    disconnectDevice() {
      if (this.bleServer && this.bleServer.connected) {
        this.bleServer.disconnect();
        console.log('Device disconnected');
        this.updateBleState('Disconnected', 'red');
      }
    },

    handleDataChange(event) {
      const value = new Uint8Array(event.target.value.buffer);
      this.speed = value[0];
      this.currentGear = value[1];
      this.adxl = new Int8Array(value.buffer)[2]; // Convert to signed int for ADXL

      console.log(`Received - Speed: ${this.speed}, Gear: ${this.currentGear}, ADXL: ${this.adxl}`);
      this.updateTimestamp();
    },

    toggleSetupMode() {
      this.setupMode = !this.setupMode; // Toggle the setup mode state

      const valueToSend = new Uint8Array([0, this.setupMode ? 1 : 0]); // Erster Byte ist der Typ, zweiter Byte ist der Wert
      if (this.setupCharacteristic && this.bleServer.connected) {
        this.setupCharacteristic.writeValue(valueToSend)
            .then(() => {
              console.log('Setup mode value sent:', this.setupMode);
            })
            .catch(error => console.error('Error writing value:', error));
      } else {
        console.error('BLE not connected or characteristic not found');
        alert('Please connect to the device first');
      }
    },

    sendFloatValues(value1, value2) {
      const valueToSend = new Uint8Array(9); // 1 Byte für den Typ + 8 Bytes für zwei Floats
      valueToSend[0] = 1; // Typ: Float-Werte

      // Float-Werte in das Array kopieren
      new DataView(valueToSend.buffer).setFloat32(1, value1, true); // true für little-endian
      new DataView(valueToSend.buffer).setFloat32(5, value2, true); // true für little-endian

      if (this.setupCharacteristic && this.bleServer.connected) {
        this.setupCharacteristic.writeValue(valueToSend)
            .then(() => {
              console.log('Float values sent:', value1, value2);
            })
            .catch(error => console.error('Error writing float values:', error));
      } else {
        console.error('BLE not connected or characteristic not found');
        alert('Please connect to the device first');
      }
    },

    updateBleState(message, color) {
      this.bleState = message;
      this.bleStateColor = color;
    },

    updateTimestamp() {
      this.timestamp = new Date().toLocaleString();
    }
  }
}
</script>

<style scoped>
#app {
  font-family: 'Arial', sans-serif;
  color: #2c3e50;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

header {
  background-color: #34495e;
  color: white;
  padding: 20px;
  border-radius: 8px 8px 0 0;
}

h1 {
  margin: 0;
}

main {
  background-color: #ecf0f1;
  padding: 20px;
  border-radius: 0 0 8px 8px;
}

section {
  background-color: white;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

h2 {
  color: #34495e;
  border-bottom: 2px solid #3498db;
  padding-bottom: 10px;
}

.btn {
  padding: 10px 15px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s;
}

.btn-primary {
  background-color: #3498db;
  color: white;
}

.btn-primary:hover {
  background-color: #2980b9;
}

.btn-secondary {
  background-color: #95a5a6;
  color: white;
}

.btn-secondary:hover {
  background-color: #7f8c8d;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.ble-state {
  margin-top: 10px;
}

.ble-state strong {
  padding: 3px 8px;
  border-radius: 3px;
}

.state-connected {
  background-color: #2ecc71;
  color: white;
}

.state-disconnected {
  background-color: #e74c3c;
  color: white;
}

.reading-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
}

.reading-item {
  text-align: center;
  padding: 15px;
  background-color: #f7f9fa;
  border-radius: 8px;
}

.reading-value {
  font-size: 24px;
  font-weight: bold;
  color: #3498db;
}

.timestamp {
  text-align: right;
  font-style: italic;
  color: #7f8c8d;
}

.setup-input {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

input[type="number"] {
  flex-grow: 1;
  padding: 10px;
  border: 1px solid #bdc3c7;
  border-radius: 5px;
}

.sent-value {
  font-weight: bold;
  color: #3498db;
}
</style>