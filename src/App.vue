<template>
  <div id="app" :class="{ 'dark-mode': isDarkMode }">

    <!-- Main Container -->
    <div class="main-container">
      <!-- Left Sidebar -->
      <div class="sidebar">
        <!-- File Upload Card -->
        <div class="card upload-card">
          <div class="card-header">
            <h3>📁 Upload Excel File</h3>
          </div>
          <div class="card-body">
            <div class="file-upload-area" :class="{ 'drag-over': isDragOver }" 
                 @dragover.prevent="isDragOver = true" 
                 @dragleave="isDragOver = false"
                 @drop.prevent="handleFileDrop"
                 @click="$refs.fileInput.click()">
              <div class="upload-icon">📊</div>
              <p class="upload-text">Drag & drop your Excel file here</p>
              <p class="upload-subtext">or click to browse</p>
              <input type="file" @change="handleFileUpload" accept=".xlsx,.xls" class="file-input" ref="fileInput" />
            </div>
            <button @click="processFile" :disabled="!selectedFile" class="btn btn-primary btn-full">
              <span class="btn-icon">⚡</span>
              Process File
            </button>
            <div v-if="selectedFile" class="file-info">
              <span class="file-name">{{ selectedFile.name }}</span>
            </div>
          </div>
        </div>

        <!-- Google Sheet Import Card (New) -->
        <div class="card sheet-import-card">
          <div class="card-header">
            <h3>📄 Import from Google Sheet (CSV URL)</h3>
          </div>
          <div class="card-body">
            <input type="text" v-model="googleSheetUrl" placeholder="Enter Google Sheet CSV URL" class="sheet-input" />
            <button @click="importFromGoogleSheet" :disabled="!googleSheetUrl" class="btn btn-secondary btn-full">
              <span class="btn-icon">🔗</span>
              Import Data
            </button>
            <p v-if="sheetImportStatus" class="import-status">{{ sheetImportStatus }}</p>
          </div>
        </div>

        <!-- Entries Display -->
        <div v-if="entries.length > 0" class="card entries-card">
          <div class="card-header">
            <h3>👥 Participants</h3>
            <span class="badge">{{ totalEntries }} total entries</span>
          </div>
          <div class="card-body">
            <div class="entries-list">
              <div v-for="entry in entries" :key="entry.name" class="entry-chip">
                <span class="entry-name">{{ entry.name }}</span>
                <span class="entry-count">{{ entry.count }}</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Controls -->
        <div v-if="entries.length > 0" class="card controls-card">
          <div class="card-body">
            <div class="controls">
              <button @click="startSpin" :disabled="isSpinning || wheelEntries.length === 0" class="btn btn-success btn-full">
                <span class="btn-icon">🚀</span>
                {{ isSpinning ? 'Spinning...' : 'Start Spin' }}
              </button>
              <button @click="stopSpin" :disabled="!isSpinning" class="btn btn-danger btn-full">
                <span class="btn-icon">⏹️</span>
                Stop
              </button>
            </div>
            
            <div v-if="wheelEntries.length === 0" class="no-entries">
              <div class="empty-state">
                <div class="empty-icon">🎊</div>
                <h3>All Done!</h3>
                <p>No more entries available</p>
                <button @click="resetEntries" class="btn btn-primary btn-full">
                  <span class="btn-icon">🔄</span>
                  Reset All Entries
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Main Content Area -->
      <div class="main-content">
        <!-- Wheel Section -->
        <div v-if="entries.length > 0" class="wheel-section">
          <div class="wheel-container">
            <div class="wheel-wrapper">
              <!-- Wheel with proper segments -->
              <svg class="wheel-svg" :style="{ transform: `rotate(${rotation}deg)` }" :class="{ spinning: isSpinning }">
                <g v-for="(entry, index) in wheelEntries" :key="index">
                  <!-- Segment path -->
                  <path 
                    :d="getSegmentPath(index)"
                    :fill="getColor(index)"
                    :stroke="'#fff'"
                    :stroke-width="2"
                    class="wheel-segment"
                  />
                  <!-- Text label -->
                  <text 
                    :x="getTextX(index)"
                    :y="getTextY(index)"
                    :transform="`rotate(${getTextRotation(index)}, ${getTextX(index)}, ${getTextY(index)})`"
                    class="segment-text"
                    :font-size="getTextSize()"
                  >
                    {{ entry }}
                  </text>
                </g>
              </svg>
              
              <!-- Center circle -->
              <div class="wheel-center">
                <div class="center-circle"></div>
              </div>
              
              <!-- Pointer -->
              <div class="pointer"></div>
            </div>
          </div>
          
          <div v-if="winner" class="winner-announcement">
            <div class="winner-card">
              <div class="winner-icon">🏆</div>
              <h2 class="winner-name">{{ winner }}</h2>
              <p class="winner-text">Congratulations!</p>
              <p class="winner-remaining">{{ getEntryCount(winner) }} entries remaining</p>
            </div>
          </div>
        </div>

        <!-- Welcome Message - REMOVED -->
        <!-- 
        <div v-else class="welcome-card">
          <div class="welcome-content">
            <div class="welcome-icon">🎯</div>
            <h2>Welcome to Lucky Wheel Spinner!</h2>
            <p>Upload an Excel file with "Name" and "Entry" columns to get started.</p>
            <div class="example">
              <h4>Example Excel format:</h4>
              <table class="example-table">
                <thead>
                  <tr>
                    <th>Name</th>
                    <th>Entry</th>
                  </tr>
                </thead>
                <tbody>
                  <tr>
                    <td>Max</td>
                    <td>1</td>
                  </tr>
                  <tr>
                    <td>Nick</td>
                    <td>5</td>
                  </tr>
                  <tr>
                    <td>Sarah</td>
                    <td>3</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
        -->
      </div>
    </div>

    <!-- Theme Toggle Button (New) -->
    <button @click="toggleTheme" class="theme-toggle-btn">
      {{ isDarkMode ? '☀️ Light Mode' : '🌙 Dark Mode' }}
    </button>
  </div>
