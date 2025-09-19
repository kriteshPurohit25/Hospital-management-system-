<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Hospital Database — Clean Storage</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    body {
      font-family: Arial, Helvetica, sans-serif;
      background: linear-gradient(135deg,#0b0c10,#1f2833);
      color: #fff;
      padding: 20px;
      line-height: 1.4;
    }
    h1 { color: #66fcf1; margin: 0 0 8px 0; }
    .form-row { margin: 6px 0; display:flex; gap:8px; flex-wrap:wrap; }
    input, select, button {
      padding:8px;
      border-radius:6px;
      border:none;
      min-width:120px;
    }
    input[type="number"] { max-width:140px; }
    button { background:#45a29e; color:#000; cursor:pointer; }
    button.secondary { background:#222; color:#66fcf1; border:1px solid #45a29e; }
    button.danger { background:#ff6b6b; color:#fff; }
    table { width:100%; border-collapse:collapse; margin-top:14px; font-size:14px; }
    th, td { padding:8px; border:1px solid #45a29e; text-align:center; }
    th { background:#1f2833; color:#66fcf1; }
    td { background:#0b0c10; }
    .small { font-size:13px; color:#c7f9f2; }
    .controls { margin-top:10px; display:flex; gap:8px; flex-wrap:wrap; }
    .notice { margin-top:8px; color:#ffd; }
  </style>
</head>
<body>
  <h1>Hospital Management Database</h1>

  <div>
    <h3 class="small">Add New Hospital</h3>
    <div class="form-row">
      <input id="name" type="text" placeholder="Hospital Name">
      <input id="type" type="text" placeholder="Type (General, Specialty, etc.)">
      <input id="beds" type="number" placeholder="Beds">
      <input id="doctors" type="number" placeholder="Doctors">
      <input id="location" type="text" placeholder="Location">
      <input id="year" type="number" placeholder="Year Established">
      <input id="patients" type="number" placeholder="Patients Treated" min="0">
      <input id="departments" type="number" placeholder="Departments" min="0">
    </div>

    <div class="controls">
      <button onclick="addHospital()">Add Hospital</button>
      <button class="secondary" onclick="cleanInvalidEntries()">Clean invalid entries</button>
      <button class="danger" onclick="resetDatabase()">Reset Database</button>
    </div>

    <div class="notice small">
      Tip: The app will automatically remove records that are missing required fields or contain invalid numbers.
    </div>
  </div>

  <h3 style="margin-top:18px">Stored Hospitals</h3>
  <table id="hospitalTable" aria-live="polite">
    <thead>
      <tr>
        <th>Name</th><th>Type</th><th>Beds</th><th>Doctors</th><th>Location</th>
        <th>Year</th><th>Patients Treated</th><th>Departments</th><th>Last Update</th>
      </tr>
    </thead>
    <tbody id="hospitalBody">
      <!-- rows inserted here -->
    </tbody>
  </table>

  <script>
    // ---------- Utility: clean stored data ----------
    function cleanStoredDataOnLoad() {
      const raw = localStorage.getItem("hospitals");
      if (!raw) {
        localStorage.setItem("hospitals", JSON.stringify([]));
        return;
      }

      try {
        const parsed = JSON.parse(raw);
        if (!Array.isArray(parsed)) {
          localStorage.setItem("hospitals", JSON.stringify([]));
          return;
        }

        const cleaned = parsed.filter(item => {
          if (!item || typeof item !== 'object') return false;

          // required string fields
          const requiredStrings = ['name','type','location','lastUpdate'];
          for (const key of requiredStrings) {
            if (!(key in item)) return false;
            if (typeof item[key] !== 'string') return false;
            if (item[key].trim() === '') return false;
          }

          // required numeric fields
          const requiredNums = ['beds','doctors','year','patients','departments'];
          for (const key of requiredNums) {
            if (!(key in item)) return false;
            const n = Number(item[key]);
            if (!isFinite(n)) return false;
          }

          const date = new Date(item.lastUpdate);
          if (isNaN(date.getTime())) return false;

          return true;
        });

        if (cleaned.length !== parsed.length) {
          console.info("Cleaned invalid entries (removed " + (parsed.length - cleaned.length) + " item(s)).");
        }
        localStorage.setItem("hospitals", JSON.stringify(cleaned));
      } catch (e) {
        console.warn("localStorage parse error; resetting hospitals to empty array.", e);
        localStorage.setItem("hospitals", JSON.stringify([]));
      }
    }

    // ---------- Public manual clean ----------
    function cleanInvalidEntries() {
      if (!confirm("This will scan and remove invalid stored records. Continue?")) return;
      cleanStoredDataOnLoad();
      showHospitals();
      alert("Invalid entries removed (if any).");
    }

    // ---------- Reset DB ----------
    function resetDatabase() {
      if (!confirm("⚠️ Are you sure you want to clear ALL hospital data?")) return;
      localStorage.setItem("hospitals", JSON.stringify([]));
      showHospitals();
    }

    // ---------- Add hospital ----------
    function addHospital() {
      const name = document.getElementById("name").value.trim();
      const type = document.getElementById("type").value.trim();
      const beds = Number(document.getElementById("beds").value);
      const doctors = Number(document.getElementById("doctors").value);
      const location = document.getElementById("location").value.trim();
      const year = Number(document.getElementById("year").value);
      const patients = Number(document.getElementById("patients").value);
      const departments = Number(document.getElementById("departments").value);

      if (!name || !type || !location) {
        alert("Please fill in all text fields (name, type, location).");
        return;
      }
      if (![beds, doctors, year, patients, departments].every(n => Number.isFinite(n))) {
        alert("Please fill numeric fields with valid numbers.");
        return;
      }
      if ([beds, doctors, year, patients, departments].some(n => n < 0)) {
        alert("Numbers must be 0 or greater.");
        return;
      }

      const today = new Date().toISOString().split("T")[0];
      const record = {
        name,
        type,
        beds: Number(beds),
        doctors: Number(doctors),
        location,
        year: Number(year),
        patients: Number(patients),
        departments: Number(departments),
        lastUpdate: today
      };

      const list = JSON.parse(localStorage.getItem("hospitals")) || [];
      list.push(record);
      localStorage.setItem("hospitals", JSON.stringify(list));
      clearForm();
      showHospitals();
    }

    function clearForm() {
      ["name","type","beds","doctors","location","year","patients","departments"]
      .forEach(id => { const el = document.getElementById(id); if (el) el.value = ""; });
    }

    // ---------- Render table ----------
    function showHospitals() {
      const tbody = document.getElementById("hospitalBody");
      const hospitals = JSON.parse(localStorage.getItem("hospitals")) || [];
      tbody.innerHTML = "";

      if (hospitals.length === 0) {
        tbody.innerHTML = `<tr><td colspan="9" class="small">No records yet — add hospitals using the form above.</td></tr>`;
        return;
      }

      hospitals.forEach(h => {
        const row = document.createElement("tr");
        row.innerHTML = `
          <td>${escapeHtml(String(h.name))}</td>
          <td>${escapeHtml(String(h.type))}</td>
          <td>${escapeHtml(String(h.beds))}</td>
          <td>${escapeHtml(String(h.doctors))}</td>
          <td>${escapeHtml(String(h.location))}</td>
          <td>${escapeHtml(String(h.year))}</td>
          <td>${escapeHtml(String(h.patients))}</td>
          <td>${escapeHtml(String(h.departments))}</td>
          <td>${escapeHtml(String(h.lastUpdate))}</td>
        `;
        tbody.appendChild(row);
      });
    }

    function escapeHtml(s) {
      return s.replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
    }

    cleanStoredDataOnLoad();
    showHospitals();

    window._cleanStoredDataOnLoad = cleanStoredDataOnLoad;
    window._showHospitals = showHospitals;
    window._dump = () => JSON.parse(localStorage.getItem("hospitals") || "[]");
  </script>
</body>
</html>
