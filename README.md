<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sistem Input Data Penjualan</title>
    <!-- Bootstrap CSS untuk tampilan profesional -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <!-- QuaggaJS untuk scan barcode -->
    <script src="https://cdn.jsdelivr.net/npm/quagga@0.12.1/dist/quagga.min.js"></script>
    <style>
        body { background-color: #f8f9fa; }
        .header { background-color: #007bff; color: white; padding: 20px; text-align: center; }
        .card { box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
        .btn-primary { background-color: #007bff; border-color: #007bff; }
        .btn-success { background-color: #28a745; border-color: #28a745; }
        .btn-warning { background-color: #ffc107; border-color: #ffc107; }
        .btn-danger { background-color: #dc3545; border-color: #dc3545; }
        .btn-secondary { background-color: #6c757d; border-color: #6c757d; }
        table { margin-top: 20px; }
        #scanner-container { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.8); z-index: 1000; }
        #scanner-container video { width: 100%; height: 100%; object-fit: cover; }
        #scanner-container canvas { display: none; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Sistem Input Data Penjualan</h1>
        <p>Kelola data penjualan Anda dengan mudah dan ekspor ke Excel.</p>
    </div>

    <div class="container mt-4">
        <div class="row">
            <div class="col-md-6">
                <div class="card">
                    <div class="card-header">
                        <h5>Form Input Data Penjualan</h5>
                    </div>
                    <div class="card-body">
                        <form id="salesForm">
                            <div class="mb-3">
                                <label for="date" class="form-label">Tanggal</label>
                                <input type="date" class="form-control" id="date" required>
                            </div>
                            <div class="mb-3">
                                <label for="product" class="form-label">Nama Produk</label>
                                <input type="text" class="form-control" id="product" required>
                            </div>
                            <div class="mb-3">
                                <label for="quantity" class="form-label">Jumlah</label>
                                <input type="number" class="form-control" id="quantity" required>
                            </div>
                            <div class="mb-3">
                                <label for="price" class="form-label">Harga per Unit (Rp)</label>
                                <input type="number" class="form-control" id="price" step="0.01" required>
                            </div>
                            <button type="submit" class="btn btn-primary" id="submitBtn">Tambah Data</button>
                            <button type="button" class="btn btn-secondary ms-2" id="scanBtn">Scan Barcode</button>
                        </form>
                    </div>
                </div>
            </div>
            <div class="col-md-6">
                <div class="card">
                    <div class="card-header">
                        <h5>Data Penjualan</h5>
                    </div>
                    <div class="card-body">
                        <table class="table table-striped" id="salesTable">
                            <thead>
                                <tr>
                                    <th>Tanggal</th>
                                    <th>Nama Produk</th>
                                    <th>Jumlah</th>
                                    <th>Harga per Unit</th>
                                    <th>Total</th>
                                    <th>Aksi</th>
                                </tr>
                            </thead>
                            <tbody></tbody>
                        </table>
                        <button id="exportBtn" class="btn btn-success">Ekspor ke CSV (untuk Excel)</button>
                        <button id="clearAllBtn" class="btn btn-danger ms-2">Hapus Semua Data</button>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- Container untuk scanner -->
    <div id="scanner-container">
        <video id="scanner-video"></video>
        <canvas id="scanner-canvas"></canvas>
        <button id="closeScanner" class="btn btn-danger" style="position: absolute; top: 10px; right: 10px;">Tutup</button>
    </div>

    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script>
        const form = document.getElementById('salesForm');
        const tableBody = document.querySelector('#salesTable tbody');
        const exportBtn = document.getElementById('exportBtn');
        const clearAllBtn = document.getElementById('clearAllBtn');
        const scanBtn = document.getElementById('scanBtn');
        const scannerContainer = document.getElementById('scanner-container');
        const closeScanner = document.getElementById('closeScanner');
        const submitBtn = document.getElementById('submitBtn');
        let salesData = [];
        let editIndex = -1;
        const PASSWORD = "pass123"; // Password untuk edit dan hapus.

        form.addEventListener('submit', function(e) {
            e.preventDefault();
            
            const date = document.getElementById('date').value;
            const product = document.getElementById('product').value;
            const quantity = parseInt(document.getElementById('quantity').value);
            const price = parseFloat(document.getElementById('price').value);
            const total = quantity * price;
            
            if (editIndex === -1) {
                // Tambah data baru
                const row = { date, product, quantity, price, total };
                salesData.push(row);
                addRowToTable(row, salesData.length - 1);
            } else {
                // Update data
                salesData[editIndex] = { date, product, quantity, price, total };
                updateTable();
                editIndex = -1;
                submitBtn.textContent = 'Tambah Data';
            }
            
            // Reset form
            form.reset();
        });

        function addRowToTable(row, index) {
            const tr = document.createElement('tr');
            tr.innerHTML = `
                <td>${row.date}</td>
                <td>${row.product}</td>
                <td>${row.quantity}</td>
                <td>Rp ${row.price.toLocaleString()}</td>
                <td>Rp ${row.total.toLocaleString()}</td>
                <td>
                    <button class="btn btn-warning btn-sm edit-btn" data-index="${index}">Edit</button>
                    <button class="btn btn-danger btn-sm delete-btn" data-index="${index}">Hapus</button>
                </td>
            `;
            tableBody.appendChild(tr);
        }

        function updateTable() {
            tableBody.innerHTML = '';
            salesData.forEach((row, index) => addRowToTable(row, index));
        }

        tableBody.addEventListener('click', function(e) {
            if (e.target.classList.contains('edit-btn')) {
                const index = parseInt(e.target.getAttribute('data-index'));
                const enteredPassword = prompt("Masukkan password untuk edit:");
                if (enteredPassword === PASSWORD) {
                    const row = salesData[index];
                    document.getElementById('date').value = row.date;
                    document.getElementById('product').value = row.product;
                    document.getElementById('quantity').value = row.quantity;
                    document.getElementById('price').value = row.price;
                    editIndex = index;
                    submitBtn.textContent = 'Update Data';
                } else {
                    alert("Password salah. Edit dibatalkan.");
                }
            } else if (e.target.classList.contains('delete-btn')) {
                const index = parseInt(e.target.getAttribute('data-index'));
                const enteredPassword = prompt("Masukkan password untuk hapus:");
                if (enteredPassword === PASSWORD) {
                    if (confirm('Apakah Anda yakin ingin menghapus data ini?')) {
                        salesData.splice(index, 1);
                        updateTable();
                    }
                } else {
                    alert("Password salah. Hapus dibatalkan.");
                }
            }
        });

        exportBtn.addEventListener('click', function() {
            if (salesData.length === 0) {
                alert('Tidak ada data untuk diekspor.');
                return;
            }
            
            // Buat CSV
            let csv = 'Tanggal,Nama Produk,Jumlah,Harga per Unit,Total\n';
            salesData.forEach(row => {
                csv += `${row.date},${row.product},${row.quantity},${row.price},${row.total}\n`;
            });
            
            // Download CSV
            const blob = new Blob([csv], { type: 'text/csv' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'data_penjualan.csv';
            a.click();
            URL.revokeObjectURL(url);
        });

        clearAllBtn.addEventListener('click', function() {
            const enteredPassword = prompt("Masukkan password untuk hapus semua data:");
            if (enteredPassword === PASSWORD) {
                if (confirm('Apakah Anda yakin ingin menghapus semua data? Tindakan ini tidak dapat dibatalkan.')) {
                    salesData = [];
                    updateTable();
                }
            } else {
                alert("Password salah. Hapus semua data dibatalkan.");
            }
        });

        // Fungsi untuk scan barcode
        scanBtn.addEventListener('click', function() {
            scannerContainer.style.display = 'block';
            Quagga.init({
                inputStream: {
                    name: "Live",
                    type: "LiveStream",
                    target: document.querySelector('#scanner-video'),
                    constraints: {
                        width: 640,
                        height: 480,
                        facingMode: "environment" // Gunakan kamera belakang jika tersedia
                    }
                },
                locator: {
                    patchSize: "medium",
                    halfSample: true
                },
                numOfWorkers: 2,
                decoder: {
                    readers: ["code_128_reader", "ean_reader", "ean_8_reader", "code_39_reader", "upc_reader", "upc_e_reader"]
                },
                locate: true
            }, function(err) {
                if (err) {
                    console.log(err);
                    alert("Gagal menginisialisasi scanner. Pastikan kamera diizinkan.");
                    return;
                }
                Quagga.start();
            });

            Quagga.onDetected(function(result) {
                const code = result.codeResult.code;
                document.getElementById('product').value = code; // Isi field produk dengan kode barcode
                Quagga.stop();
                scannerContainer.style.display = 'none';
                alert(`Barcode terdeteksi: ${code}. Field produk telah diisi.`);
            });
        });

        closeScanner.addEventListener('click', function() {
            Quagga.stop();
            scannerContainer.style.display = 'none';
        });
    </script>
</body>
</html>
