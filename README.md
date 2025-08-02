# 🌊 Swell Window Calculator

A modern Vue.js application for calculating swell exposure windows at coastal locations. Select a spot on an interactive map and draw swell windows to determine the degrees of swell exposure available to that location.

## 🚀 Features

- **Interactive Spot Selection**: Click anywhere to select and name a coastal spot
- **Visual Swell Window Drawing**: Draw triangular swell windows by selecting two points that create openings to open ocean
- **Automatic Angle Calculation**: Automatically calculates exposure angles from drawn windows
- **Exposure Quality Rating**: Get quality ratings based on total swell exposure range
- **Multiple Windows**: Add multiple swell windows for complex coastline configurations
- **Real-time Analysis**: Instant feedback on swell exposure calculations
- **Responsive Design**: Works seamlessly on desktop and mobile devices

## 🎯 How to Use

1. **Select a Spot**: Click anywhere on the map to select a location and give it a name
2. **Draw Swell Windows**: 
   - Click "Draw Swell Window" in the popup or use the controls
   - Click two points to create a triangular opening representing access to open ocean swell
   - The triangle connects your spot to the two points you selected
3. **Add More Windows**: Repeat step 2 for additional swell openings if needed
4. **View Results**: See the calculated exposure angles and quality rating in the right panel

## 📐 Understanding Swell Windows

Swell windows represent the angular range through which ocean swells can reach a specific spot without obstruction. The application:

- Calculates bearing angles from your spot to each window boundary
- Merges overlapping angle ranges
- Provides total exposure range in degrees
- Rates exposure quality from "Limited" to "Excellent"

## 🛠️ Tech Stack

- **Vue 3** with Composition API
- **Vite** for fast development and building
- **Leaflet** for interactive mapping
- **Mathematical calculations** for bearing and angle computations

## 📦 Installation

```bash
npm install
```

## 🏃‍♂️ Development

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) to view it in the browser.

## 🏗️ Build

```bash
npm run build
```

## 🎯 Usage

1. **Select a Location**: Click anywhere on the map to select and name a spot
2. **Draw Swell Windows**: Click "Draw Swell Window" then select two boundary points
3. **Add Multiple Windows**: Repeat for additional swell openings
4. **View Analysis**: The right panel shows:
   - **Number of Windows**: Total swell windows drawn
   - **Total Exposure**: Combined angular range in degrees
   - **Quality Rating**: Overall swell exposure rating
   - **Detailed Angles**: Specific bearing ranges for each window

## 🔮 Future Enhancements

- Integration with real bathymetry data
- Export functionality for swell window configurations
- Saved locations and user accounts
- Advanced filtering for different swell periods
- Integration with wave forecasting APIs
- Coastal obstruction detection

## 📱 Mobile Support

The application is fully responsive and optimized for mobile devices, making it perfect for checking conditions on the go.

---

Built with ❤️ using Vue 3 and Vite
# swell-exposure
