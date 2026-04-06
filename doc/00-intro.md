composer create-project laravel/laravel pharmacy_system
cd pharmacy_system

php artisan serve --host=0.0.0.0 
DB_DATABASE=pharmacy_db
DB_USERNAME=root
DB_PASSWORD=

composer require laravel/breeze --dev
php artisan breeze:install
npm install
npm run build
php artisan migrate 
php artisan make:migration create_suppliers_table
 
Schema::create('suppliers', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('phone')->nullable();
    $table->string('address')->nullable();
    $table->timestamps();
});

php artisan make:migration create_medicines_table 
Schema::create('medicines', function (Blueprint $table) {
    $table->id();
    $table->string('trade_name');
    $table->string('generic_name')->nullable();
    $table->string('form')->nullable(); // tablet, syrup...
    $table->string('strength')->nullable(); // 500mg
    $table->string('company')->nullable();
    $table->decimal('purchase_price', 12, 2)->default(0);
    $table->decimal('sale_price', 12, 2)->default(0);
    $table->integer('min_stock_level')->default(5);
    $table->string('location')->nullable();
    $table->timestamps();
});

 php artisan make:migration create_batches_table
Schema::create('batches', function (Blueprint $table) {
    $table->id();
    $table->foreignId('medicine_id')->constrained()->onDelete('cascade');
    $table->date('expiry_date');
    $table->integer('quantity')->default(0);
    $table->timestamps();
});
 php artisan make:migration create_purchases_table
 Schema::create('purchases', function (Blueprint $table) {
    $table->id();
    $table->foreignId('supplier_id')->constrained()->onDelete('cascade');
    $table->string('invoice_number')->nullable();
    $table->date('purchase_date');
    $table->decimal('total', 12, 2)->default(0);
    $table->timestamps();
});
php artisan make:migration create_purchase_items_table
 Schema::create('purchase_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('purchase_id')->constrained()->onDelete('cascade');
    $table->foreignId('medicine_id')->constrained()->onDelete('cascade');
    $table->integer('quantity');
    $table->decimal('purchase_price', 12, 2);
    $table->date('expiry_date');
    $table->timestamps();
});
php artisan make:migration create_sales_table
Schema::create('sales', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade'); // cashier
    $table->dateTime('sale_date');
    $table->decimal('total', 12, 2)->default(0);
    $table->decimal('discount', 12, 2)->default(0);
    $table->decimal('paid', 12, 2)->default(0);
    $table->decimal('balance', 12, 2)->default(0);
    $table->timestamps();
});
php artisan make:migration create_sale_items_table
 php artisan migrate
php artisan make:model Supplier
php artisan make:model Medicine
php artisan make:model Batch
php artisan make:model Purchase
php artisan make:model PurchaseItem
php artisan make:model Sale
php artisan make:model SaleItem
$search = $request->search;

$medicines = Medicine::where('trade_name', 'LIKE', "%$search%")
    ->orWhere('generic_name', 'LIKE', "%$search%")
    ->paginate(20);
 $batches = Batch::where('medicine_id', $medicine_id)
    ->where('quantity', '>', 0)
    ->orderBy('expiry_date', 'ASC')
    ->get();

$qtyToSell = $requestQty;
foreach($batches as $batch){
    if($qtyToSell <= 0) break;
    if($batch->quantity >= $qtyToSell){
        $batch->quantity -= $qtyToSell;
        $batch->save();
        $qtyToSell = 0;
    } else {
        $qtyToSell -= $batch->quantity;
        $batch->quantity = 0;
        $batch->save();
    }
}

if($qtyToSell > 0){
    return back()->with('error', 'Not enough stock!');
}
 Batch::create([
    'medicine_id' => $medicine_id,
    'expiry_date' => $expiry_date,
    'quantity' => $qty,
]);
composer require barryvdh/laravel-dompdf
 resources/lang/en/messages.php
resources/lang/fr/messages.php
return [
    "sales" => "Sales",
    "purchase" => "Purchase",
];
{{ __('messages.sales') }}
 Laravel Blade + Bootstrap.
Bootstrap
 npm install bootstrap
npm run build
 routes/web.php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\MedicineController;
use App\Http\Controllers\SupplierController;
use App\Http\Controllers\PurchaseController;
use App\Http\Controllers\SaleController;
use App\Http\Controllers\ReportController;

Route::get('/', function () {
    return redirect()->route('dashboard');
});

Route::middleware(['auth'])->group(function () {

    Route::get('/dashboard', [ReportController::class, 'dashboard'])->name('dashboard');

    // Medicines
    Route::resource('medicines', MedicineController::class);

    // Suppliers
    Route::resource('suppliers', SupplierController::class);

    // Purchases
    Route::get('/purchases', [PurchaseController::class, 'index'])->name('purchases.index');
    Route::get('/purchases/create', [PurchaseController::class, 'create'])->name('purchases.create');
    Route::post('/purchases', [PurchaseController::class, 'store'])->name('purchases.store');
    Route::get('/purchases/{id}', [PurchaseController::class, 'show'])->name('purchases.show');

    // Sales
    Route::get('/sales', [SaleController::class, 'index'])->name('sales.index');
    Route::get('/sales/create', [SaleController::class, 'create'])->name('sales.create');
    Route::post('/sales', [SaleController::class, 'store'])->name('sales.store');
    Route::get('/sales/{id}', [SaleController::class, 'show'])->name('sales.show');

    // Invoice PDF
    Route::get('/sales/{id}/invoice', [SaleController::class, 'invoice'])->name('sales.invoice');

});
 php artisan make:controller MedicineController --resource
