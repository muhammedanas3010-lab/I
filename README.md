const express = require('express');
const mongoose = require('mongoose');

const app = express();
app.use(express.json());

// 1. Database Connection
mongoose.connect('mongodb://127.0.0.1:27017/full_store_db')
  .then(() => console.log('MongoDB Connected Successfully'))
  .catch(err => console.error('DB Connection Error:', err));

// 2. Schemas & Models

// Employee Schema (Password illathe add cheyyaan)
const employeeSchema = new mongoose.Schema({
  name: { type: String, required: true },
  role: { type: String, required: true },
  addedAt: { type: Date, default: Date.now }
});

// Item / Order Schema
const itemSchema = new mongoose.Schema({
  itemName: { type: String, required: true },
  costPrice: { type: Number, required: true },    // Material Cost
  sellingPrice: { type: Number, required: true },
  paidAmount: { type: Number, default: 0 },       // Advance / Received
  paymentStatus: { 
    type: String, 
    enum: ['PENDING', 'PARTIAL', 'PAID'], 
    default: 'PENDING' 
  },
  isProfitAdded: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now }    // Date-wise Analysis-nu vendi
});

// Store Total Profit Schema
const storeSchema = new mongoose.Schema({
  storeName: { type: String, default: "Main Store" },
  totalProfit: { type: Number, default: 0 }
});

const Employee = mongoose.model('Employee', employeeSchema);
const Item = mongoose.model('Item', itemSchema);
const Store = mongoose.model('Store', storeSchema);

// 3. API Endpoints

// Employee Endpoints
app.post('/api/employees', async (req, res) => {
  try {
    const { name, role } = req.body;
    const newEmployee = new Employee({ name, role });
    await newEmployee.save();
    return res.status(201).json(newEmployee);
  } catch (err) {
    return res.status(500).json({ error: err.message });
  }
});

app.get('/api/employees', async (req, res) => {
  const employees = await Employee.find();
  return res.json(employees);
});

// Item Add Endpoint (Each Order Service Charge +300 Added to Selling Price)
app.post('/api/items', async (req, res) => {
  try {
    const { itemName, costPrice, sellingPrice } = req.body;
    
    // Oro order-il ninnum +300 Selling Price-ileeku add cheyyunnu
    const adjustedSellingPrice = Number(sellingPrice) + 300;

    const newItem = new Item({
      itemName,
      costPrice: Number(costPrice),
      sellingPrice: adjustedSellingPrice
    });

    await newItem.save();
    return res.status(201).json({ message: 'Item Added Successfully!', item: newItem });
  } catch (error) {
    return res.status(500).json({ error: error.message });
  }
});

// Payment & Automatic Profit Calculation
app.post('/api/pay', async (req, res) => {
  try {
    const { itemId, amountPaidNow } = req.body;
    const item = await Item.findById(itemId);
    if (!item) return res.status(404).json({ message: 'Item Not Found' });

    item.paidAmount += Number(amountPaidNow);

    if (item.paidAmount >= item.sellingPrice) {
      item.paymentStatus = 'PAID';

      if (!item.isProfitAdded) {
        // Net Profit = Selling Price (Includes +300) - Cost Price
        const netProfit = item.sellingPrice - item.costPrice;
        await Store.findOneAndUpdate(
          { storeName: 'Main Store' },
          { $inc: { totalProfit: netProfit } },
          { upsert: true, new: true }
        );
        item.isProfitAdded = true;
      }
    } else {
      item.paymentStatus = 'PARTIAL';
    }

    await item.save();
    return res.status(200).json({ message: 'Payment Updated!', item });
  } catch (error) {
    return res.status(500).json({ error: error.message });
  }
});