</template>

<script>
import * as XLSX from 'xlsx'
// For Google Sheet import, you might need a library like PapaParse for robust CSV parsing
// import Papa from 'papaparse';

export default {
  name: 'WheelSpinner',
  data() {
    return {
      selectedFile: null,
      entries: [],
      originalEntries: [], // To store the initial set of entries for reset
      wheelEntries: [], // Entries currently on the wheel
      rotation: 0,
      isSpinning: false,
      winner: null,
      spinInterval: null,
      isDragOver: false,
      wheelRadius: 280, // Radius of the wheel in pixels
      googleSheetUrl: '', // New data property for Google Sheet URL
      sheetImportStatus: '', // New data property for import status
      isDarkMode: false, // New data property for theme
    }
  },
  computed: {
    totalEntries() {
      return this.entries.reduce((sum, entry) => sum + entry.count, 0)
    },
    segmentAngle() {
      return this.wheelEntries.length > 0 ? 360 / this.wheelEntries.length : 0
    }
  },
  methods: {
    handleFileUpload(event) {
      this.selectedFile = event.target.files[0]
      this.googleSheetUrl = ''; // Clear sheet URL if a file is selected
      this.sheetImportStatus = '';
    },
    handleFileDrop(event) {
      this.isDragOver = false;
      this.selectedFile = event.dataTransfer.files[0];
      this.googleSheetUrl = ''; // Clear sheet URL if a file is dropped
      this.sheetImportStatus = '';
    },
    processFile() {
      if (!this.selectedFile) return

      const reader = new FileReader()
      reader.onload = (e) => {
        try {
          const data = new Uint8Array(e.target.result)
          const workbook = XLSX.read(data, { type: 'array' })
          const firstSheetName = workbook.SheetNames[0]
          const worksheet = workbook.Sheets[firstSheetName]
          const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 })

          if (jsonData.length < 2) {
            alert('Excel file must have a header row and at least one data row.')
            return
          }

          const header = jsonData[0].map(h => String(h).trim());
          const nameIndex = header.indexOf('Name')
          const entryIndex = header.indexOf('Entry')

          if (nameIndex === -1 || entryIndex === -1) {
            alert('Excel file must contain "Name" and "Entry" columns.')
            return
          }

          const parsedEntries = []
          for (let i = 1; i < jsonData.length; i++) {
            const row = jsonData[i]
            const name = row[nameIndex] ? String(row[nameIndex]).trim() : null;
            const count = row[entryIndex] ? parseInt(row[entryIndex], 10) : 0;

            if (name && !isNaN(count) && count > 0) {
              parsedEntries.push({ name, count })
            }
          }
          
          this.entries = this.aggregateEntries(parsedEntries);
          this.originalEntries = JSON.parse(JSON.stringify(this.entries)); // Deep copy
          this.wheelEntries = this.generateWheelEntries();
          this.winner = null;
          this.rotation = 0;
          this.sheetImportStatus = ''; // Clear sheet status
        } catch (error) {
          console.error('Error processing Excel file:', error)
          alert('Error processing Excel file. Please ensure it is a valid .xlsx or .xls file and formatted correctly.')
        }
      }
      reader.readAsArrayBuffer(this.selectedFile)
    },

    async importFromGoogleSheet() {
      if (!this.googleSheetUrl) return;
      this.sheetImportStatus = 'Importing...';
      try {
        const response = await fetch(this.googleSheetUrl);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const csvText = await response.text();
        
        const lines = csvText.split(/\r\n|\n/); // Split by newline characters
        const newEntries = [];
        if (lines.length < 2) {
            throw new Error('CSV data must have a header row and at least one data row.');
        }

        const header = lines[0].split(',').map(h => h.trim());
        const nameIndex = header.indexOf('Name');
        const entryIndex = header.indexOf('Entry');

        if (nameIndex === -1 || entryIndex === -1) {
          throw new Error('CSV must contain "Name" and "Entry" columns.');
        }

        for (let i = 1; i < lines.length; i++) {
          if (lines[i].trim() === '') continue; // Skip empty lines
          const values = lines[i].split(',');
          if (values.length >= Math.max(nameIndex, entryIndex) + 1) {
            const name = values[nameIndex] ? values[nameIndex].trim() : null;
            const count = values[entryIndex] ? parseInt(values[entryIndex].trim(), 10) : 0;
            if (name && !isNaN(count) && count > 0) {
              newEntries.push({ name, count });
            }
          }
        }

        this.entries = this.aggregateEntries(newEntries);
        this.originalEntries = JSON.parse(JSON.stringify(this.entries));
        this.wheelEntries = this.generateWheelEntries();
        this.winner = null;
        this.rotation = 0;
        this.selectedFile = null; // Clear Excel file selection
        this.sheetImportStatus = `Successfully imported ${newEntries.length} rows.`;
      } catch (error) {
        console.error('Error importing from Google Sheet:', error);
        this.sheetImportStatus = `Error: ${error.message}`;
        alert(`Error importing from Google Sheet: ${error.message}`);
      }
    },

    aggregateEntries(parsedEntries) {
      const entryMap = new Map();
      parsedEntries.forEach(item => {
        if (entryMap.has(item.name)) {
          entryMap.set(item.name, entryMap.get(item.name) + item.count);
        } else {
          entryMap.set(item.name, item.count);
        }
      });
      return Array.from(entryMap, ([name, count]) => ({ name, count })).sort((a, b) => a.name.localeCompare(b.name));
    },

    generateWheelEntries() {
      const tempWheelEntries = []
      this.entries.forEach(entry => {
        for (let i = 0; i < entry.count; i++) {
          tempWheelEntries.push(entry.name)
        }
      })
      // Shuffle the entries for randomness on the wheel
      for (let i = tempWheelEntries.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [tempWheelEntries[i], tempWheelEntries[j]] = [tempWheelEntries[j], tempWheelEntries[i]];
      }
      return tempWheelEntries;
    },

    startSpin() {
      if (this.isSpinning || this.wheelEntries.length === 0) return
      this.isSpinning = true
      this.winner = null

      // Set up continuous spinning
      const spinSpeed = 10 // Adjust this value to control speed (degrees per frame)
      
      const animateSpin = () => {
        this.rotation += spinSpeed
        this.spinInterval = requestAnimationFrame(animateSpin)
      }
      
      this.spinInterval = requestAnimationFrame(animateSpin)
    },

    stopSpin() {
      if (!this.isSpinning) return
      
      // Cancel the continuous spinning animation
      cancelAnimationFrame(this.spinInterval)
      
      // Set up a slowing down animation
      const initialSpeed = 10 // Should match the speed in startSpin
      let currentSpeed = initialSpeed
      const deceleration = 0.02 // Reduced from 0.05 to make the animation last longer
      
      const slowDownSpin = () => {
        if (currentSpeed > 0.1) {
          this.rotation += currentSpeed
          currentSpeed -= deceleration
          this.spinInterval = requestAnimationFrame(slowDownSpin)
        } else {
          // When almost stopped, determine the winner
          this.determineWinner()
          this.isSpinning = false
        }
      }
      
      this.spinInterval = requestAnimationFrame(slowDownSpin)
    },

    determineWinner() {
      const degreesPerSegment = 360 / this.wheelEntries.length
      const normalizedRotation = (this.rotation % 360 + 360) % 360 // Normalize rotation to 0-360
      
      // The pointer is at the top (0 degrees or 360 degrees).
      // We need to find which segment aligns with the pointer.
      // A segment's range is [startAngle, endAngle]. The pointer is at an effective 0 degrees.
      // The wheel rotates clockwise, so a positive rotation means the 0-degree mark of the wheel has moved clockwise.
      // The winning segment is the one whose range covers the pointer's position *after* accounting for wheel rotation.
      // If the wheel rotates `R` degrees, the pointer is effectively at `360 - R` on the *unrotated* wheel.
      const pointerEffectiveAngle = (360 - normalizedRotation) % 360;

      let winningIndex = -1;
      for (let i = 0; i < this.wheelEntries.length; i++) {
        const segmentStartAngle = i * degreesPerSegment;
        const segmentEndAngle = (i + 1) * degreesPerSegment;
        // Check if pointerEffectiveAngle falls within this segment's range
        if (pointerEffectiveAngle >= segmentStartAngle && pointerEffectiveAngle < segmentEndAngle) {
          winningIndex = i;
          break;
        }
      }
      
      // Fallback if something went wrong (e.g., floating point issues at boundaries)
      if (winningIndex === -1) {
         winningIndex = Math.floor(pointerEffectiveAngle / degreesPerSegment) % this.wheelEntries.length;
      }

      this.winner = this.wheelEntries[winningIndex]
      this.updateEntriesAfterWin(this.winner)
    },

    updateEntriesAfterWin(winnerName) {
      const winnerEntry = this.entries.find(e => e.name === winnerName)
      if (winnerEntry) {
        winnerEntry.count--
        if (winnerEntry.count === 0) {
          this.entries = this.entries.filter(e => e.name !== winnerName)
        }
      }
      this.wheelEntries = this.generateWheelEntries() // Regenerate wheel without the winner's instance
    },

    resetEntries() {
      this.entries = JSON.parse(JSON.stringify(this.originalEntries)); // Restore from original
      this.wheelEntries = this.generateWheelEntries();
      this.winner = null;
      this.rotation = 0;
      this.isSpinning = false;
      this.selectedFile = null;
      this.googleSheetUrl = '';
      this.sheetImportStatus = '';
    },

    // Wheel drawing methods
    getSegmentPath(index) {
      const angle = this.segmentAngle
      const startAngleRad = (index * angle - 90) * (Math.PI / 180) // -90 to start from top
      const endAngleRad = ((index + 1) * angle - 90) * (Math.PI / 180)
      
      const x1 = this.wheelRadius + this.wheelRadius * Math.cos(startAngleRad)
      const y1 = this.wheelRadius + this.wheelRadius * Math.sin(startAngleRad)
      const x2 = this.wheelRadius + this.wheelRadius * Math.cos(endAngleRad)
      const y2 = this.wheelRadius + this.wheelRadius * Math.sin(endAngleRad)
      
      const largeArcFlag = angle > 180 ? 1 : 0
      
      return `M ${this.wheelRadius},${this.wheelRadius} L ${x1},${y1} A ${this.wheelRadius},${this.wheelRadius} 0 ${largeArcFlag} 1 ${x2},${y2} Z`
    },

    getColor(index) {
      const colors = [
        '#FFC107', '#2196F3', '#4CAF50', '#9C27B0', 
        '#FF5722', '#00BCD4', '#E91E63', '#673AB7', 
        '#FF9800', '#8BC34A'
      ]
      
      // This ensures adjacent segments will have different colors
      // by alternating between even and odd indices in the colors array
      if (index % 2 === 0) {
        return colors[(index / 2) % (colors.length / 2)]
      } else {
        return colors[Math.floor(colors.length / 2) + Math.floor(index / 2) % Math.floor(colors.length / 2)]
      }
    },

    getTextX(index) {
      const angle = this.segmentAngle
      const textAngleRad = ((index + 0.5) * angle - 90) * (Math.PI / 180)
      // Position text about 60-70% out from the center
      return this.wheelRadius + (this.wheelRadius * 0.65) * Math.cos(textAngleRad)
    },

    getTextY(index) {
      const angle = this.segmentAngle
      const textAngleRad = ((index + 0.5) * angle - 90) * (Math.PI / 180)
      return this.wheelRadius + (this.wheelRadius * 0.65) * Math.sin(textAngleRad)
    },

    getTextRotation(index) {
      const angle = this.segmentAngle
      return (index + 0.5) * angle // Rotate text to align with segment
    },

    getTextSize() {
      // Adjust text size based on number of entries to prevent overlap
      if (this.wheelEntries.length > 20) return '10px'
      if (this.wheelEntries.length > 10) return '12px'
      return '14px'
    },

    getEntryCount(name) {
      const entry = this.entries.find(e => e.name === name);
      return entry ? entry.count : 0;
    },
    
    toggleTheme() {
      this.isDarkMode = !this.isDarkMode;
      if (typeof localStorage !== 'undefined') {
        localStorage.setItem('theme', this.isDarkMode ? 'dark' : 'light');
      }
    }
  },
  mounted() {
    if (typeof localStorage !== 'undefined') {
      const savedTheme = localStorage.getItem('theme');
      if (savedTheme) {
        this.isDarkMode = savedTheme === 'dark';
      }
    }
    // Initialize with some example data if needed for development, or ensure it's empty
    // this.entries = [{ name: 'Sample1', count: 2 }, { name: 'Sample2', count: 3 }];
    // this.originalEntries = JSON.parse(JSON.stringify(this.entries));
    // this.wheelEntries = this.generateWheelEntries();
  }
}
</script>