php artisan make:controller SupplierController --resource
php artisan make:controller PurchaseController
php artisan make:controller SaleController
php artisan make:controller ReportController
 app/Http/Controllers/MedicineController.php
 <?php

namespace App\Http\Controllers;

use App\Models\Medicine;
use Illuminate\Http\Request;

class MedicineController extends Controller
{
    public function index(Request $request)
    {
        $search = $request->search;

        $medicines = Medicine::query()
            ->when($search, function ($q) use ($search) {
                $q->where('trade_name', 'LIKE', "%$search%")
                  ->orWhere('generic_name', 'LIKE', "%$search%");
            })
            ->orderBy('trade_name')
            ->paginate(20);

        return view('medicines.index', compact('medicines', 'search'));
    }

    public function create()
    {
        return view('medicines.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'trade_name' => 'required',
            'sale_price' => 'required|numeric',
            'purchase_price' => 'required|numeric'
        ]);

        Medicine::create($request->all());

        return redirect()->route('medicines.index')->with('success', 'Medicine Added');
    }

    public function edit(Medicine $medicine)
    {
        return view('medicines.edit', compact('medicine'));
    }

    public function update(Request $request, Medicine $medicine)
    {
        $medicine->update($request->all());

        return redirect()->route('medicines.index')->with('success', 'Medicine Updated');
    }

    public function destroy(Medicine $medicine)
    {
        $medicine->delete();

        return redirect()->route('medicines.index')->with('success', 'Medicine Deleted');
    }
}
 app/Models/Medicine.php
protected $fillable = [
    'trade_name',
    'generic_name',
    'form',
    'strength',
    'company',
    'purchase_price',
    'sale_price',
    'min_stock_level',
    'location'
];
app/Http/Controllers/PurchaseController.php
<?php

namespace App\Http\Controllers;

use App\Models\Purchase;
use App\Models\PurchaseItem;
use App\Models\Supplier;
use App\Models\Medicine;
use App\Models\Batch;
use Illuminate\Http\Request;

class PurchaseController extends Controller
{
    public function index()
    {
        $purchases = Purchase::with('supplier')->orderBy('id', 'desc')->paginate(20);
        return view('purchases.index', compact('purchases'));
    }

    public function create()
    {
        $suppliers = Supplier::orderBy('name')->get();
        $medicines = Medicine::orderBy('trade_name')->get();

        return view('purchases.create', compact('suppliers', 'medicines'));
    }

    public function store(Request $request)
    {
        $request->validate([
            'supplier_id' => 'required',
            'purchase_date' => 'required',
            'items' => 'required|array'
        ]);

        $purchase = Purchase::create([
            'supplier_id' => $request->supplier_id,
            'invoice_number' => $request->invoice_number,
            'purchase_date' => $request->purchase_date,
            'total' => 0
        ]);

        $total = 0;

        foreach ($request->items as $item) {

            if(empty($item['medicine_id'])) continue;

            $subtotal = $item['quantity'] * $item['purchase_price'];
            $total += $subtotal;

            PurchaseItem::create([
                'purchase_id' => $purchase->id,
                'medicine_id' => $item['medicine_id'],
                'quantity' => $item['quantity'],
                'purchase_price' => $item['purchase_price'],
                'expiry_date' => $item['expiry_date']
            ]);

            // Add stock to batches
            Batch::create([
                'medicine_id' => $item['medicine_id'],
                'expiry_date' => $item['expiry_date'],
                'quantity' => $item['quantity']
            ]);
        }

        $purchase->update(['total' => $total]);

        return redirect()->route('purchases.index')->with('success', 'Purchase Saved Successfully');
    }

    public function show($id)
    {
        $purchase = Purchase::with('supplier')->findOrFail($id);
        $items = PurchaseItem::with('medicine')->where('purchase_id', $id)->get();

        return view('purchases.show', compact('purchase', 'items'));
    }
}
app/Http/Controllers/SaleController.php
<?php

namespace App\Http\Controllers;

use App\Models\Sale;
use App\Models\SaleItem;
use App\Models\Medicine;
use App\Models\Batch;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use PDF;

class SaleController extends Controller
{
    public function index()
    {
        $sales = Sale::with('user')->orderBy('id', 'desc')->paginate(20);
        return view('sales.index', compact('sales'));
    }

    public function create(Request $request)
    {
        $search = $request->search;

        $medicines = Medicine::query()
            ->when($search, function ($q) use ($search) {
                $q->where('trade_name', 'LIKE', "%$search%")
                  ->orWhere('generic_name', 'LIKE', "%$search%");
            })
            ->orderBy('trade_name')
            ->limit(30)
            ->get();

        return view('sales.create', compact('medicines', 'search'));
    }

