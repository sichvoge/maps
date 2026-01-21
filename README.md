# GeoMap Request Analytics

A Vue.js application that displays a geographic map showing the number of requests by country using Chart.js and chartjs-chart-geo.

## Features

- Interactive world map visualization using Chart.js
- Color-coded countries based on request volume
- Configurable country data via external JSON file
- Responsive design with dark mode support
- Top 5 countries legend display

## Project Structure

```
maps/
├── public/
│   └── country-data.json    # Configuration file for country request data
├── src/
│   ├── components/
│   │   └── GeoMap.vue       # Main geo map component
│   ├── App.vue              # Root application component
│   ├── main.js              # Application entry point
│   └── style.css            # Global styles
├── index.html               # HTML template
├── package.json             # Dependencies
└── vite.config.js           # Vite configuration
```

## Getting Started

### Prerequisites

- Node.js (v16 or higher)
- npm or yarn

### Installation

1. Install dependencies:

```bash
npm install
```

### Development

Run the development server:

```bash
npm run dev
```

The application will be available at `http://localhost:5173`

### Build for Production

```bash
npm run build
```

The production-ready files will be in the `dist/` directory.

## Configuration

### Updating Country Data

Edit the `public/country-data.json` file to change the countries and request counts:

```json
{
  "data": [
    {
      "country": "US",
      "name": "United States",
      "requests": 15420
    },
    {
      "country": "DE",
      "name": "Germany",
      "requests": 8930
    }
  ]
}
```

- `country`: ISO 3166-1 alpha-2 country code (e.g., "US", "DE", "GB")
- `name`: Full country name for display
- `requests`: Number of requests (any positive integer)

### Color Scale

The map uses a dynamic color scale based on request volume:
- **Red** (70-100%): Highest traffic countries
- **Orange** (50-70%): High traffic countries
- **Yellow** (30-50%): Medium-high traffic countries
- **Green** (10-30%): Medium traffic countries
- **Blue** (0-10%): Low traffic countries
- **Gray**: No data available

## Technologies Used

- **Vue 3**: Progressive JavaScript framework
- **Vite**: Next-generation frontend tooling
- **Chart.js**: JavaScript charting library
- **chartjs-chart-geo**: Geographic chart types for Chart.js
- **World Atlas**: TopoJSON world map data

## License

MIT