// Date-wise Analysis API
app.get('/api/analysis', async (req, res) => {
  try {
    const { startDate, endDate } = req.query;
    let query = {};

    if (startDate && endDate) {
      query.createdAt = {
        $gte: new Date(startDate),$lte: new Date(new Date(endDate).setHours(23, 59, 59))
      };
    }

    const items = await Item.find(query);
    let totalMaterialCost = 0;
    let totalAdvance = 0;
    let totalProfitGenerated = 0;

    items.forEach(item => {
      totalMaterialCost += item.costPrice;
      totalAdvance += item.paidAmount;
      if (item.paymentStatus === 'PAID') {
        totalProfitGenerated += (item.sellingPrice - item.costPrice);
      }
    });

    return res.json({
      totalOrders: items.length,
      totalMaterialCost,
      totalAdvance,
      advanceRemaining: totalMaterialCost - totalAdvance,
      totalProfitGenerated
    });
  } catch (err) {
    return res.status(500).json({ error: err.message });
  }
});

app.get('/api/items', async (req, res) => {
  const items = await Item.find().sort({ createdAt: -1 });
  return res.json(items);
});

app.get('/api/store', async (req, res) => {
  const store = await Store.findOne({ storeName: 'Main Store' });
  return res.json({ totalProfit: store ? store.totalProfit : 0 });
});