    public function store(Request $request)
    {
        $request->validate([
            'items' => 'required|array',
            'paid' => 'required|numeric'
        ]);

        $sale = Sale::create([
            'user_id' => Auth::id(),
            'sale_date' => now(),
            'total' => 0,
            'discount' => $request->discount ?? 0,
            'paid' => $request->paid,
            'balance' => 0
        ]);

        $total = 0;

        foreach ($request->items as $item) {

            if(empty($item['medicine_id'])) continue;

            $medicine = Medicine::findOrFail($item['medicine_id']);
            $qtyToSell = $item['quantity'];

            // FIFO batches
            $batches = Batch::where('medicine_id', $medicine->id)
                ->where('quantity', '>', 0)
                ->orderBy('expiry_date', 'ASC')
                ->get();

            foreach ($batches as $batch) {
                if ($qtyToSell <= 0) break;

                if ($batch->quantity >= $qtyToSell) {
                    $batch->quantity -= $qtyToSell;
                    $batch->save();
                    $qtyToSell = 0;
                } else {
                    $qtyToSell -= $batch->quantity;
                    $batch->quantity = 0;
                    $batch->save();
                }
            }

            if ($qtyToSell > 0) {
                return back()->with('error', 'Not enough stock for: ' . $medicine->trade_name);
            }

            $subtotal = $item['quantity'] * $medicine->sale_price;
            $total += $subtotal;

            SaleItem::create([
                'sale_id' => $sale->id,
                'medicine_id' => $medicine->id,
                'quantity' => $item['quantity'],
                'sale_price' => $medicine->sale_price,
                'subtotal' => $subtotal
            ]);
        }

        $grandTotal = $total - ($request->discount ?? 0);
        $balance = $grandTotal - $request->paid;

        $sale->update([
            'total' => $grandTotal,
            'balance' => $balance
        ]);

        return redirect()->route('sales.show', $sale->id)->with('success', 'Sale Completed');
    }

    public function show($id)
    {
        $sale = Sale::with('user')->findOrFail($id);
        $items = SaleItem::with('medicine')->where('sale_id', $id)->get();

        return view('sales.show', compact('sale', 'items'));
    }

    public function invoice($id)
    {
        $sale = Sale::with('user')->findOrFail($id);
        $items = SaleItem::with('medicine')->where('sale_id', $id)->get();

        $pdf = PDF::loadView('sales.invoice', compact('sale', 'items'));
        return $pdf->download("invoice_$id.pdf");
    }
}
composer require barryvdh/laravel-dompdf
config/app.php
 Service Provider
 resources/views/layouts/app.blade.php
<!DOCTYPE html>
<html>
<head>
    <title>Pharmacy System</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">

<nav class="navbar navbar-expand-lg navbar-dark bg-primary px-3">
    <a class="navbar-brand text-white" href="{{ route('dashboard') }}">Pharmacy</a>

    <div class="ms-auto">
        <a class="btn btn-light btn-sm" href="{{ route('sales.create') }}">New Sale</a>
        <a class="btn btn-warning btn-sm" href="{{ route('purchases.create') }}">New Purchase</a>
    </div>
</nav>

<div class="container mt-3">
    @if(session('success'))
        <div class="alert alert-success">{{ session('success') }}</div>
    @endif

    @if(session('error'))
        <div class="alert alert-danger">{{ session('error') }}</div>
    @endif

    @yield('content')
</div>

</body>
</html>
resources/views/medicines/index.blade.php
@extends('layouts.app')

@section('content')

<h4>Medicines</h4>

<form method="GET" class="mb-3">
    <input type="text" name="search" value="{{ $search }}" class="form-control"
           placeholder="Search by Trade Name or Generic Name">
</form>

<a href="{{ route('medicines.create') }}" class="btn btn-success mb-2">Add Medicine</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>Trade Name</th>
        <th>Generic Name</th>
        <th>Sale Price (CFA)</th>
        <th>Purchase Price (CFA)</th>
        <th>Action</th>
    </tr>

    @foreach($medicines as $m)
    <tr>
        <td>{{ $m->trade_name }}</td>
        <td>{{ $m->generic_name }}</td>
        <td>{{ $m->sale_price }}</td>
        <td>{{ $m->purchase_price }}</td>
        <td>
            <a href="{{ route('medicines.edit', $m->id) }}" class="btn btn-sm btn-primary">Edit</a>
        </td>
    </tr>
    @endforeach
</table>

{{ $medicines->links() }}