<style>
/* Global Styles */
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin: 0;
  background-color: #f4f7f6; /* Light grayish background */
  color: #333;
  transition: background-color 0.3s, color 0.3s;
}

#app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* Header */
.header {
  background-color: #4A90E2; /* A calm blue */
  color: white;
  padding: 20px 40px;
  text-align: center;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.header .title {
  margin: 0;
  font-size: 2.5em;
  font-weight: 600;
}

.header .title .icon {
  margin-right: 10px;
}

.header .subtitle {
  margin: 5px 0 0;
  font-size: 1.1em;
  font-weight: 300;
}

/* Main Container */
.main-container {
  display: flex;
  flex-grow: 1;
  padding: 20px;
  gap: 20px;
}

/* Sidebar */
.sidebar {
  width: 350px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

/* Main Content */
.main-content {
  flex-grow: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Cards */
.card {
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  overflow: hidden;
  transition: background-color 0.3s, border-color 0.3s, box-shadow 0.3s;
}

.card-header {
  background-color: #f9f9f9; /* Slightly off-white for header */
  padding: 15px 20px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card-header h3 {
  margin: 0;
  font-size: 1.2em;
  color: #333;
}

.card-body {
  padding: 20px;
}

/* File Upload */
.file-upload-area {
  border: 2px dashed #ccc;
  border-radius: 6px;
  padding: 30px;
  text-align: center;
  cursor: pointer;
  margin-bottom: 15px;
  transition: border-color 0.3s, background-color 0.3s;
}

.file-upload-area.drag-over {
  border-color: #4A90E2;
  background-color: #e9f2fc;
}

.upload-icon {
  font-size: 3em;
  color: #4A90E2;
  margin-bottom: 10px;
}

.upload-text {
  font-size: 1.1em;
  font-weight: 500;
  margin: 5px 0;
}

.upload-subtext {
  font-size: 0.9em;
  color: #777;
}

.file-input {
  display: none; /* Hidden, triggered by area click */
}

.file-info {
  margin-top: 10px;
  font-size: 0.9em;
  color: #555;
  text-align: center;
}

.file-name {
  font-weight: 500;
}

/* Buttons */
.btn {
  padding: 12px 20px;
  font-size: 1em;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.2s, transform 0.1s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.btn:disabled {
  background-color: #ccc;
  cursor: not-allowed;
  color: #888;
}

.btn-full {
  width: 100%;
}

.btn-primary {
  background-color: #2196F3; /* Blue */
  color: white;
}
.btn-primary:not(:disabled):hover {
  background-color: #357ABD;
}

.btn-success {
  background-color: #4CAF50; /* Green */
  color: white;
}
.btn-success:not(:disabled):hover {
  background-color: #4CAE4C;
}

.btn-danger {
  background-color: #F44336; /* Red */
  color: white;
}
.btn-danger:not(:disabled):hover {
  background-color: #C9302C;
}

.btn-secondary {
  background-color: #6c757d;
  color: white;
}
.btn-secondary:not(:disabled):hover {
  background-color: #5a6268;
}

/* Entries Display */
.entries-card .badge {
  background-color: #e9f2fc;
  color: #4A90E2;
  padding: 5px 10px;
  border-radius: 12px;
  font-size: 0.8em;
  font-weight: 500;
}

.entries-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  max-height: 200px; /* Or adjust as needed */
  overflow-y: auto;
}

.entry-chip {
  background-color: #f0f0f0;
  border-radius: 15px;
  padding: 5px 12px;
  display: flex;
  align-items: center;
  font-size: 0.9em;
}

.entry-name {
  font-weight: 500;
  margin-right: 6px;
}

.entry-count {
  background-color: #4A90E2;
  color: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8em;
}

/* Controls */
.controls-card .controls {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
}

.no-entries .empty-state {
  text-align: center;
  padding: 20px;
  background-color: #f9f9f9;
  border-radius: 6px;
}

.empty-icon {
  font-size: 3em;
  margin-bottom: 10px;
}

/* Wheel Section */
.wheel-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.wheel-container {
  position: relative;
  width: calc(var(--wheel-radius, 280px) * 2);
  height: calc(var(--wheel-radius, 280px) * 2);
  display: flex;
  justify-content: center;
  align-items: center;
}

.wheel-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
}

.wheel-svg {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  transition: transform 0s linear; /* Continuous for JS animation */
  /* box-shadow: 0 0 20px rgba(0,0,0,0.2); */
}

.wheel-svg.spinning {
  /* Handled by JS, but could add a base transition if preferred for CSS-only spin start/stop */
}

.wheel-segment {
  cursor: pointer; /* Optional: if segments are interactive */
  transition: opacity 0.2s;
}
/* .wheel-segment:hover {
  opacity: 0.8;
} */

.segment-text {
  font-family: 'Arial Black', Gadget, sans-serif;
  fill: white;
  text-anchor: middle;
  dominant-baseline: middle;
  pointer-events: none; /* Text should not interfere with clicks on segments */
  font-weight: bold;
  text-shadow: 1px 1px 1px rgba(0,0,0,0.3);
}

.wheel-center {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 60px; /* Size of the center circle */
  height: 60px;
  background-color: #fff;
  border-radius: 50%;
  box-shadow: 0 0 10px rgba(0,0,0,0.2);
  display: flex;
  justify-content: center;
  align-items: center;
}

.center-circle {
  width: 40px;
  height: 40px;
  background-color: #e0e0e0;
  border-radius: 50%;
  border: 5px solid #fff;
}

.pointer {
  position: absolute;
  top: -10px; /* Position above the wheel */
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 15px solid transparent;
  border-right: 15px solid transparent;
  border-bottom: 25px solid #D9534F; /* Red pointer */
  z-index: 10;
}

/* Winner Announcement */
.winner-announcement {
  margin-top: 20px;
}

.winner-card {
  background-color: #fff;
  padding: 25px 30px;
  border-radius: 8px;
  box-shadow: 0 5px 15px rgba(0,0,0,0.15);
  text-align: center;
  border-top: 5px solid #FFC107; /* Gold accent */
}

.winner-icon {
  font-size: 3em;
  margin-bottom: 10px;
}

.winner-name {
  font-size: 2em;
  color: #333;
  margin: 5px 0;
}

.winner-text {
  font-size: 1.1em;
  color: #555;
  margin-bottom: 5px;
}

.winner-remaining {
  font-size: 0.9em;
  color: #777;
}

/* Welcome Message (Placeholder if entries are empty and no welcome card) */
.main-content > div:not(.wheel-section):not(.winner-announcement) {
  /* This is a bit of a hack if the welcome card is removed and entries are empty */
  /* Consider a proper empty state component if the welcome card is permanently gone */
  text-align: center;
  color: #777;
}

/* Example Table (from original welcome message, can be removed if not needed elsewhere) */
.example {
  margin-top: 20px;
  text-align: left;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
}
.example h4 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #333;
}
.example-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9em;
}
.example-table th, .example-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}
.example-table th {
  background-color: #f0f0f0;
  font-weight: 600;
}

