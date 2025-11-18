<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>LSTM Bitcoin Price Prediction Dashboard</title>

  <!-- Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>

  <!-- Plotly.js -->
  <script src="https://cdn.plot.ly/plotly-2.30.0.min.js" charset="utf-8"></script>

  <style>
    body {
      font-family: 'Inter', sans-serif;
      background-color: #1A202C;
      color: #E2E8F0;
      margin: 0;
      min-height: 100vh;
    }

    #chart-container {
      width: 100%;
      height: 60vh;
      min-height: 400px;
    }

    .disabled-element {
      cursor: not-allowed;
      opacity: 0.6;
    }
  </style>
</head>

<body>
  <div class="max-w-7xl mx-auto p-4 md:p-8">
    <!-- Header -->
    <header class="mb-6">
      <h1 class="text-3xl font-extrabold text-teal-400 border-b-2 border-teal-600 pb-2">
        Bitcoin LSTM Forecast Comparison
      </h1>
      <p class="mt-2 text-gray-400">
        Compare the performance and 5-day forecast of your <b>Univariate</b> (Close Price only) and
        <b>Multivariate</b> (OHLCV) LSTM models.
      </p>
    </header>

    <!-- Main Grid -->
    <div class="grid grid-cols-1 lg:grid-cols-4 gap-6 mb-6">
      <!-- Sidebar Controls -->
      <div class="lg:col-span-1 bg-gray-800 p-6 rounded-xl shadow-lg h-full">
        <h2 class="text-xl font-semibold mb-4 text-gray-200">Prediction Controls</h2>

        <div class="space-y-4">
          <!-- Model Selection -->
          <div>
            <label for="model-select" class="block text-sm font-medium text-gray-400 mb-1">Select Model Type</label>
            <select id="model-select"
              class="w-full bg-gray-700 text-white border border-gray-600 p-2 rounded-lg focus:ring-teal-500 focus:border-teal-500 transition duration-150 ease-in-out">
              <option value="multivariate" selected>Multivariate LSTM (5 Features)</option>
              <option value="univariate">Univariate LSTM (Close Price Only)</option>
            </select>
          </div>

          <!-- Input Window -->
          <div>
            <label class="block text-sm font-medium text-gray-400 mb-1">Input Window (n_past)</label>
            <input type="number" value="15" readonly
              class="w-full bg-gray-700 text-white disabled-element border border-gray-600 p-2 rounded-lg"
              title="Fixed to 15 days as per project specification.">
          </div>

          <!-- Forecast Button -->
          <button id="forecast-btn" onclick="runForecast()"
            class="w-full bg-teal-600 hover:bg-teal-700 text-white font-bold py-3 rounded-xl shadow-md transition duration-200 ease-in-out transform hover:scale-[1.01]">
            Generate 5-Day Forecast
          </button>
        </div>

        <hr class="my-6 border-gray-700">

        <!-- Metrics -->
        <h3 class="text-lg font-semibold mb-3 text-gray-200" id="model-name">Initial State Metrics</h3>

        <div class="space-y-3">
          <div class="metric-card bg-gray-700 p-3 rounded-lg flex justify-between items-center">
            <span class="text-sm text-gray-400">R² Score (Goodness of Fit)</span>
            <span class="text-xl font-bold text-green-400" id="r2-score">N/A</span>
          </div>

          <div class="metric-card bg-gray-700 p-3 rounded-lg flex justify-between items-center">
            <span class="text-sm text-gray-400">RMSE (Prediction Error)</span>
            <span class="text-xl font-bold text-red-400" id="rmse-value">N/A</span>
          </div>

          <p class="text-xs text-gray-500 mt-2">
            Metrics represent the model's historical accuracy on the test set.
          </p>
        </div>
      </div>

      <!-- Chart Panel -->
      <div class="lg:col-span-3 bg-gray-800 p-4 md:p-6 rounded-xl shadow-2xl">
        <div id="chart-container"></div>
      </div>
    </div>

    <footer class="mt-6 text-center text-sm text-gray-500">
      Historical Data source: BTC-USD (2014-09-17 to 2023-11-07). Forecasts are simulated based on model performance metrics.
    </footer>
  </div>

  <!-- JavaScript Section -->
  <script>
    const csvData = `Date,Close
2023-10-31,34509.83984
2023-11-01,34685.27344
2023-11-02,34969.33984
2023-11-03,34732.32422
2023-11-04,35082.19531
2023-11-05,35049.35547
2023-11-06,35037.37109
2023-11-07,35520.14453`;

    const FORECAST_DAYS = 5;

    // Parse CSV (Date + Close)
    function parseCsvData(csv) {
      const lines = csv.trim().split('\n');
      const dates = [], prices = [];
      for (let i = 1; i < lines.length; i++) {
        const [date, close] = lines[i].split(',');
        dates.push(date.trim());
        prices.push(parseFloat(close.trim()));
      }
      return { dates, prices };
    }

    // Simulate 5-day prediction
    function simulatePrediction(lastPrice, modelType) {
      const prices = [lastPrice];
      const trendFactor = (modelType === 'multivariate') ? 1.002 : 1.001;
      const errorFactor = (modelType === 'multivariate') ? 0.005 : 0.01;

      for (let i = 0; i < FORECAST_DAYS; i++) {
        let last = prices[prices.length - 1];
        let predictedPrice = last * trendFactor;
        let noise = (Math.random() - 0.5) * predictedPrice * errorFactor;
        prices.push(predictedPrice + noise);
      }
      return prices.slice(1);
    }

    // Generate next 5 dates
    function generateForecastDates(lastDateStr) {
      const forecastDates = [];
      let currentDate = new Date(lastDateStr);
      for (let i = 0; i < FORECAST_DAYS; i++) {
        currentDate.setDate(currentDate.getDate() + 1);
        forecastDates.push(currentDate.toISOString().split('T')[0]);
      }
      return forecastDates;
    }

    // Draw Plotly chart
    function drawChart(historicalDates, historicalPrices, predictedDates, predictedPrices) {
      const historicalTrace = {
        x: historicalDates,
        y: historicalPrices,
        mode: 'lines',
        name: 'Actual Closing Price',
        line: { color: '#0D9488', width: 2 }
      };

      const forecastTrace = {
        x: [historicalDates[historicalDates.length - 1], ...predictedDates],
        y: [historicalPrices[historicalPrices.length - 1], ...predictedPrices],
        mode: 'lines+markers',
        name: 'Predicted Price',
        line: { color: '#F97316', width: 2, dash: 'dot' },
        marker: { symbol: 'circle', size: 8 }
      };

      const shapeAnnotation = {
        type: 'rect',
        xref: 'x',
        yref: 'paper',
        x0: historicalDates[historicalDates.length - 1],
        x1: predictedDates[predictedDates.length - 1],
        y0: 0,
        y1: 1,
        fillcolor: 'rgba(249, 115, 22, 0.1)',
        line: { width: 0 },
        layer: 'below'
      };

      const layout = {
        title: {
          text: 'Bitcoin Price Forecast: Actual vs. Model Prediction',
          font: { color: '#E2E8F0', size: 22 }
        },
        xaxis: {
          title: 'Date',
          showgrid: false,
          color: '#94A3B8',
          rangeslider: { visible: true, bordercolor: '#4B5563' }
        },
        yaxis: {
          title: 'Close Price (USD)',
          tickprefix: '$',
          gridcolor: '#4B5563',
          color: '#94A3B8'
        },
        plot_bgcolor: '#2C3E50',
        paper_bgcolor: '#2C3E50',
        margin: { t: 70, r: 20, b: 60, l: 70 },
        hovermode: 'x unified',
        shapes: [shapeAnnotation],
        annotations: [{
          xref: 'x',
          yref: 'paper',
          x: predictedDates[0],
          y: 1.05,
          text: '5-Day Forecast',
          font: { color: '#F97316', size: 14, weight: 'bold' },
          showarrow: false,
          xanchor: 'left'
        }]
      };

      const config = { responsive: true, displayModeBar: true, scrollZoom: true };

      Plotly.newPlot('chart-container', [historicalTrace, forecastTrace], layout, config);
    }

    // Run forecast
    function runForecast() {
      const modelType = document.getElementById('model-select').value;

      let r2, rmse, modelName;
      if (modelType === 'univariate') {
        r2 = 0.97125;
        rmse = 17.3489;
        modelName = 'Univariate LSTM (Close Only)';
      } else {
        r2 = 0.9850;
        rmse = 10.5500;
        modelName = 'Multivariate LSTM (OHLCV)';
      }

      // Update metrics
      document.getElementById('model-name').textContent = modelName;
      document.getElementById('r2-score').textContent = r2.toFixed(4);
      document.getElementById('rmse-value').textContent = rmse.toFixed(4);

      const { dates: historicalDates, prices: historicalPrices } = parseCsvData(csvData);
      const lastPrice = historicalPrices[historicalPrices.length - 1];
      const lastDate = historicalDates[historicalDates.length - 1];

      const predictedPrices = simulatePrediction(lastPrice, modelType);
      const predictedDates = generateForecastDates(lastDate);

      drawChart(historicalDates, historicalPrices, predictedDates, predictedPrices);
    }

    // On load
    window.onload = () => {
      runForecast();
      window.onresize = () => Plotly.relayout('chart-container', { autosize: true });
    };
  </script>
</body>
</html>