// 4. Frontend UI Dashboard
app.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>Store Management Dashboard</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 20px; background: #f4f6f9; }
        .card { background: #fff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); margin-bottom: 20px; }
        .flex { display: flex; gap: 20px; flex-wrap: wrap; }
        .profit-box { background: #27ae60; color: white; padding: 15px; border-radius: 8px; font-size: 20px; font-weight: bold; }
        input, button, select { padding: 10px; margin: 5px 0; font-size: 14px; }
        button { background: #3498db; color: white; border: none; cursor: pointer; border-radius: 4px; }
        button:hover { background: #2980b9; }
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { border: 1px solid #ddd; padding: 10px; text-align: left; }
        th { background-color: #f2f2f2; }
        .hint { font-size: 12px; color: #e74c3c; font-weight: bold; margin-top: 4px; }
        .badge { padding: 4px 8px; border-radius: 4px; color: #fff; font-size: 12px; }
        .PAID { background-color: #2ecc71; }
        .PARTIAL { background-color: #f39c12; }
        .PENDING { background-color: #e74c3c; }
      </style>
    </head>
    <body>

      <h2>Store & Employee Dashboard</h2>

      <div class="flex">
        <div class="card" style="flex: 1;">
          <h3>Total Store Profit</h3>
          <div class="profit-box">Total Profit: ₹<span id="totalProfit">0</span></div>
        </div>

        <!-- Employee Add Section -->
        <div class="card" style="flex: 1;">
          <h3>New Employee Add Cheyyuka</h3>
          <form id="empForm">
            <input type="text" id="empName" placeholder="Employee Name" required>
            <input type="text" id="empRole" placeholder="Role / Post" required>
            <button type="submit">Add Employee</button>
          </form>
          <h4>Employees List:</h4>
          <ul id="empList"></ul>
        </div>
      </div>

      <!-- Date-wise Analysis Section -->
      <div class="card">
        <h3>Date-based Analysis</h3>
        <input type="date" id="startDate">
        <input type="date" id="endDate">
        <button onclick="runAnalysis()">Filter Analysis</button>
        
        <div id="analysisResult" style="margin-top: 15px;"></div>
      </div>

      <!-- Add Item Section -->
      <div class="card">
        <h3>Puthiya Order / Item Store Cheyyuka</h3>
        <p style="font-size: 12px; color: #555;">* Auto Charge: Oro order-il ninnum auto aayi ₹300 Selling Price-ileeku add aagunnadh aanu.</p>
        <form id="itemForm">
          <input type="text" id="itemName" placeholder="Item Name" required>
          <input type="number" id="costPrice" placeholder="Material Cost (₹)" required>
          <input type="number" id="sellingPrice" placeholder="Selling Price (₹)" required>
          <button type="submit">Save Order</button>
        </form>
      </div>

      <!-- Items Table -->
      <div class="card">
        <h3>Orders List</h3>
        <table>
          <thead>
            <tr>
              <th>Date</th>
              <th>Item Name</th>
              <th>Material Cost</th>
              <th>Selling Price (+₹300)</th>
              <th>Advance / Received
                <div class="hint">* Material Cost-il ninnum baaki kittaathath hint aayi kaanam</div>
              </th>
              <th>Status</th>
              <th>Payment Action</th>
            </tr>
          </thead>
          <tbody id="itemsTable"></tbody>
        </table>
      </div>

      <script>
        async function fetchStoreInfo() {
          const res = await fetch('/api/store');
          const data = await res.json();
          document.getElementById('totalProfit').innerText = data.totalProfit;
        }

        async function fetchEmployees() {
          const res = await fetch('/api/employees');
          const emps = await res.json();
          const list = document.getElementById('empList');
          list.innerHTML = emps.map(e => \`<li><b>\${e.name}</b> (\${e.role})</li>\`).join('');
        }

        async function fetchItems() {
          const res = await fetch('/api/items');
          const items = await res.json();
          const tableBody = document.getElementById('itemsTable');
          tableBody.innerHTML = '';

          items.forEach(item => {
            const isFullPaid = item.paymentStatus === 'PAID';
            const remainingToCoverCost = item.costPrice - item.paidAmount;
            
            let advanceHint = "";
            if (remainingToCoverCost > 0) {
              advanceHint = \`<div class="hint">Material Cost-ileeku ഇനി ₹\${remainingToCoverCost} കൂടി വേണം</div>\`;
            } else {
              advanceHint = \`<div style="color: green; font-size: 11px;">Material Cost Covered!</div>\`;
            }

            const itemDate = new Date(item.createdAt).toLocaleDateString();

            tableBody.innerHTML += \`
              <tr>
                <td>\${itemDate}</td>
                <td>\${item.itemName}</td>
                <td>₹\${item.costPrice}</td>
                <td>₹\${item.sellingPrice}</td>
                <td>
                  ₹\${item.paidAmount}
                  \${advanceHint}
                </td>
                <td><span class="badge \${item.paymentStatus}">\${item.paymentStatus}</span></td>
                <td>
                  \${isFullPaid ? '<b>Completed</b>' : \`
                    <input type="number" id="pay_\${item._id}" placeholder="Amount" style="width: 80px;">
                    <button onclick="makePayment('\${item._id}')">Pay</button>
                  \`}
                </td>
              </tr>
            \`;
          });
        }

        async function runAnalysis() {
          const startDate = document.getElementById('startDate').value;
          const endDate = document.getElementById('endDate').value;
          
          let url = '/api/analysis';
          if(startDate && endDate) {
            url += \`?startDate=\${startDate}&endDate=\${endDate}\`;
          }

          const res = await fetch(url);
          const data = await res.json();

          document.getElementById('analysisResult').innerHTML = \`
            <p><b>Total Orders:</b> \${data.totalOrders}</p>
            <p><b>Total Material Cost:</b> ₹\${data.totalMaterialCost}</p>
            <p><b>Total Advance/Received:</b> ₹\${data.totalAdvance}</p>
            <p><b>Advance Balance Remaining (Against Cost):</b> ₹\${data.advanceRemaining}</p>
            <p><b>Total Profit From Completed Orders:</b> ₹\${data.totalProfitGenerated}</p>
          \`;
        }

        document.getElementById('empForm').addEventListener('submit', async (e) => {
          e.preventDefault();
          const name = document.getElementById('empName').value;
          const role = document.getElementById('empRole').value;

          await fetch('/api/employees', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ name, role })
          });

          document.getElementById('empForm').reset();
          fetchEmployees();
        });

        document.getElementById('itemForm').addEventListener('submit', async (e) => {
          e.preventDefault();
          const itemName = document.getElementById('itemName').value;
          const costPrice = document.getElementById('costPrice').value;
          const sellingPrice = document.getElementById('sellingPrice').value;

          await fetch('/api/items', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ itemName, costPrice, sellingPrice })
          });

          document.getElementById('itemForm').reset();
          loadData();
        });

        async function makePayment(itemId) {
          const amountPaidNow = document.getElementById('pay_' + itemId).value;
          if (!amountPaidNow || amountPaidNow <= 0) return alert('Valid amount enter cheyyuka');

          await fetch('/api/pay', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ itemId, amountPaidNow })
          });

          loadData();
        }

        function loadData() {
          fetchStoreInfo();
          fetchEmployees();
          fetchItems();
          runAnalysis();
        }

        loadData();
      </script>
    </body>
    </html>
  `);
});

// 5. Server Run
const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});