@endsection
 resources/views/sales/invoice.blade.php
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: DejaVu Sans; font-size: 12px; }
        table { width:100%; border-collapse: collapse; }
        table, th, td { border: 1px solid black; padding: 5px; }
        th { background: #f2f2f2; }
    </style>
</head>
<body>

<h2>PHARMACY INVOICE</h2>

<p>
    <strong>Invoice No:</strong> {{ $sale->id }} <br>
    <strong>Date:</strong> {{ $sale->sale_date }} <br>
    <strong>Cashier:</strong> {{ $sale->user->name }}
</p>

<table>
    <tr>
        <th>Medicine</th>
        <th>Qty</th>
        <th>Price (CFA)</th>
        <th>Subtotal</th>
    </tr>

    @foreach($items as $item)
    <tr>
        <td>{{ $item->medicine->trade_name }}</td>
        <td>{{ $item->quantity }}</td>
        <td>{{ $item->sale_price }}</td>
        <td>{{ $item->subtotal }}</td>
    </tr>
    @endforeach
</table>

<br>

<p>
    <strong>Total:</strong> {{ $sale->total }} CFA <br>
    <strong>Paid:</strong> {{ $sale->paid }} CFA <br>
    <strong>Balance:</strong> {{ $sale->balance }} CFA
</p>

</body>
</html>
resources/views/sales/create.blade.php
@extends('layouts.app')

@section('content')

<h4>New Sale (POS)</h4>

<form method="GET" class="mb-3">
    <input type="text" name="search" value="{{ $search }}" class="form-control"
           placeholder="Search Trade Name / Generic Name">
</form>

<form method="POST" action="{{ route('sales.store') }}">
    @csrf

    <div class="row">
        <div class="col-md-8">

            <h5>Medicines List</h5>

            <div class="table-responsive">
                <table class="table table-bordered bg-white">
                    <tr>
                        <th>Medicine</th>
                        <th>Price (CFA)</th>
                        <th width="120">Qty</th>
                    </tr>

                    @foreach($medicines as $index => $m)
                    <tr>
                        <td>
                            <strong>{{ $m->trade_name }}</strong><br>
                            <small class="text-muted">{{ $m->generic_name }}</small>
                        </td>
                        <td>{{ $m->sale_price }}</td>
                        <td>
                            <input type="hidden" name="items[{{ $index }}][medicine_id]" value="{{ $m->id }}">

                            <input type="number" name="items[{{ $index }}][quantity]" value="0" min="0"
                                   class="form-control">
                        </td>
                    </tr>
                    @endforeach
                </table>
            </div>

        </div>

        <div class="col-md-4">

            <h5>Payment</h5>

            <div class="mb-2">
                <label>Discount (CFA)</label>
                <input type="number" name="discount" value="0" class="form-control">
            </div>

            <div class="mb-2">
                <label>Paid (CFA)</label>
                <input type="number" name="paid" value="0" class="form-control" required>
            </div>

            <button class="btn btn-success w-100 mt-2">Complete Sale</button>

        </div>
    </div>

</form>

@endsection
resources/views/sales/show.blade.php
@extends('layouts.app')

@section('content')

<h4>Sale Details</h4>

<p>
    <strong>Sale ID:</strong> {{ $sale->id }} <br>
    <strong>Date:</strong> {{ $sale->sale_date }} <br>
    <strong>Cashier:</strong> {{ $sale->user->name }}
</p>

<table class="table table-bordered bg-white">
    <tr>
        <th>Medicine</th>
        <th>Qty</th>
        <th>Price</th>
        <th>Subtotal</th>
    </tr>

    @foreach($items as $item)
    <tr>
        <td>{{ $item->medicine->trade_name }}</td>
        <td>{{ $item->quantity }}</td>
        <td>{{ $item->sale_price }}</td>
        <td>{{ $item->subtotal }}</td>
    </tr>
    @endforeach
</table>

<p>
    <strong>Total:</strong> {{ $sale->total }} CFA <br>
    <strong>Paid:</strong> {{ $sale->paid }} CFA <br>
    <strong>Balance:</strong> {{ $sale->balance }} CFA
</p>

<a href="{{ route('sales.invoice', $sale->id) }}" class="btn btn-primary">
    Download Invoice PDF
</a>

@endsection
 resources/views/sales/index.blade.php
@extends('layouts.app')

@section('content')

<h4>Sales History</h4>

<table class="table table-bordered bg-white">
    <tr>
        <th>ID</th>
        <th>Date</th>
        <th>Total</th>
        <th>Paid</th>
        <th>Balance</th>
        <th>Cashier</th>
        <th>Action</th>
    </tr>

    @foreach($sales as $s)
    <tr>
        <td>{{ $s->id }}</td>
        <td>{{ $s->sale_date }}</td>
        <td>{{ $s->total }} CFA</td>
        <td>{{ $s->paid }} CFA</td>
        <td>{{ $s->balance }} CFA</td>
        <td>{{ $s->user->name }}</td>
        <td>
            <a href="{{ route('sales.show', $s->id) }}" class="btn btn-sm btn-primary">View</a>
        </td>
    </tr>
    @endforeach
</table>

{{ $sales->links() }}

@endsection
resources/views/purchases/create.blade.php
@extends('layouts.app')

@section('content')

<h4>New Purchase</h4>

<form method="POST" action="{{ route('purchases.store') }}">
@csrf

<div class="mb-2">
    <label>Supplier</label>
    <select name="supplier_id" class="form-control" required>
        <option value="">-- Select Supplier --</option>
        @foreach($suppliers as $s)
            <option value="{{ $s->id }}">{{ $s->name }}</option>
        @endforeach
    </select>
</div>

<div class="mb-2">
    <label>Invoice Number</label>
    <input type="text" name="invoice_number" class="form-control">
</div>

<div class="mb-2">
    <label>Purchase Date</label>
    <input type="date" name="purchase_date" class="form-control" required>
</div>

<hr>

<h5>Medicines</h5>

<div class="table-responsive">
<table class="table table-bordered bg-white">
    <tr>
        <th>Medicine</th>
        <th width="120">Qty</th>
        <th width="150">Purchase Price</th>
        <th width="160">Expiry Date</th>
    </tr>

    @for($i=0; $i<15; $i++)
    <tr>
        <td>
            <select name="items[{{ $i }}][medicine_id]" class="form-control">
                <option value="">-- Select Medicine --</option>
                @foreach($medicines as $m)
                    <option value="{{ $m->id }}">{{ $m->trade_name }} ({{ $m->generic_name }})</option>
                @endforeach
            </select>
        </td>

        <td>
            <input type="number" name="items[{{ $i }}][quantity]" value="0" min="0" class="form-control">
        </td>

        <td>
            <input type="number" name="items[{{ $i }}][purchase_price]" value="0" min="0" class="form-control">
        </td>

        <td>
            <input type="date" name="items[{{ $i }}][expiry_date]" class="form-control">
        </td>
    </tr>
    @endfor

</table>
</div>

<button class="btn btn-success w-100">Save Purchase</button>

</form>

@endsection
resources/views/purchases/index.blade.php
 @extends('layouts.app')

@section('content')

<h4>Purchases</h4>

<a href="{{ route('purchases.create') }}" class="btn btn-success mb-2">New Purchase</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>ID</th>
        <th>Supplier</th>
        <th>Date</th>
        <th>Total</th>
        <th>Action</th>
    </tr>

    @foreach($purchases as $p)
    <tr>
        <td>{{ $p->id }}</td>
        <td>{{ $p->supplier->name }}</td>
        <td>{{ $p->purchase_date }}</td>
        <td>{{ $p->total }} CFA</td>
        <td>
            <a href="{{ route('purchases.show', $p->id) }}" class="btn btn-sm btn-primary">View</a>
        </td>
    </tr>
    @endforeach
</table>

{{ $purchases->links() }}

@endsection
 resources/views/purchases/show.blade.php
 @extends('layouts.app')

@section('content')

<h4>Purchase Details</h4>

<p>
    <strong>Purchase ID:</strong> {{ $purchase->id }} <br>
    <strong>Supplier:</strong> {{ $purchase->supplier->name }} <br>
    <strong>Date:</strong> {{ $purchase->purchase_date }} <br>
    <strong>Total:</strong> {{ $purchase->total }} CFA
</p>

<table class="table table-bordered bg-white">
    <tr>
        <th>Medicine</th>
        <th>Qty</th>
        <th>Price</th>
        <th>Expiry</th>
    </tr>

    @foreach($items as $item)
    <tr>
        <td>{{ $item->medicine->trade_name }}</td>
        <td>{{ $item->quantity }}</td>
        <td>{{ $item->purchase_price }}</td>
        <td>{{ $item->expiry_date }}</td>
    </tr>
    @endforeach
</table>

@endsection
 app/Models/Purchase.php
public function supplier()
{
    return $this->belongsTo(Supplier::class);
}
app/Models/Sale.php
public function user()
{
    return $this->belongsTo(User::class);
}
 app/Models/PurchaseItem.php
public function medicine()
{
    return $this->belongsTo(Medicine::class);
}
app/Models/SaleItem.php
public function medicine()
{
    return $this->belongsTo(Medicine::class);
}
ReportController.php
<?php

namespace App\Http\Controllers;

use App\Models\Sale;
use App\Models\Purchase;
use App\Models\Batch;
use App\Models\Medicine;
use Illuminate\Support\Facades\DB;

class ReportController extends Controller
{
    public function dashboard()
    {
        $todaySales = Sale::whereDate('sale_date', today())->sum('total');
        $todayPurchases = Purchase::whereDate('purchase_date', today())->sum('total');

        $stockSummary = Batch::select('medicine_id', DB::raw('SUM(quantity) as total_qty'))
            ->groupBy('medicine_id')
            ->get();

        $lowStock = [];

        foreach($stockSummary as $row){
            $medicine = Medicine::find($row->medicine_id);
            if($medicine && $row->total_qty <= $medicine->min_stock_level){
                $lowStock[] = [
                    'medicine' => $medicine->trade_name,
                    'qty' => $row->total_qty
                ];
            }
        }

        return view('dashboard', compact('todaySales', 'todayPurchases', 'lowStock'));
    }
}
resources/views/dashboard.blade.php
@extends('layouts.app')

@section('content')

<h4>Dashboard</h4>

<div class="row">
    <div class="col-md-6">
        <div class="card p-3 mb-2">
            <h5>Today Sales</h5>
            <h3>{{ $todaySales }} CFA</h3>
        </div>
    </div>

    <div class="col-md-6">
        <div class="card p-3 mb-2">
            <h5>Today Purchases</h5>
            <h3>{{ $todayPurchases }} CFA</h3>
        </div>
    </div>
</div>

<div class="card p-3 mt-3">
    <h5>Low Stock Alert</h5>

    @if(count($lowStock) == 0)
        <p class="text-success">No low stock medicines</p>
    @else
        <table class="table table-bordered">
            <tr>
                <th>Medicine</th>
                <th>Qty</th>
            </tr>
            @foreach($lowStock as $item)
            <tr>
                <td>{{ $item['medicine'] }}</td>
                <td>{{ $item['qty'] }}</td>
            </tr>
            @endforeach
        </table>
    @endif
</div>

@endsection
resources/views/medicines/create.blade.php
@extends('layouts.app')

@section('content')

<h4>Add Medicine</h4>

<form method="POST" action="{{ route('medicines.store') }}">
@csrf

<div class="mb-2">
    <label>Trade Name</label>
    <input type="text" name="trade_name" class="form-control" required>
</div>

<div class="mb-2">
    <label>Generic Name</label>
    <input type="text" name="generic_name" class="form-control">
</div>

<div class="mb-2">
    <label>Form (Tablet/Syrup/Injection)</label>
    <input type="text" name="form" class="form-control">
</div>

<div class="mb-2">
    <label>Strength (ex: 500mg)</label>
    <input type="text" name="strength" class="form-control">
</div>

<div class="mb-2">
    <label>Company</label>
    <input type="text" name="company" class="form-control">
</div>

<div class="mb-2">
    <label>Purchase Price (CFA)</label>
    <input type="number" name="purchase_price" class="form-control" required>
</div>

<div class="mb-2">
    <label>Sale Price (CFA)</label>
    <input type="number" name="sale_price" class="form-control" required>
</div>

<div class="mb-2">
    <label>Min Stock Level</label>
    <input type="number" name="min_stock_level" value="5" class="form-control">
</div>

<div class="mb-2">
    <label>Location (Shelf)</label>
    <input type="text" name="location" class="form-control">
</div>

<button class="btn btn-success w-100">Save Medicine</button>

</form>

@endsection
resources/views/medicines/edit.blade.php
@extends('layouts.app')

@section('content')

<h4>Edit Medicine</h4>

<form method="POST" action="{{ route('medicines.update', $medicine->id) }}">
@csrf
@method('PUT')

<div class="mb-2">
    <label>Trade Name</label>
    <input type="text" name="trade_name" value="{{ $medicine->trade_name }}" class="form-control" required>
</div>

<div class="mb-2">
    <label>Generic Name</label>
    <input type="text" name="generic_name" value="{{ $medicine->generic_name }}" class="form-control">
</div>

<div class="mb-2">
    <label>Form</label>
    <input type="text" name="form" value="{{ $medicine->form }}" class="form-control">
</div>

<div class="mb-2">
    <label>Strength</label>
    <input type="text" name="strength" value="{{ $medicine->strength }}" class="form-control">
</div>

<div class="mb-2">
    <label>Company</label>
    <input type="text" name="company" value="{{ $medicine->company }}" class="form-control">
</div>

<div class="mb-2">
    <label>Purchase Price (CFA)</label>
    <input type="number" name="purchase_price" value="{{ $medicine->purchase_price }}" class="form-control" required>
</div>

<div class="mb-2">
    <label>Sale Price (CFA)</label>
    <input type="number" name="sale_price" value="{{ $medicine->sale_price }}" class="form-control" required>
</div>

<div class="mb-2">
    <label>Min Stock Level</label>
    <input type="number" name="min_stock_level" value="{{ $medicine->min_stock_level }}" class="form-control">
</div>

<div class="mb-2">
    <label>Location</label>
    <input type="text" name="location" value="{{ $medicine->location }}" class="form-control">
</div>

<button class="btn btn-primary w-100">Update Medicine</button>

</form>

@endsection
app/Http/Controllers/SupplierController.php
<?php

namespace App\Http\Controllers;

use App\Models\Supplier;
use Illuminate\Http\Request;

class SupplierController extends Controller
{
    public function index()
    {
        $suppliers = Supplier::orderBy('name')->paginate(20);
        return view('suppliers.index', compact('suppliers'));
    }

    public function create()
    {
        return view('suppliers.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required'
        ]);

        Supplier::create($request->all());

        return redirect()->route('suppliers.index')->with('success', 'Supplier Added');
    }

    public function edit(Supplier $supplier)
    {
        return view('suppliers.edit', compact('supplier'));
    }

    public function update(Request $request, Supplier $supplier)
    {
        $supplier->update($request->all());

        return redirect()->route('suppliers.index')->with('success', 'Supplier Updated');
    }

    public function destroy(Supplier $supplier)
    {
        $supplier->delete();
        return redirect()->route('suppliers.index')->with('success', 'Supplier Deleted');
    }
}
app/Models/Supplier.php
protected $fillable = [
    'name',
    'phone',
    'address'
];
resources/views/suppliers/index.blade.php

