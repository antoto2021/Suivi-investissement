<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Suivi des investissements</title>
  <!-- Chart.js + DataLabels -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels"></script>
  <style>
    /* ----- Variables de taille de police ----- */
    :root {
      --font-body: 10px;
      --font-form: 1.1em;
      --font-table: 1.3em;
      --font-summary: 1.5em;
      --font-legend: 12px;
      --font-title-chart: 18px;
      --font-axis: 15px;
      --font-datalabel: 20px;
      --font-results: 1.5em;
    }
    body { font-family: Arial, sans-serif; font-size: var(--font-body); margin: 20px; background: #f5f5f5; }
    h1, h3 { text-align: center; font-size: var(--font-title-chart); }
    .form-container { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; background: #fff; padding: 5px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); margin-bottom: 20px; }
    .form-container input, .form-container button { padding: 8px; border: 1px solid #ccc; border-radius: 4px; font-size: var(--font-form); }
    .form-container button { background: #007BFF; color: #fff; border: none; cursor: pointer; }
    .form-container button:hover { background: #0056b3; }
	
    #searchInput { width: 30%; padding: 5px; margin-bottom: 15px; font-size: var(--font-form); box-sizing: border-box; }
	
    table { width: 100%; border-collapse: collapse; margin-bottom: 20px; background: #fff; font-size: var(--font-table); }
    th, td { padding: 12px; border-bottom: 1px solid #ddd; text-align: left; }
    th { background: #f0f0f0; cursor: pointer; }
	
    #summary { font-weight: bold; margin-bottom: 20px; font-size: var(--font-summary); }
	
    .charts-wrapper {
      display: flex;
      gap: 20px;
      flex-wrap: wrap;
      margin-bottom: 30px;
      /* each chart takes up equal width */
      justify-content: space-between;
    }
    .chart-container {
      background: #fff;
      padding: 5px;
      border-radius: 8px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
      /* fixed flex basis to avoid overflow */
      flex: 1 1 calc(50% - 20px);
      position: relative;
      height: 200px;
      overflow: hidden;
    }
    .chart-container { background: #fff; padding: 15px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); flex: 1; position: relative; height: 500px; }
    .pie-container { flex: 0 0 30%;   height: 500px;  padding: 15px;}
    .chart-container canvas { width: 100% !important; height: 85% !important; }
  </style>
</head>
<body>
  <h1>Suivi de mes investissements</h1>

  <div class="form-container">
    <input type="text" id="entity" placeholder="Entité">
    <input type="text" id="name" placeholder="Nom">
    <input type="date" id="date">
    <input type="number" id="amount" placeholder="Montant (€)" step="0.01">
    <input type="number" id="return" placeholder="Rendement (%)" step="0.01">
    <button onclick="saveInvestment()" id="saveBtn">Ajouter</button>
  </div>

  <input type="text" id="searchInput" onkeyup="filterTable()" placeholder="Rechercher...">

  <table id="investmentsTable">
    <thead>
      <tr>
        <th onclick="sortTable(0)">Entité</th>
        <th onclick="sortTable(1)">Nom</th>
        <th onclick="sortTable(2)">Date</th>
        <th onclick="sortTable(3)">Montant (€)</th>
        <th onclick="sortTable(4)">Rendement (%)</th>
        <th>Valeur future (€)</th>
        <th>Actions</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <div id="summary"></div>

  <div class="charts-wrapper">
    <div class="chart-container pie-container">
      <h3>Répartition par entité</h3>
      <canvas id="pieChart"></canvas>
    </div>
    <div class="chart-container">
      <h3>Projection du capital composé</h3>
      <label for="projectionYears">Années :</label>
      <input type="number" id="projectionYears" value="20" min="1" onchange="renderProjectionChart()">
      <canvas id="projectionChart"></canvas>
    </div>
  </div>

  <script>
        const fontConfig = {
      legend: parseInt(getComputedStyle(document.documentElement).getPropertyValue('--font-legend')),
      title: parseInt(getComputedStyle(document.documentElement).getPropertyValue('--font-title-chart')),
      axis: parseInt(getComputedStyle(document.documentElement).getPropertyValue('--font-axis')),
      datalabel: parseInt(getComputedStyle(document.documentElement).getPropertyValue('--font-datalabel'))
    };
    const msPerYear = 1000 * 60 * 60 * 24 * 365;
    // Données initiales ou récupérées du localStorage
    const stored = localStorage.getItem('investments');
    let investments = stored
      ? JSON.parse(stored)
      : [
          {entity:'Castor Vinci',name:'Action Vinci',date:'2024-01-01',amount:5446,return:0},
          {entity:'Obligation immobilier',name:'',date:'2025-01-01',amount:0,return:11},
          {entity:'Obligation immobilier',name:'',date:'2025-01-01',amount:0,return:11},
          {entity:'CTO bourse',name:'Ishares Physical Gold ETC',date:'2025-01-01',amount:1200,return:9.56},
          {entity:'CTO bourse',name:'NASDAQ 100',date:'2025-01-01',amount:500,return:22.19},
          {entity:'CTO bourse',name:'NASDAQ 100',date:'2024-01-01',amount:500,return:22.19},
          {entity:'CTO bourse',name:'S&P 500',date:'2024-01-01',amount:478,return:10.5},
          {entity:'CTO bourse',name:'Neurones',date:'2024-01-01',amount:202,return:8},
          {entity:'CTO bourse',name:'Accor',date:'2024-01-01',amount:120,return:8},
          {entity:'CTO bourse',name:'Mercedes',date:'2024-01-01',amount:95,return:8},
          {entity:'CTO bourse',name:'Rocket Lab',date:'2024-01-01',amount:100,return:8},
          {entity:'PEA',name:'S&P 500',date:'2025-01-01',amount:460,return:10.5}
        ];
    // Enregistre dans localStorage
    function saveLocal() {
      localStorage.setItem('investments', JSON.stringify(investments));
    }
    let editingIndex = null;
    let pieChart, projChart;

    document.addEventListener('DOMContentLoaded', () => {
      renderTable(); renderPie(); renderProjectionChart();
    });

    // CRUD functions
    function saveInvestment() {
      const e = document.getElementById('entity').value;
      const n = document.getElementById('name').value;
      const d = document.getElementById('date').value;
      const a = parseFloat(document.getElementById('amount').value);
      const r = parseFloat(document.getElementById('return').value);
      if (editingIndex !== null) {
        investments[editingIndex] = { entity: e, name: n, date: d, amount: a, return: r };
        editingIndex = null;
        document.getElementById('saveBtn').textContent = 'Ajouter';
      } else {
        investments.push({ entity: e, name: n, date: d, amount: a, return: r });
      }
      saveLocal();             // <-- save to localStorage
      resetForm(); renderTable(); renderPie(); renderProjectionChart();
    };
        editingIndex = null;
        document.getElementById('saveBtn').textContent = 'Ajouter';
      } else {
        investments.push({ entity: e, name: n, date: d, amount: a, return: r });
      }
      resetForm(); renderTable(); renderPie(); renderProjectionChart();
    }
    function startEdit(i) {
      const inv = investments[i];
      ['entity','name','date','amount','return'].forEach(f => document.getElementById(f).value = inv[f]);
      editingIndex = i;
      document.getElementById('saveBtn').textContent = 'Enregistrer';
    }
    function deleteInvestment(i) { investments.splice(i,1); renderTable(); renderPie(); renderProjectionChart(); }
    function resetForm() { ['entity','name','date','amount','return'].forEach(f => document.getElementById(f).value = ''); }
    function sortTable(col) {
      const keys = ['entity','name','date','amount','return'];
      investments.sort((a,b) => {
        let x = a[keys[col]], y = b[keys[col]];
        if (col === 2) return new Date(x) - new Date(y);
        if (col >= 3 && col <= 4) return x - y;
        return x.localeCompare(y);
      });
      renderTable(); renderPie(); renderProjectionChart();
    }
    function filterTable() {
      const f = document.getElementById('searchInput').value.toLowerCase();
      document.querySelectorAll('#investmentsTable tbody tr').forEach(r => r.style.display = r.textContent.toLowerCase().includes(f) ? '' : 'none');
    }

    // Table rendering
    function renderTable() {
      const tbody = document.querySelector('#investmentsTable tbody');
      tbody.innerHTML = '';
      const now = new Date();
      investments.forEach((inv,i) => {
        const yrs = (now - new Date(inv.date)) / msPerYear;
        const future = inv.amount * Math.pow(1 + inv.return/100, yrs);
        const tr = document.createElement('tr');
        tr.innerHTML = `
          <td>${inv.entity}</td>
          <td>${inv.name}</td>
          <td>${inv.date}</td>
          <td>${inv.amount.toLocaleString('fr-FR',{minimumFractionDigits:2})} €</td>
          <td>${inv.return.toFixed(2)}</td>
          <td>${future.toLocaleString('fr-FR',{minimumFractionDigits:2})} €</td>
          <td><button onclick="startEdit(${i})">✏️</button> <button onclick="deleteInvestment(${i})">🗑️</button></td>
        `;
        tbody.appendChild(tr);
      });
      updateSummary();
    }
    function updateSummary() {
      const now = new Date();
      const total = investments.reduce((s,inv) => s + inv.amount, 0);
      const future = investments.reduce((s,inv) => {
        const yrs = (now - new Date(inv.date)) / msPerYear;
        return s + inv.amount * Math.pow(1 + inv.return/100, yrs);
      }, 0);
      document.getElementById('summary').textContent =
        `Total investi: ${total.toLocaleString('fr-FR',{minimumFractionDigits:2})} € · Valeur future: ${future.toLocaleString('fr-FR',{minimumFractionDigits:2})} €`;
    }

    // Graphs
    function renderPie() {
      const agg = {};
      investments.forEach(i => agg[i.entity] = (agg[i.entity]||0) + i.amount);
      const labels = Object.keys(agg), data = Object.values(agg);
      const ctx = document.getElementById('pieChart').getContext('2d');
      if (pieChart) pieChart.destroy();
      pieChart = new Chart(ctx, {
        type: 'pie', data: { labels, datasets: [{ data }] },
        options: {
          maintainAspectRatio: false,
          plugins: { legend: { labels: { font: { size: fontConfig.legend } } }, datalabels: { font: { size: fontConfig.datalabel } } }
        }
      });
    }
    function renderProjectionChart() {
      const years = parseInt(document.getElementById('projectionYears').value) || 20;
      const labels = Array.from({ length: years+1 }, (_, t) => t+'a');
      const data = labels.map((_, t) => investments.reduce((s, inv) => s + inv.amount * Math.pow(1+inv.return/100, t), 0));
      const lastVal = data[data.length-1];
      const ctx = document.getElementById('projectionChart').getContext('2d');
      if (projChart) projChart.destroy();
      projChart = new Chart(ctx, {
        type: 'line', data: { labels, datasets: [{ label: 'Capital projeté', data }] },
        options: {
          maintainAspectRatio: false,
          plugins: { legend: { labels: { font: { size: fontConfig.legend } } }, title: { display: true, text: `Capital après ${years} ans : ${lastVal.toLocaleString('fr-FR',{minimumFractionDigits:2})} €`, font: { size: fontConfig.title } } },
          scales: { x: { ticks: { font: { size: fontConfig.axis } } }, y: { beginAtZero: true, ticks: { font: { size: fontConfig.axis } } } }
        }
      });
    }
  </script>
</body>
</html>