/* Styles for new elements */
/* Styles for new elements */
.sheet-import-card .sheet-input {
  width: 100%; /* Make it full width of container */
  padding: 10px;
  margin-bottom: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
}

.import-status {
  margin-top: 10px;
  font-size: 0.9em;
  color: #555;
}

.theme-toggle-btn {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 10px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  z-index: 1000;
  box-shadow: 0 2px 5px rgba(0,0,0,0.2);
}

.theme-toggle-btn:hover {
  background-color: #0056b3;
}

/* Dark Mode Styles */
#app.dark-mode body, /* Apply to body as well if #app doesn't cover everything */
#app.dark-mode {
  background-color: #121212; /* Darker background for the whole app */
  color: #e0e0e0; /* Lighter text for dark mode */
}

#app.dark-mode .header {
  background-color: #2d2d2d; /* Darker header */
  border-bottom: 1px solid #444;
  box-shadow: 0 2px 4px rgba(0,0,0,0.3);
}

#app.dark-mode .header .title,
#app.dark-mode .header .subtitle {
  color: #f0f0f0;
}

#app.dark-mode .card {
  background-color: #1e1e1e; /* Darker cards */
  border: 1px solid #444;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25);
}

#app.dark-mode .card-header {
  background-color: #3a3a3a;
  border-bottom: 1px solid #555;
}