@extends('layouts.app')

@section('content')

<h4>Suppliers</h4>

<a href="{{ route('suppliers.create') }}" class="btn btn-success mb-2">Add Supplier</a>

<table class="table table-bordered bg-white">
    <tr>
        <th>Name</th>
        <th>Phone</th>
        <th>Address</th>
        <th>Action</th>
    </tr>

    @foreach($suppliers as $s)
    <tr>
        <td>{{ $s->name }}</td>
        <td>{{ $s->phone }}</td>
        <td>{{ $s->address }}</td>
        <td>
            <a href="{{ route('suppliers.edit', $s->id) }}" class="btn btn-sm btn-primary">Edit</a>
        </td>
    </tr>
    @endforeach
</table>

{{ $suppliers->links() }}

@endsection
resources/views/suppliers/create.blade.php
@extends('layouts.app')

@section('content')

<h4>Add Supplier</h4>

<form method="POST" action="{{ route('suppliers.store') }}">
@csrf

<div class="mb-2">
    <label>Supplier Name</label>
    <input type="text" name="name" class="form-control" required>
</div>

<div class="mb-2">
    <label>Phone</label>
    <input type="text" name="phone" class="form-control">
</div>

<div class="mb-2">
    <label>Address</label>
    <input type="text" name="address" class="form-control">
