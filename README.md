<!DOCTYPE html>
<html>
<head>
  <title>Generate Qurbani Bill</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/qrious@4.0.2/dist/qrious.min.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      padding: 20px;
      background: #f8f8f8;
    }

    #invoice {
      width: 700px;
      margin: auto;
      padding: 20px;
      background: rgb(254, 251, 251);
      border: 2px solid #000;
    }

    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 15px;
    }

    thead {
      background-color: #c0392b;
      color: white;
    }

    tbody tr {
      background-color: white;
      color: black;
    }

    td, th {
      border: 1px solid black;
      padding: 5px;
    }

    #Success {
      color: green;
      font-weight: bold;
    }

    .blue-text {
      color: #ea130c;
    }
  </style>
</head>
<body>

<form id="submit-to-google-sheet">
  <h3>Customer Info</h3>
  <input type="text" name="name" placeholder="Name" required>
  <input type="text" name="address" placeholder="Address" required>
  <input type="text" name="mobile" placeholder="Mobile" required>

  <h3>Qurbani Share Details</h3>
  <div id="bill-entries">
    <div class="entry">
      <input type="text" name="itemName[]" placeholder="Name">
      <input type="number" name="unitPrice[]" placeholder="Unit Price">
      <input type="number" name="amount[]" placeholder="Amount">
    </div>
  </div>
  <button type="button" onclick="addEntry()">+ Add Entry</button>

  <h3>Mode of Payment</h3>
  <label><input type="checkbox" name="paymentMode" value="Cash"> Cash</label>
  <label><input type="checkbox" name="paymentMode" value="UPI/Card"> UPI/Card</label>

  <h3>Do you want to donate meat?</h3>
  <label><input type="radio" name="donation" value="Yes"> Yes</label>
  <label><input type="radio" name="donation" value="No"> No</label>

  <br><br>
  <input type="hidden" name="invoiceNumber" id="invoiceNumber" value="">
  <button type="submit">Submit & Generate Bill</button>
</form>

<div id="Success" style="display: none;">Submitted!</div>

<!-- Invoice Display -->
<div id="invoice" style="display: none;">
  <div style="display: flex; justify-content: space-between;">
    <div><h2 class="blue-text">NAIMI QURBANI CENTRE</h2></div>
    <div class="blue-text" style="text-align:right;">
      <strong>JAMIA NAIMIA</strong><br>
      Syed Mahfooz Alam Naimi<br>
      Deewan Bazar, Moradabad UP-244001<br>
      Phone No: +91 9412475657<br>
      mnaimi550@gmail.com
    </div>
  </div>

  <div style="display:flex; justify-content:space-between;">
    <div class="blue-text">
      <p><strong>Name:</strong> <span id="inv-name"></span></p>
      <p><strong>Address:</strong> <span id="inv-address"></span></p>
      <p><strong>Mobile:</strong> <span id="inv-mobile"></span></p>
    </div>
    <div>
      <p><strong>Date of Booking:</strong> __/__/____</p>
      <p><strong>Date of Delivery:</strong> __/__/____</p>
      <p><strong>Animal No:</strong> __/__</p>
      <canvas id="qrCode" width="120" height="120"></canvas>
    </div>
  </div>

  <p><strong>Payment Selected:</strong> <span id="inv-payment" class="blue-text"></span></p>

  <table>
    <thead><tr><th>Name</th><th>Unit Price</th><th>Amount</th></tr></thead>
    <tbody id="invoice-table-body"></tbody>
  </table>

  <p><strong>Total:</strong> <span id="total">0/-</span></p>

  <p><strong>Do you want to donate meat?:</strong> <span id="inv-donation"></span></p>

  <div style="display: flex; justify-content: flex-end; margin-top: 20px;">
    <div>
      <p><strong>Advance Payment:</strong> ________</p>
      <p><strong>Payment Left:</strong> ________</p>
      <p><strong>Signature:</strong></p>
      <p><strong>Syed Mahfooz Alam Naimi</strong></p>
      <div style="width: 150px; height: 50px; border: 1px solid black;"></div>
    </div>
  </div>

  <p style="text-align: right;"><strong>Date & Time:</strong> <span id="date-time"></span></p>
  <p style="text-align: right; font-weight:bold;">Invoice Number: <span id="display-invoice-number"></span></p>
  <p style="text-align: center; color: #b30000;"><strong>Eidul Adha Mubarak in Advance</strong></p>
</div>

<button id="downloadBtn" style="display:none;" onclick="generatePDF()">Download PDF</button>