#app.dark-mode .card-header h3 {
  color: #e0e0e0;
}

#app.dark-mode .card-body p,
#app.dark-mode .card-body .upload-text,
#app.dark-mode .card-body .upload-subtext,
#app.dark-mode .file-info .file-name,
#app.dark-mode .entry-chip .entry-name,
#app.dark-mode .winner-text,
#app.dark-mode .winner-remaining,
#app.dark-mode .example h4,
#app.dark-mode .example-table th,
#app.dark-mode .example-table td,
#app.dark-mode .import-status,
#app.dark-mode .no-entries p,
#app.dark-mode .no-entries h3 {
  color: #c0c0c0;
}

#app.dark-mode .file-upload-area {
  border-color: #555;
}
#app.dark-mode .file-upload-area.drag-over {
  border-color: #007bff;
  background-color: #2a3a4a;
}
#app.dark-mode .upload-icon {
  color: #007bff;
}

#app.dark-mode .btn-primary {
  background-color: #0056b3;
  border-color: #004085;
}
#app.dark-mode .btn-primary:not(:disabled):hover {
  background-color: #00376b;
}

#app.dark-mode .btn-success {
  background-color: #1e7e34;
  border-color: #155724;
}
#app.dark-mode .btn-success:not(:disabled):hover {
  background-color: #124f21;
}