</div>

<button class="btn btn-success w-100">Save Supplier</button>

</form>

@endsection
resources/views/suppliers/edit.blade.php
@extends('layouts.app')

@section('content')

<h4>Edit Supplier</h4>

<form method="POST" action="{{ route('suppliers.update', $supplier->id) }}">
@csrf
@method('PUT')

<div class="mb-2">
    <label>Supplier Name</label>
    <input type="text" name="name" value="{{ $supplier->name }}" class="form-control" required>
</div>

<div class="mb-2">
    <label>Phone</label>
    <input type="text" name="phone" value="{{ $supplier->phone }}" class="form-control">
</div>

<div class="mb-2">
    <label>Address</label>
    <input type="text" name="address" value="{{ $supplier->address }}" class="form-control">
</div>

<button class="btn btn-primary w-100">Update Supplier</button>

</form>

@endsection
SaleController
if($item['quantity'] <= 0) continue;
php artisan make:controller StockController
routes/web.php
use App\Http\Controllers\StockController;

Route::get('/stock', [StockController::class, 'index'])->name('stock.index');
app/Http/Controllers/StockController.php
<?php

namespace App\Http\Controllers;

use App\Models\Medicine;
use App\Models\Batch;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class StockController extends Controller
{
    public function index(Request $request)
    {
        $search = $request->search;

        $stocks = Medicine::query()
            ->when($search, function ($q) use ($search) {
                $q->where('trade_name', 'LIKE', "%$search%")
                  ->orWhere('generic_name', 'LIKE', "%$search%");
            })
            ->select('medicines.*')
            ->selectSub(function ($query) {
                $query->from('batches')
                    ->selectRaw('SUM(quantity)')
                    ->whereColumn('batches.medicine_id', 'medicines.id');
            }, 'total_qty')
            ->selectSub(function ($query) {
                $query->from('batches')
                    ->selectRaw('MIN(expiry_date)')
                    ->whereColumn('batches.medicine_id', 'medicines.id')
                    ->where('quantity', '>', 0);
            }, 'nearest_expiry')
            ->orderBy('trade_name')
            ->paginate(30);

        return view('stock.index', compact('stocks', 'search'));
    }
}
resources/views/stock/index.blade.php
@extends('layouts.app')

