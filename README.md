<!doctype html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>برنامج حسابات الدكانة</title>
<style>
body {font-family: "Segoe UI", Arial; background: #f6f6f6; color: #222; max-width: 1000px; margin: auto; padding: 20px;}
h1 {text-align: center;}
.card {background: white; border-radius: 10px; padding: 15px; margin-bottom: 15px; box-shadow: 0 2px 10px rgba(0,0,0,0.05);}
input, select, button {padding: 7px; border: 1px solid #ccc; border-radius: 6px; margin:2px;}
table {width: 100%; border-collapse: collapse; margin-top: 10px;}
th, td {padding: 8px; border-bottom: 1px solid #eee; text-align: center;}
.btn {background: #444; color: white; border: none; cursor: pointer; margin:2px;}
.btn:hover {background: #222;}
.hidden {display:none;}
</style>
</head>
<body>

<h1>📦 برنامج حسابات الدكانة</h1>

<div class="card">
  <button class="btn" onclick="showSection('current')">🏪 المخزون الحالي</button>
  <button class="btn" onclick="showSection('finished')">❌ الأصناف المنتهية</button>
  <button class="btn" onclick="undoLast()">↩️ تراجع عن آخر عملية</button>
</div>

<div id="current" class="section">
  <div class="card">
    <h3>إضافة صنف جديد</h3>
    <form id="addProductForm">
      <input name="name" placeholder="اسم الصنف" required>
      <input name="cost" type="number" step="0.01" placeholder="سعر الشراء (للحبة)" required>
      <input name="price" type="number" step="0.01" placeholder="سعر البيع (للحبة)" required>
      <input name="cartonQty" type="number" placeholder="عدد الحبات داخل الكرتونة" value="1" required>
      <input name="qty" type="number" placeholder="عدد الكراتين الحالية" value="0" required>
      <button type="submit" class="btn">إضافة الصنف</button>
    </form>
  </div>

  <div class="card">
    <h3>عملية بيع / شراء</h3>
    <form id="txnForm">
      <select name="productId" id="productSelect" required></select>
      <select name="type">
        <option value="buy">شراء</option>
        <option value="sell">بيع</option>
      </select>
      <select name="unitType">
        <option value="carton">كرتونة</option>
        <option value="piece">حبة</option>
      </select>
      <input name="qty" type="number" value="1" required>
      <input name="unitPrice" type="number" step="0.01" placeholder="السعر للوحدة" required>
      <button type="submit" class="btn">تسجيل</button>
    </form>
  </div>

  <div class="card">
    <h3>الأصناف</h3>
    <div id="productsArea"></div>
  </div>

  <div class="card">
    <h3>الملخص</h3>
    <div id="summaryArea"></div>
  </div>
</div>

<div id="finished" class="section hidden">
  <div class="card">
    <h3>الأصناف المنتهية</h3>
    <div id="finishedArea"></div>
  </div>
</div>

<script>
const STORAGE_KEY = "dakane_data_v4";
let state = JSON.parse(localStorage.getItem(STORAGE_KEY) || '{"products":[],"txns":[],"finished":[],"history":[]}');

function save() { localStorage.setItem(STORAGE_KEY, JSON.stringify(state)); }

function showSection(id){
  document.querySelectorAll('.section').forEach(s => s.classList.add('hidden'));
  document.getElementById(id).classList.remove('hidden');
}

function refresh(){
  const select = document.getElementById("productSelect");
  select.innerHTML = "";
  const activeProducts = state.products.filter(p => p.totalPieces > 0);
  activeProducts.forEach(p => {
    const o = document.createElement("option");
    o.value = p.id;
    o.textContent = p.name;
    select.appendChild(o);
  });

  const pa = document.getElementById("productsArea");
  if(activeProducts.length ===0){
    pa.innerHTML = "<p>لا يوجد أصناف متوفرة حالياً.</p>";
  }else{
    pa.innerHTML = `<table>
      <tr><th>الصنف</th><th>سعر الشراء (للحبة)</th><th>سعر البيع (للحبة)</th><th>عدد الكراتين</th><th>عدد الحبات</th><th>الحبات المتبقية</th></tr>`+
      activeProducts.map(p=>`<tr>
        <td>${p.name}</td>
        <td>${p.cost}</td>
        <td>${p.price}</td>
        <td>${p.qty}</td>
        <td>${p.totalPieces}</td>
        <td>${p.totalPieces}</td>
      </tr>`).join("")+`</table>`;
  }

  const fa = document.getElementById("finishedArea");
  if(state.finished.length ===0) fa.innerHTML="<p>لا توجد أصناف منتهية بعد.</p>";
  else{
    fa.innerHTML = `<table><tr><th>الصنف</th><th>آخر عملية بيع</th></tr>`+
    state.finished.map(p=>`<tr><td>${p.name}</td><td>${new Date(p.lastSold).toLocaleString()}</td></tr>`).join("")+
    `</table>`;
  }

  let totalBuy=0, totalSell=0, totalProfit=0;
  state.txns.forEach(t=>{
    const prod = state.products.find(x=>x.id===t.productId)||state.finished.find(x=>x.id===t.productId);
    if(t.type==="buy") totalBuy += t.qty*t.unitPrice;
    else if(t.type==="sell") { totalSell += t.qty*t.unitPrice; totalProfit+=(t.unitPrice-(prod?prod.cost:0))*t.qty; }
  });
  document.getElementById("summaryArea").innerHTML=`
    <p>إجمالي المشتريات: ${totalBuy.toFixed(2)}</p>
    <p>إجمالي المبيعات: ${totalSell.toFixed(2)}</p>
    <p>الربح الصافي: ${totalProfit.toFixed(2)}</p>
  `;
  save();
}

// إضافة صنف جديد
document.getElementById("addProductForm").onsubmit = e=>{
  e.preventDefault();
  const fd = new FormData(e.target);
  const cartonQty = parseInt(fd.get("cartonQty"));
  const qty = parseInt(fd.get("qty"));
  const newProd = {
    id: Date.now().toString(),
    name: fd.get("name"),
    cost: parseFloat(fd.get("cost")),
    price: parseFloat(fd.get("price")),
    cartonQty: cartonQty,
    qty: qty,
    totalPieces: qty*cartonQty
  };
  state.products.push(newProd);
  state.history.push({type:"add", product:{...newProd}});
  e.target.reset();
  refresh();
};

// عملية بيع / شراء
document.getElementById("txnForm").onsubmit = e=>{
  e.preventDefault();
  const fd = new FormData(e.target);
  const prod = state.products.find(x=>x.id===fd.get("productId"));
  const type = fd.get("type");
  const unitType = fd.get("unitType");
  const qty = parseInt(fd.get("qty"));
  const price = parseFloat(fd.get("unitPrice"));
  if(!prod) return alert("اختر صنف صحيح");

  let qtyPieces = qty;
  if(unitType==="carton") qtyPieces *= prod.cartonQty;

  state.history.push({type:"txn", productId:prod.id, prev:{...prod}, txn:{type, qty:qtyPieces, unitPrice:price}});

  if(type==="buy") {
    prod.totalPieces += qtyPieces;
    if(unitType==="carton") prod.qty += qty;
  } else if(type==="sell"){
    if(qtyPieces>prod.totalPieces && !confirm("الكمية أكبر من المخزون، هل تريد المتابعة؟")) return;
    prod.totalPieces -= qtyPieces;
    if(unitType==="carton") prod.qty -= qty;
    if(prod.totalPieces<=0){
      prod.lastSold = Date.now();
      state.finished.push(prod);
      state.products = state.products.filter(p=>p.totalPieces>0);
    }
  }

  state.txns.push({date:Date.now(),productId:prod.id,type,qty:qtyPieces,unitPrice:price});
  refresh();
};

// التراجع عن آخر عملية
function undoLast(){
  if(state.history.length===0) return alert("لا توجد عملية للتراجع");
  const last = state.history.pop();
  if(last.type==="add"){
    state.products = state.products.filter(p=>p.id!==last.product.id);
  } else if(last.type==="txn"){
    const prod = state.products.find(p=>p.id===last.productId) || state.finished.find(p=>p.id===last.productId);
    if(prod){
      Object.assign(prod,last.prev);
      if(prod.totalPieces>0 && !state.products.includes(prod)){
        state.products.push(prod);
        state.finished = state.finished.filter(f=>f.id!==prod.id);
      }
    }
    // احذف آخر txn إذا كانت بيع/شراء
    state.txns = state.txns.slice(0,-1);
  }
  refresh();
}

refresh();
</script>

</body>
</html>