#app.dark-mode .btn-danger {
  background-color: #a71d2a;
  border-color: #721c24;
}
#app.dark-mode .btn-danger:not(:disabled):hover {
  background-color: #8c1620;
}

#app.dark-mode .btn-secondary {
  background-color: #495057;
  border-color: #383d41;
}
#app.dark-mode .btn-secondary:not(:disabled):hover {
  background-color: #313539;
}

#app.dark-mode .btn:disabled {
  background-color: #444;
  color: #777;
}

#app.dark-mode .sheet-input {
  background-color: #333;
  color: #e0e0e0;
  border-color: #555;
}
#app.dark-mode .sheet-input::placeholder {
  color: #888;
}

#app.dark-mode .entries-card .badge {
  background-color: #3a4a5a;
  color: #a0c0e0;
}

#app.dark-mode .entry-chip {
  background-color: #3f3f3f;
}

#app.dark-mode .entry-count {
  background-color: #007bff;
}

#app.dark-mode .no-entries .empty-state {
  background-color: #3a3a3a;
}

#app.dark-mode .wheel-segment {
  /* stroke: #2d2d2d; Dark mode might need different segment stroke */
}

#app.dark-mode .segment-text {
  /* fill: #1e1e1e; /* Or a light color that contrasts with dark segments */
  /* text-shadow: 1px 1px 1px rgba(255,255,255,0.2); */
}