@section('content')

<h4>Stock Report</h4>

<form method="GET" class="mb-3">
    <input type="text" name="search" value="{{ $search }}" class="form-control"
           placeholder="Search by Trade Name or Generic Name">
</form>

<table class="table table-bordered bg-white">
    <tr>
        <th>Trade Name</th>
        <th>Generic Name</th>
        <th>Total Qty</th>
        <th>Min Stock</th>
        <th>Nearest Expiry</th>
        <th>Status</th>
    </tr>

    @foreach($stocks as $s)
    <tr>
        <td>{{ $s->trade_name }}</td>
        <td>{{ $s->generic_name }}</td>
        <td>{{ $s->total_qty ?? 0 }}</td>
        <td>{{ $s->min_stock_level }}</td>
        <td>{{ $s->nearest_expiry ?? '-' }}</td>

        <td>
            @if(($s->total_qty ?? 0) <= $s->min_stock_level)
                <span class="badge bg-danger">LOW</span>
            @else
                <span class="badge bg-success">OK</span>
            @endif
        </td>
    </tr>
    @endforeach
</table>

{{ $stocks->links() }}

@endsection
resources/views/layouts/app.blade.php
<a class="btn btn-info btn-sm" href="{{ route('stock.index') }}">Stock</a>
ms-auto
<div class="ms-auto">
    <a class="btn btn-light btn-sm" href="{{ route('sales.create') }}">New Sale</a>
    <a class="btn btn-warning btn-sm" href="{{ route('purchases.create') }}">New Purchase</a>
    <a class="btn btn-info btn-sm" href="{{ route('stock.index') }}">Stock</a>
</div>
routes/web.php
Route::get('/expiry-report', [StockController::class, 'expiryReport'])->name('stock.expiry');
app/Http/Controllers/StockController.php
public function expiryReport(Request $request)
{
    $days = $request->days ?? 30;

    $expiryBatches = Batch::with('medicine')
        ->where('quantity', '>', 0)
        ->whereDate('expiry_date', '<=', now()->addDays($days))
        ->orderBy('expiry_date', 'ASC')
        ->get();

    return view('stock.expiry', compact('expiryBatches', 'days'));
}
app/Models/Batch.php
public function medicine()
{
    return $this->belongsTo(Medicine::class);
}
resources/views/stock/expiry.blade.php
@extends('layouts.app')

@section('content')

<h4>Expiry Report</h4>

<form method="GET" class="mb-3">
    <label>Show medicines expiring within (days)</label>
    <input type="number" name="days" value="{{ $days }}" class="form-control" style="max-width:200px;">
    <button class="btn btn-primary mt-2">Filter</button>
</form>

<table class="table table-bordered bg-white">
    <tr>
        <th>Medicine</th>
        <th>Generic Name</th>
        <th>Qty</th>
        <th>Expiry Date</th>
        <th>Status</th>
    </tr>

    @foreach($expiryBatches as $b)
    <tr>
        <td>{{ $b->medicine->trade_name }}</td>
        <td>{{ $b->medicine->generic_name }}</td>
        <td>{{ $b->quantity }}</td>
        <td>{{ $b->expiry_date }}</td>

        <td>
            @if(\Carbon\Carbon::parse($b->expiry_date)->isPast())
                <span class="badge bg-danger">EXPIRED</span>
            @else
                <span class="badge bg-warning text-dark">NEAR EXPIRY</span>
            @endif
        </td>
    </tr>
    @endforeach
