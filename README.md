# AI projects
# dashboard.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Executive Sales Dashboard</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        /* Force Chart.js canvases to stay within their grid containers */
        canvas { max-height: 300px; width: 100% !important; }
    </style>
</head>
<body class="bg-gray-50 text-gray-800 font-sans antialiased p-6">

    <div class="max-w-7xl mx-auto space-y-6">
        <div class="bg-white p-4 border border-gray-200 shadow-sm rounded-md flex justify-between items-center">
            <div>
                <h1 class="text-xl font-bold text-gray-900">Executive Sales Analytics</h1>
                <p class="text-xs text-gray-500 mt-1">Upload your sales CSV to generate automated insights.</p>
            </div>
            <div>
                <input type="file" id="csvFileInput" accept=".csv" class="block w-full text-sm text-gray-500
                  file:mr-4 file:py-2 file:px-4
                  file:rounded-md file:border-0
                  file:text-sm file:font-semibold
                  file:bg-blue-50 file:text-blue-700
                  hover:file:bg-blue-100 cursor-pointer" />
            </div>
        </div>

        <div id="dashboard" class="hidden grid grid-cols-1 md:grid-cols-2 gap-6">
            
            <div class="bg-white p-4 border border-gray-200 shadow-sm rounded-md">
                <h2 class="text-sm font-bold uppercase tracking-wider text-gray-700 mb-2">1. Revenue by Category</h2>
                <div class="h-64 mb-4"><canvas id="chartCategory"></canvas></div>
                <div class="text-sm bg-gray-50 p-3 border-l-4 border-blue-500 rounded text-gray-700" id="insightCategory"></div>
            </div>

            <div class="bg-white p-4 border border-gray-200 shadow-sm rounded-md">
                <h2 class="text-sm font-bold uppercase tracking-wider text-gray-700 mb-2">2. YoY Sales Growth</h2>
                <div class="h-64 mb-4"><canvas id="chartYoY"></canvas></div>
                <div class="text-sm bg-gray-50 p-3 border-l-4 border-green-500 rounded text-gray-700" id="insightYoY"></div>
            </div>

            <div class="bg-white p-4 border border-gray-200 shadow-sm rounded-md">
                <h2 class="text-sm font-bold uppercase tracking-wider text-gray-700 mb-2">3. Price vs. Volume</h2>
                <div class="h-64 mb-4"><canvas id="chartElasticity"></canvas></div>
                <div class="text-sm bg-gray-50 p-3 border-l-4 border-purple-500 rounded text-gray-700" id="insightElasticity"></div>
            </div>

            <div class="bg-white p-4 border border-gray-200 shadow-sm rounded-md">
                <h2 class="text-sm font-bold uppercase tracking-wider text-gray-700 mb-2">4. Seasonality (Monthly Avg)</h2>
                <div class="h-64 mb-4"><canvas id="chartSeasonality"></canvas></div>
                <div class="text-sm bg-gray-50 p-3 border-l-4 border-orange-500 rounded text-gray-700" id="insightSeasonality"></div>
            </div>

            <div class="bg-white p-4 border border-gray-200 shadow-sm rounded-md md:col-span-2">
                <h2 class="text-sm font-bold uppercase tracking-wider text-gray-700 mb-2">5. Product Portfolio (Pareto)</h2>
                <div class="h-72 mb-4"><canvas id="chartPareto"></canvas></div>
                <div class="text-sm bg-gray-50 p-3 border-l-4 border-red-500 rounded text-gray-700" id="insightPareto"></div>
            </div>

        </div>
    </div>

    <script>
        // Store chart instances to destroy them on re-upload
        let charts = {};

        document.getElementById('csvFileInput').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (!file) return;

            Papa.parse(file, {
                header: true,
                dynamicTyping: true,
                skipEmptyLines: true,
                complete: function(results) {
                    processData(results.data);
                    document.getElementById('dashboard').classList.remove('hidden');
                }
            });
        });

        const formatCurrency = (val) => new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(val);

        function processData(data) {
            // Clean/Format Data
            const cleaned = data.map(d => ({
                product: d['Product Name'],
                category: d['Category'],
                price: parseFloat(d['Price']),
                quantity: parseInt(d['Quantity']),
                sales: parseFloat(d['Sales Amount']),
                date: new Date(d['Date'])
            })).filter(d => !isNaN(d.sales)); // Remove invalid rows

            generateCategoryInsight(cleaned);
            generateYoYInsight(cleaned);
            generateElasticityInsight(cleaned);
            generateSeasonalityInsight(cleaned);
            generateParetoInsight(cleaned);
        }

        // --- 1. Category Revenue ---
        function generateCategoryInsight(data) {
            const catSales = {};
            let total = 0;
            data.forEach(d => {
                catSales[d.category] = (catSales[d.category] || 0) + d.sales;
                total += d.sales;
            });

            const sorted = Object.entries(catSales).sort((a,b) => b[1] - a[1]);
            const labels = sorted.map(i => i[0]);
            const values = sorted.map(i => i[1]);

            renderChart('chartCategory', 'doughnut', {
                labels,
                datasets: [{ data: values, backgroundColor: ['#3b82f6', '#10b981', '#f59e0b', '#ef4444', '#8b5cf6'] }]
            });

            const topCat = sorted[0];
            const pct = ((topCat[1] / total) * 100).toFixed(1);
            document.getElementById('insightCategory').innerHTML = 
                `<strong>Finding:</strong> <b>${topCat[0]}</b> is the dominant revenue driver, generating <b>${formatCurrency(topCat[1])}</b> (${pct}% of total sales). Focus marketing spend here for the highest proven ROI.`;
        }

        // --- 2. YoY Growth ---
        function generateYoYInsight(data) {
            const yearSales = {};
            data.forEach(d => {
                const yr = d.date.getFullYear();
                if(!isNaN(yr)) yearSales[yr] = (yearSales[yr] || 0) + d.sales;
            });

            const sortedYears = Object.keys(yearSales).sort();
            const values = sortedYears.map(y => yearSales[y]);

            renderChart('chartYoY', 'bar', {
                labels: sortedYears,
                datasets: [{ label: 'Total Revenue', data: values, backgroundColor: '#10b981' }]
            });

            if (sortedYears.length > 1) {
                const first = yearSales[sortedYears[0]];
                const last = yearSales[sortedYears[sortedYears.length-1]];
                const growth = (((last - first) / first) * 100).toFixed(1);
                const trend = growth > 0 ? "grown" : "contracted";
                document.getElementById('insightYoY').innerHTML = 
                    `<strong>Finding:</strong> Revenue has <b>${trend} by ${growth}%</b> from ${sortedYears[0]} to ${sortedYears[sortedYears.length-1]}.`;
            } else {
                document.getElementById('insightYoY').innerHTML = "Not enough multi-year data for YoY trend analysis.";
            }
        }

        // --- 3. Price Elasticity ---
        function generateElasticityInsight(data) {
            const scatterData = data.map(d => ({ x: d.price, y: d.quantity }));
            
            if(charts['chartElasticity']) charts['chartElasticity'].destroy();
            charts['chartElasticity'] = new Chart(document.getElementById('chartElasticity'), {
                type: 'scatter',
                data: { datasets: [{ label: 'Transactions', data: scatterData, backgroundColor: '#8b5cf6' }] },
                options: {
                    scales: {
                        x: { title: { display: true, text: 'Unit Price ($)' } },
                        y: { title: { display: true, text: 'Quantity Sold' } }
                    }
                }
            });

            // Calculate simple correlation/trend
            const avgQ_HighPrice = data.filter(d => d.price > 500).reduce((acc, d) => acc + d.quantity, 0) / (data.filter(d => d.price > 500).length || 1);
            const avgQ_LowPrice = data.filter(d => d.price <= 500).reduce((acc, d) => acc + d.quantity, 0) / (data.filter(d => d.price <= 500).length || 1);

            let insight = "Volume remains relatively stable across price points, indicating low price sensitivity.";
            if (avgQ_LowPrice > avgQ_HighPrice * 1.5) {
                insight = "Clear negative elasticity: Lower-priced items move in significantly higher volumes. Consider bundle pricing for high-tier goods.";
            }
            document.getElementById('insightElasticity').innerHTML = `<strong>Finding:</strong> ${insight}`;
        }

        // --- 4. Seasonality ---
        function generateSeasonalityInsight(data) {
            const monthNames = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
            const monthSales = new Array(12).fill(0);
            
            data.forEach(d => {
                const m = d.date.getMonth();
                if(!isNaN(m)) monthSales[m] += d.sales;
            });

            renderChart('chartSeasonality', 'line', {
                labels: monthNames,
                datasets: [{ label: 'Aggregated Monthly Sales', data: monthSales, borderColor: '#f97316', tension: 0.3, fill: true, backgroundColor: 'rgba(249, 115, 22, 0.1)' }]
            });

            const maxIndex = monthSales.indexOf(Math.max(...monthSales));
            const minIndex = monthSales.indexOf(Math.min(...monthSales));

            document.getElementById('insightSeasonality').innerHTML = 
                `<strong>Finding:</strong> Historical data shows peak purchasing activity in <b>${monthNames[maxIndex]}</b>. The lowest activity is in <b>${monthNames[minIndex]}</b>. Align inventory and marketing campaigns accordingly.`;
        }

        // --- 5. Pareto (80/20) ---
        function generateParetoInsight(data) {
            const prodSales = {};
            let totalSales = 0;
            data.forEach(d => {
                prodSales[d.product] = (prodSales[d.product] || 0) + d.sales;
                totalSales += d.sales;
            });

            const sorted = Object.entries(prodSales).sort((a,b) => b[1] - a[1]);
            const labels = sorted.map(i => i[0]);
            const bars = sorted.map(i => i[1]);
            
            let cumulative = 0;
            const line = sorted.map(i => {
                cumulative += i[1];
                return (cumulative / totalSales) * 100;
            });

            if(charts['chartPareto']) charts['chartPareto'].destroy();
            charts['chartPareto'] = new Chart(document.getElementById('chartPareto'), {
                type: 'bar',
                data: {
                    labels: labels,
                    datasets: [
                        { type: 'line', label: 'Cumulative %', data: line, borderColor: '#ef4444', yAxisID: 'y1' },
                        { type: 'bar', label: 'Sales Revenue', data: bars, backgroundColor: '#cbd5e1', yAxisID: 'y' }
                    ]
                },
                options: {
                    scales: {
                        y: { type: 'linear', position: 'left' },
                        y1: { type: 'linear', position: 'right', max: 100, grid: { drawOnChartArea: false } }
                    }
                }
            });

            let top80Count = line.findIndex(val => val >= 80) + 1;
            if(top80Count === 0) top80Count = sorted.length;
            const pctCatalog = ((top80Count / sorted.length) * 100).toFixed(0);

            document.getElementById('insightPareto').innerHTML = 
                `<strong>Finding:</strong> Just <b>${top80Count} products</b> (${pctCatalog}% of your catalog) generate 80% of total revenue. Prioritize stock stability for these items; consider phasing out the long tail of underperformers.`;
        }

        // Helper to render basic charts
        function renderChart(canvasId, type, dataObj) {
            if(charts[canvasId]) charts[canvasId].destroy();
            charts[canvasId] = new Chart(document.getElementById(canvasId), {
                type: type,
                data: dataObj,
                options: { responsive: true, maintainAspectRatio: false }
            });
        }
    </script>
</body>
</html>


