# Windy API Integration Setup

## Getting Your Windy API Key

1. Visit [Windy API](https://api.windy.com/) 
2. Sign up for a free account
3. Go to your dashboard and create a new API key
4. Copy your API key

## Configuration

1. Open `.env.local` in your project root
2. Replace `YOUR_WINDY_API_KEY` with your actual API key:
   ```
   VITE_WINDY_API_KEY=your_actual_api_key_here
   ```
3. Save the file and restart your development server

## Features

The Windy integration provides:

- **Real-time wind data** - Current wind speed and direction
- **Wave/swell data** - Wave height, direction, and period
- **Interactive map** - Full Windy weather map embedded in your app
- **Smart matching** - Automatically detects if wind/swell aligns with your swell windows
- **Fallback mode** - Uses simulated data when API key is not configured

## Usage

1. Select a spot on the main map
2. Draw some swell windows
3. Click "Show Wind" or "Show Waves/Swell" in the Windy Integration panel
4. The component will show current conditions and whether they match your swell windows

## API Limits

- **Free tier**: 1000 requests per month
- **Wind data**: Available on free tier
- **Wave/swell data**: Available on free tier (using ECMWF, GFS, or ICON wave models)
- Check your usage at [Windy API Dashboard](https://api.windy.com/dashboard)

## Wave Data Access

✅ **Wave/swell data is available with free API keys!**

The app automatically tries multiple wave data sources:
- `ecmwfWaves` - European Centre for Medium-Range Weather Forecasts
- `gfsWaves` - Global Forecast System 
- `iconEuWaves` - ICON European waves
- `iconWaves` - ICON Global waves

The component will automatically select the best available wave overlay for your API key.

## Troubleshooting

If you see "Loading Windy data..." for a long time:
1. Check that your API key is correctly set in `.env.local`
2. Verify the API key works by visiting the Windy API dashboard
3. Check browser console for any error messages
4. The component will automatically fall back to demo data if the API fails

### Common Error: "No products available for overlay waves"
This means the generic "waves" overlay isn't available, but the app will automatically try specific wave models like `ecmwfWaves` or `gfsWaves` which should work with free API keys.

### Common Error: "Missing <div id="windy"></div>"
This error has been fixed in the current implementation. The app automatically includes the required div element that Windy needs.

### API Key Issues
- Make sure your API key doesn't have any extra spaces
- Restart your development server after changing the `.env.local` file
- Check that the API key is active in your Windy dashboard