</table>

@endsection
resources/views/layouts/app.blade.php
<a class="btn btn-danger btn-sm" href="{{ route('stock.expiry') }}">Expiry</a>
ms-auto
<div class="ms-auto">
    <a class="btn btn-light btn-sm" href="{{ route('sales.create') }}">New Sale</a>
    <a class="btn btn-warning btn-sm" href="{{ route('purchases.create') }}">New Purchase</a>
    <a class="btn btn-info btn-sm" href="{{ route('stock.index') }}">Stock</a>
    <a class="btn btn-danger btn-sm" href="{{ route('stock.expiry') }}">Expiry</a>
</div>
routes/web.php
use App\Http\Controllers\BackupController;

Route::get('/backup', [BackupController::class, 'download'])->name('backup.download');
php artisan make:controller BackupController
app/Http/Controllers/BackupController.php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\Response;

class BackupController extends Controller
{
    public function download()
    {
        $dbHost = env('DB_HOST', '127.0.0.1');
        $dbUser = env('DB_USERNAME', 'root');
        $dbPass = env('DB_PASSWORD', '');
        $dbName = env('DB_DATABASE', 'pharmacy_db');

        $fileName = "backup_" . date("Y-m-d_H-i-s") . ".sql";
        $filePath = storage_path("app/" . $fileName);

        // mysqldump command
        $command = "mysqldump --host=$dbHost --user=$dbUser";

        if (!empty($dbPass)) {
            $command .= " --password=$dbPass";
        }

        $command .= " $dbName > \"$filePath\"";

        $result = null;
        $output = null;
        exec($command, $output, $result);

        if ($result !== 0) {
            return back()->with('error', 'Backup failed! mysqldump not found or permission denied.');
        }

        return Response::download($filePath)->deleteFileAfterSend(true);
    }
}
resources/views/layouts/app.blade.php
<a class="btn btn-dark btn-sm" href="{{ route('backup.download') }}">Backup</a>
<div class="ms-auto">
    <a class="btn btn-light btn-sm" href="{{ route('sales.create') }}">New Sale</a>
    <a class="btn btn-warning btn-sm" href="{{ route('purchases.create') }}">New Purchase</a>
    <a class="btn btn-info btn-sm" href="{{ route('stock.index') }}">Stock</a>
    <a class="btn btn-danger btn-sm" href="{{ route('stock.expiry') }}">Expiry</a>
    <a class="btn btn-dark btn-sm" href="{{ route('backup.download') }}">Backup</a>
</div>
routes/web.php
use App\Http\Controllers\BackupController;

Route::get('/restore', [BackupController::class, 'restoreForm'])->name('backup.restore.form');
Route::post('/restore', [BackupController::class, 'restore'])->name('backup.restore.submit');
resources/views/backup/restore.blade.php
@extends('layouts.app')

@section('content')

<h4>Restore Database (Admin Only)</h4>

@if(session('error'))
    <div class="alert alert-danger">{{ session('error') }}</div>
@endif

@if(session('success'))
    <div class="alert alert-success">{{ session('success') }}</div>
@endif

<form method="POST" action="{{ route('backup.restore.submit') }}" enctype="multipart/form-data">
@csrf

<div class="mb-2">
    <label>Admin Password</label>
    <input type="password" name="admin_password" class="form-control" required>
</div>

<div class="mb-2">
    <label>Upload SQL File</label>
    <input type="file" name="sql_file" class="form-control" accept=".sql" required>
</div>

<button class="btn btn-danger w-100">Restore Database</button>
</form>

@endsection
app/Http/Controllers/BackupController.php
public function restoreForm()
{
    return view('backup.restore');
}

public function restore(Request $request)
{
    $request->validate([
        'admin_password' => 'required',
        'sql_file' => 'required|mimes:sql'
    ]);

    // تحقق من كلمة مرور الـ Admin (يمكنك استخدام نفس كلمة مرور الـ Admin في users table)
    $adminPassword = $request->admin_password;

    $admin = auth()->user();
    if (!\Hash::check($adminPassword, $admin->password)) {
        return back()->with('error', 'Incorrect Admin Password!');
    }

    // حفظ الملف مؤقتاً
    $file = $request->file('sql_file');
    $filePath = $file->getRealPath();

    $dbHost = env('DB_HOST', '127.0.0.1');
    $dbUser = env('DB_USERNAME', 'root');
    $dbPass = env('DB_PASSWORD', '');
    $dbName = env('DB_DATABASE', 'pharmacy_db');

    $command = "mysql --host=$dbHost --user=$dbUser";

    if (!empty($dbPass)) {
        $command .= " --password=$dbPass";
    }

    $command .= " $dbName < \"$filePath\"";

    $result = null;
    exec($command, $output, $result);

    if ($result !== 0) {
        return back()->with('error', 'Restore failed! Check SQL file or permissions.');
    }

    return back()->with('success', 'Database restored successfully!');
}
resources/views/layouts/app.blade.php
@if(auth()->user()->role == 'Admin')
    <a class="btn btn-danger btn-sm" href="{{ route('backup.restore.form') }}">Restore</a>
@endif