<script>
  const scriptURL = 'https://script.google.com/macros/s/AKfycbxKgWmI8gnalklORMJdzHvnCw3Pv2DPnF0P9OmH0YhoMXL3KAdAdkqU8EFu08bTMZ9CSA/exec';
  const form = document.getElementById('submit-to-google-sheet');
  const successMessage = document.getElementById('Success');
  const downloadBtn = document.getElementById('downloadBtn');

  function getInvoiceNumber() {
    let last = parseInt(localStorage.getItem("invoiceNumber")) || 0;
    last += 1;
    localStorage.setItem("invoiceNumber", last);
    return last;
  }

  form.addEventListener('submit', async e => {
    e.preventDefault();

    const invoiceNum = getInvoiceNumber();
    document.getElementById('invoiceNumber').value = invoiceNum;
    document.getElementById('display-invoice-number').innerText = invoiceNum;

    const formData = new FormData(form);
    const modes = formData.getAll('paymentMode');
    const donation = formData.get('donation') || 'No';
    formData.append('invoiceNumber', invoiceNum);

    populateInvoice(formData);

    const names = formData.getAll('itemName[]');
    const unitPrices = formData.getAll('unitPrice[]');
    const amounts = formData.getAll('amount[]');

    // Submit Customer Info
    const customerData = new URLSearchParams();
    customerData.append('Invoice No', invoiceNum);
    customerData.append('Name', formData.get('name'));
    customerData.append('Address', formData.get('address'));
    customerData.append('Mobile', formData.get('mobile'));
    customerData.append('Payment Mode', modes.join(', '));
    customerData.append('Donation', donation);

    try {
      await fetch(scriptURL + "?sheet=CustomerInfo", {
        method: 'POST',
        body: customerData,
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      });

      // Submit Qurbani Entries
      for (let i = 0; i < names.length; i++) {
        const entry = new URLSearchParams();
        entry.append('Invoice No', invoiceNum);
        entry.append('Item Name', names[i]);
        entry.append('Unit Price', unitPrices[i]);
        entry.append('Amount', amounts[i]);

        await fetch(scriptURL + "?sheet=QurbaniDetails", {
          method: 'POST',
          body: entry,
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
        });
      }

      successMessage.style.display = "block";
      successMessage.textContent = "Submitted!";
      document.getElementById('invoice').style.display = 'block';
      downloadBtn.style.display = 'inline-block';
    } catch (err) {
      console.error("Error:", err);
      successMessage.style.display = "block";
      successMessage.textContent = "Error: " + err.message;
    }
  });

  function addEntry() {
    const div = document.createElement('div');
    div.className = 'entry';
    div.innerHTML = `
      <input type="text" name="itemName[]" placeholder="Name">
      <input type="number" name="unitPrice[]" placeholder="Unit Price">
      <input type="number" name="amount[]" placeholder="Amount">
    `;
    document.getElementById('bill-entries').appendChild(div);
  }

  function populateInvoice(formData) {
    document.getElementById('inv-name').innerText = formData.get('name');
    document.getElementById('inv-address').innerText = formData.get('address');
    document.getElementById('inv-mobile').innerText = formData.get('mobile');
    document.getElementById('inv-payment').innerText = formData.getAll('paymentMode').join(', ');
    document.getElementById('inv-donation').innerText = formData.get('donation') || 'No';

    const names = formData.getAll('itemName[]');
    const unitPrices = formData.getAll('unitPrice[]');
    const amounts = formData.getAll('amount[]');
    const tbody = document.getElementById('invoice-table-body');
    tbody.innerHTML = '';
    let total = 0;

    for (let i = 0; i < names.length; i++) {
      const up = parseFloat(unitPrices[i]) || 0;
      const amt = parseFloat(amounts[i]) || 0;
      total += amt;
      tbody.innerHTML += `<tr><td>${names[i]}</td><td>${up}/-</td><td>${amt}/-</td></tr>`;
    }

    document.getElementById('total').innerText = `${total}/-`;

    new QRious({
      element: document.getElementById("qrCode"),
      value: 'upi://pay?pa=9149054331@ptsbi&pn=SYED%20MAHFOOZ%20ALAM',
      size: 150
    });

    const now = new Date();
    document.getElementById('date-time').innerText = now.toLocaleString();
  }

  function generatePDF() {
    const element = document.getElementById('invoice');
    const opt = {
      margin: 0,
      filename: 'qurbani-bill.pdf',
      image: { type: 'jpeg', quality: 1 },
      html2canvas: { scale: 3, useCORS: true },
      jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' }
    };
    html2pdf().set(opt).from(element).save();
  }
</script>

</body>
</html>
