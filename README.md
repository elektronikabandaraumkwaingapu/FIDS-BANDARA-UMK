<!DOCTYPE html>
<html>
<head>
  <style>
    :root { --primary-dark: #002366; --accent-orange: #f39c12; --glass: rgba(255, 255, 255, 0.1); }
    * { box-sizing: border-box; }
    body {
      font-family: 'Arial', sans-serif;
      background: linear-gradient(135deg, #001540 0%, #004e92 100%);
      color: white; margin: 0; min-height: 100vh; display: flex; flex-direction: column; overflow-x: hidden;
    }
    .header {
      display: flex; flex-wrap: wrap; justify-content: space-between; align-items: center;
      padding: 15px 20px; background: rgba(0,0,0,0.5); border-bottom: 3px solid var(--accent-orange); gap: 10px;
    }
    .airport-title { text-align: center; flex: 1; min-width: 250px; }
    .airport-title h1 { margin: 0; font-size: clamp(1.2em, 4vw, 1.8em); }
    .time { font-size: clamp(1.2em, 5vw, 2em); font-weight: bold; color: #00ff00; font-family: monospace; }
    .table-container { padding: 15px; flex-grow: 1; overflow-x: auto; }
    table { width: 100%; border-collapse: collapse; min-width: 700px; }
    th { padding: 12px; background: var(--primary-dark); color: var(--accent-orange); border: 1px solid rgba(255,255,255,0.1); text-transform: uppercase; font-size: 0.8em; }
    td { padding: 12px; text-align: center; background: var(--glass); border-bottom: 1px solid rgba(255,255,255,0.1); font-weight: bold; }
    .status-box { padding: 5px 10px; border-radius: 5px; min-width: 120px; display: inline-block; font-size: 0.8em; }
    .status-green { background: #27ae60; } .status-red { background: #c0392b; } .status-blue { background: #2980b9; }
    .marquee-footer { background: var(--accent-orange); color: black; padding: 8px 0; font-weight: bold; }
  </style>
</head>
<body>

<div class="header">
  <div style="text-align:center">
    <div style="font-size:0.7em">WITA</div>
    <div id="clock" class="time">00:00:00</div>
  </div>
  <div class="airport-title">
    <h1>UMBU MEHANG KUNDA</h1>
    <p style="margin:5px 0 0; color:var(--accent-orange); font-size:0.8em">FLIGHT INFORMATION DISPLAY SYSTEM</p>
  </div>
  <div style="text-align:center">
    <div style="font-size:0.7em; color:#3498db">UTC</div>
    <div id="utc-clock" class="time" style="color:#3498db">00:00:00</div>
  </div>
</div>

<div class="table-container">
  <table>
    <thead>
      <tr>
        <th><span class="lang-text" data-id="penerbangan">PENERBANGAN</span></th>
        <th><span class="lang-text" data-id="asal">ASAL</span></th>
        <th><span class="lang-text" data-id="tujuan">TUJUAN</span></th>
        <th><span class="lang-text" data-id="jadwal">JADWAL</span></th>
        <th><span class="lang-text" data-id="estimasi">ESTIMASI</span></th>
        <th><span class="lang-text" data-id="keterangan">KETERANGAN</span></th>
      </tr>
    </thead>
    <tbody id="flight-data">
      <tr><td colspan="6">Memuat data... (Loading data...)</td></tr>
    </tbody>
  </table>
</div>

<div class="marquee-footer">
  <marquee>Selamat Datang di Bandar Udara Umbu Mehang Kunda, Waingapu. Periksa kembali barang bawaan Anda. | Welcome to Umbu Mehang Kunda Airport.</marquee>
</div>

<script>
  // 1. Logika Jam
  function updateTime() {
    const now = new Date();
    document.getElementById('clock').innerText = now.toLocaleTimeString('id-ID', {hour12: false});
    const utcTime = new Date(now.getTime() + (now.getTimezoneOffset() * 60000));
    document.getElementById('utc-clock').innerText = utcTime.toLocaleTimeString('id-ID', {hour12: false});
  }
  setInterval(updateTime, 1000);

  // 2. Logika Bahasa
  let isEnglish = false;
  const langData = {
    penerbangan: ["PENERBANGAN", "FLIGHT"],
    asal: ["ASAL", "ORIGIN"],
    tujuan: ["TUJUAN", "DESTINATION"],
    jadwal: ["JADWAL", "SCHEDULE"],
    estimasi: ["ESTIMASI", "ESTIMATE"],
    keterangan: ["KETERANGAN", "REMARKS"]
  };

  setInterval(() => {
    isEnglish = !isEnglish;
    document.querySelectorAll('.lang-text').forEach(el => {
      const id = el.getAttribute('data-id');
      el.innerText = isEnglish ? langData[id][1] : langData[id][0];
    });
  }, 5000);

  // 3. Ambil Data (Koreksi Utama)
  function startFetch() {
    google.script.run
      .withSuccessHandler(renderTable)
      .withFailureHandler(err => {
         document.getElementById('flight-data').innerHTML = `<tr><td colspan="6" style="color:red">Error: ${err.message}</td></tr>`;
      })
      .getFlightData();
  }

  function renderTable(data) {
    const tableBody = document.getElementById('flight-data');
    if (!data || data.length === 0) {
      tableBody.innerHTML = '<tr><td colspan="6">Tidak ada jadwal penerbangan.</td></tr>';
      return;
    }

    tableBody.innerHTML = data.map(row => {
      const statusText = (row[5] || "").toUpperCase();
      let statusColor = "status-blue";
      if(/LANDED|MENDARAT|CHECK-IN|TAKE-OFF|BERANGKAT|BOARDING|ON SCHEDULE|TERJADWAL/.test(statusText)) statusColor = "status-green";
      else if(/DELAYED|TERLAMBAT|RETURN|KEMBALI|CANCELED|DIBATALKAN|RESCHEDULE/.test(statusText)) statusColor = "status-red";

      return `<tr>
        <td><span style="background:white; color:black; padding:3px 8px; border-radius:3px">${row[0]}</span></td>
        <td>${row[1]}</td><td>${row[2]}</td><td>${row[3]}</td><td>${row[4]}</td>
        <td><span class="status-box ${statusColor}">${row[5]}</span></td>
      </tr>`;
    }).join('');
  }

  window.onload = () => {
    updateTime();
    startFetch();
    setInterval(startFetch, 20000); // Refresh data tiap 20 detik
  };
</script>
</body>
</html>