#app.dark-mode .wheel-center {
  background-color: #444;
  box-shadow: 0 0 10px rgba(0,0,0,0.5);
}
#app.dark-mode .center-circle {
  background-color: #555;
  border-color: #444;
}

#app.dark-mode .pointer {
  border-bottom-color: #c9302c; /* Darker red for pointer */
}

#app.dark-mode .winner-card {
  background-color: #2d2d2d;
  border-top-color: #b8860b; /* Darker gold */
  box-shadow: 0 5px 15px rgba(0,0,0,0.4);
}
#app.dark-mode .winner-name {
  color: #e0e0e0;
}

#app.dark-mode .example {
  background-color: #3a3a3a;
  border-color: #555;
}
#app.dark-mode .example-table th {
  background-color: #484848;
}
#app.dark-mode .example-table th,
#app.dark-mode .example-table td {
  border-color: #666;
}

#app.dark-mode .theme-toggle-btn {  
  background-color: #4a4a4a;
  color: #e0e0e0;
}
#app.dark-mode .theme-toggle-btn:hover {
  background-color: #333;
}

/* Ensure scrollbars are styled for dark mode if they become too obtrusive */
#app.dark-mode ::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}
#app.dark-mode ::-webkit-scrollbar-track {
  background: #2d2d2d;
}
#app.dark-mode ::-webkit-scrollbar-thumb {
  background: #555;
  border-radius: 4px;
}
#app.dark-mode ::-webkit-scrollbar-thumb:hover {
  background: #666;
}

/* Larger elements when no file is imported */
.no-data-state .card {
  width: 100%;
  max-width: 600px; /* Increased from default */
  margin: 0 auto;
}

.no-data-state .file-upload-area {
  padding: 50px; /* Increased padding */
  margin-bottom: 25px;
}

.no-data-state .upload-icon {
  font-size: 4em; /* Larger icon */
  margin-bottom: 20px;
}

.no-data-state .upload-text {
  font-size: 1.4em; /* Larger text */
}

.no-data-state .upload-subtext {
  font-size: 1.1em;
}

.no-data-state .sheet-input {
  padding: 15px;
  font-size: 1.1em;
}

.no-data-state .btn {
  padding: 15px 25px;
  font-size: 1.2em;
}
</style>