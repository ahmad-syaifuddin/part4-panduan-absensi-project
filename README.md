<div align="center">

<p> <b>Visitors Count 👁️</b> </p>
<img src="https://profile-counter.deno.dev/part4-panduan-absensi-project/count.svg" alt="Profile Counter Repo :: Visitor's Count" />

</div>

<div align="center">

# 🚀 Part 4: Panduan Absensi Project

### Export Data & Laporan Absensi yang Kece Badai! ✨

</div>

## 📤 Export Excel Rekap Bulanan (Admin Side)

Sekarang kita tambahin tombol "Export ke Excel" di halaman laporan bulanan.

---

## 📱 Export Riwayat Absensi (Sisi Karyawan)

Sekarang giliran karyawan yang bisa download riwayat absensi mereka sendiri! Support Excel dan PDF. 🎉

---

### 🔧 Langkah 1: Bikin Class Export untuk Karyawan

Jalankan command Artisan:

```bash
php artisan make:export MyAttendanceHistoryExport
```

Buka `app/Exports/MyAttendanceHistoryExport.php`:

```php
<?php

namespace App\Exports;

use App\Models\Attendance;
use Illuminate\Support\Facades\Auth;
use Maatwebsite\Excel\Concerns\FromCollection;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use Carbon\Carbon;

class MyAttendanceHistoryExport implements FromCollection, WithHeadings, WithMapping
{
    /**
     * Ambil data absensi user yang lagi login
     */
    public function collection()
    {
        return Attendance::where('employee_id', Auth::user()->employee->id)
                         ->latest()
                         ->get();
    }
    
    /**
     * Header tabel Excel
     */
    public function headings(): array
    {
        return [
            'Tanggal', 
            'Jam Masuk', 
            'Jam Pulang', 
            'Status', 
            'Keterangan'
        ];
    }

    /**
     * Format tiap baris data
     */
    public function map($attendance): array
    {
        return [
            Carbon::parse($attendance->date)->isoFormat('dddd, D MMMM Y'),
            $attendance->time_in ?? '--:--',
            $attendance->time_out ?? '--:--',
            $attendance->status,
            $attendance->notes ?? '-',
        ];
    }
}
```

**🔐 Security Note:**
Class ini otomatis ambil data dari `Auth::user()->employee->id`, jadi karyawan cuma bisa download data mereka sendiri!

---

### 🛣️ Langkah 2: Tambahin Route & Method Controller

Buka `routes/web.php`:

```php
// routes/web.php
use App\Http\Controllers\AttendanceController;

Route::middleware('auth')->group(function () {
    // ... route karyawan lainnya ...
    
    // 🆕 ROUTE EXPORT RIWAYAT KARYAWAN
    Route::get('/attendance/export/excel', [AttendanceController::class, 'exportMyHistoryExcel'])
          ->name('attendance.export.excel');
          
    Route::get('/attendance/export/pdf', [AttendanceController::class, 'exportMyHistoryPdf'])
          ->name('attendance.export.pdf');
});
```

Sekarang buka `app/Http/Controllers/AttendanceController.php`:

```php
<?php

namespace App\Http\Controllers;

// ... use statements lainnya ...
use App\Exports\MyAttendanceHistoryExport; // 👈 TAMBAHKAN INI
use Maatwebsite\Excel\Facades\Excel;       // 👈 TAMBAHKAN INI

class AttendanceController extends Controller
{
    // ... method index, clockIn, clockOut, dll ga berubah ...

    /**
     * Export riwayat absensi ke Excel
     */
    public function exportMyHistoryExcel()
    {
        $userName = str_replace(' ', '-', Auth::user()->name);
        $fileName = 'riwayat-absensi-' . $userName . '.xlsx';
        
        return Excel::download(new MyAttendanceHistoryExport, $fileName);
    }

    /**
     * Export riwayat absensi ke PDF
     */
    public function exportMyHistoryPdf()
    {
        $userName = str_replace(' ', '-', Auth::user()->name);
        $fileName = 'riwayat-absensi-' . $userName . '.pdf';
        
        return Excel::download(
            new MyAttendanceHistoryExport, 
            $fileName, 
            \Maatwebsite\Excel\Excel::DOMPDF
        );
    }
}
```

**⚠️ Penting - Install Package PDF:**

Buat export PDF, kamu butuh package tambahan. Jalankan:

```bash
composer require barryvdh/laravel-dompdf
```

Wait sampe instalasi selesai ya! ☕

---

### 🎨 Langkah 3: Tambahin Tombol di View

Buka `resources/views/attendance/index.blade.php` dan tambahin tombol export:

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            📋 Riwayat Absensi Saya
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">

                    {{-- 🆕 TOMBOL-TOMBOL EXPORT --}}
                    <div class="flex justify-end space-x-2 mb-4">
                        {{-- Tombol Export Excel --}}
                        <a 
                            href="{{ route('attendance.export.excel') }}" 
                            class="inline-flex items-center px-4 py-2 bg-green-600 hover:bg-green-700 text-white text-sm font-medium rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-green-500"
                        >
                            <i class="fas fa-file-excel mr-2"></i> Export Excel
                        </a>
                        
                        {{-- Tombol Export PDF --}}
                        <a 
                            href="{{ route('attendance.export.pdf') }}" 
                            class="inline-flex items-center px-4 py-2 bg-red-600 hover:bg-red-700 text-white text-sm font-medium rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-red-500"
                        >
                            <i class="fas fa-file-pdf mr-2"></i> Export PDF
                        </a>
                    </div>

                    {{-- Tabel Riwayat Absensi (ga berubah) --}}
                    <div class="overflow-x-auto">
                        {{-- ... kode tabel existing ... --}}
                    </div>
                    
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

---

## ✅ Testing Export Karyawan (Versi Pertama)

1. **Logout dari akun admin**
2. **Login sebagai karyawan**
3. **Buka halaman "Riwayat Absensi"**
4. **Cek dua tombol baru** di kanan atas (hijau & merah)
5. **Klik "Export Excel"** → Download file `.xlsx`
6. **Klik "Export PDF"** → Download file `.pdf`
7. **Buka kedua file** dan cek isinya

> **💡 Note:** PDF versi ini masih plain. Kita bakal bikin lebih kece di langkah berikutnya!

---

## 🎨 Level Up: Percantik Tampilan PDF!

PDF hasil export masih terlalu polos? Mari kita bikin jadi profesional banget dengan template HTML custom! 💅

---

### 📄 Langkah 1: Bikin Template PDF yang Kece

Bikin file baru: `resources/views/attendance/history_pdf.blade.php`

```blade
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Riwayat Absensi - {{ $employee->nama_lengkap }}</title>
    <style>
        /* 🎨 Custom Styling untuk PDF */
        body {
            font-family: 'Helvetica', 'Arial', sans-serif;
            font-size: 12px;
            color: #333;
            line-height: 1.6;
        }
        
        .container {
            width: 100%;
            margin: 0 auto;
            padding: 20px;
        }
        
        /* Header Section */
        .header {
            text-align: center;
            margin-bottom: 30px;
            padding-bottom: 20px;
            border-bottom: 3px solid #4F46E5;
        }
        
        .header h1 {
            margin: 0 0 10px 0;
            font-size: 24px;
            color: #4F46E5;
        }
        
        .header p {
            margin: 5px 0;
            font-size: 14px;
            color: #666;
        }
        
        .employee-info {
            background: #F3F4F6;
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 20px;
        }
        
        .employee-info table {
            width: 100%;
        }
        
        .employee-info td {
            padding: 5px 10px;
        }
        
        .employee-info td:first-child {
            font-weight: bold;
            width: 150px;
        }
        
        /* Table Styling */
        .table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        
        .table thead {
            background-color: #4F46E5;
            color: white;
        }
        
        .table th, .table td {
            border: 1px solid #E5E7EB;
            padding: 10px;
            text-align: left;
        }
        
        .table th {
            font-weight: bold;
            font-size: 11px;
            text-transform: uppercase;
            letter-spacing: 0.5px;
        }
        
        .table tbody tr:nth-child(even) {
            background-color: #F9FAFB;
        }
        
        .table tbody tr:hover {
            background-color: #F3F4F6;
        }
        
        /* Status Badge */
        .status-badge {
            padding: 4px 8px;
            border-radius: 4px;
            font-size: 10px;
            font-weight: bold;
            display: inline-block;
        }
        
        .status-hadir {
            background-color: #D1FAE5;
            color: #065F46;
        }
        
        .status-terlambat {
            background-color: #FEF3C7;
            color: #92400E;
        }
        
        .status-izin {
            background-color: #DBEAFE;
            color: #1E40AF;
        }
        
        .status-sakit {
            background-color: #FED7AA;
            color: #9A3412;
        }
        
        .status-alpa {
            background-color: #FEE2E2;
            color: #991B1B;
        }
        
        /* Footer */
        .footer {
            margin-top: 40px;
            padding-top: 20px;
            border-top: 2px solid #E5E7EB;
            text-align: right;
            font-size: 10px;
            color: #6B7280;
        }
        
        /* No Data Message */
        .no-data {
            text-align: center;
            padding: 40px;
            color: #9CA3AF;
            font-style: italic;
        }
    </style>
</head>
<body>
    <div class="container">
        
        {{-- 📌 Header --}}
        <div class="header">
            <h1>📋 LAPORAN RIWAYAT ABSENSI</h1>
            <p>Sistem Absensi Karyawan</p>
        </div>

        {{-- 👤 Info Karyawan --}}
        <div class="employee-info">
            <table>
                <tr>
                    <td>Nama Lengkap</td>
                    <td>: {{ $employee->nama_lengkap }}</td>
                </tr>
                <tr>
                    <td>NIP</td>
                    <td>: {{ $employee->nip }}</td>
                </tr>
                <tr>
                    <td>Jabatan</td>
                    <td>: {{ $employee->position ?? '-' }}</td>
                </tr>
                <tr>
                    <td>Total Data</td>
                    <td>: {{ $attendances->count() }} record</td>
                </tr>
            </table>
        </div>

        {{-- 📊 Tabel Data Absensi --}}
        <table class="table">
            <thead>
                <tr>
                    <th style="width: 5%;">No</th>
                    <th style="width: 25%;">Tanggal</th>
                    <th style="width: 15%;">Jam Masuk</th>
                    <th style="width: 15%;">Jam Pulang</th>
                    <th style="width: 20%;">Status</th>
                    <th style="width: 20%;">Keterangan</th>
                </tr>
            </thead>
            <tbody>
                @forelse ($attendances as $attendance)
                    <tr>
                        <td style="text-align: center;">{{ $loop->iteration }}</td>
                        <td>{{ \Carbon\Carbon::parse($attendance->date)->isoFormat('dddd, D MMMM Y') }}</td>
                        <td>{{ $attendance->time_in ?? '--:--' }}</td>
                        <td>{{ $attendance->time_out ?? '--:--' }}</td>
                        <td>
                            <span class="status-badge 
                                @if(str_contains($attendance->status, 'Hadir')) status-hadir
                                @elseif(str_contains($attendance->status, 'Terlambat')) status-terlambat
                                @elseif($attendance->status == 'Izin') status-izin
                                @elseif($attendance->status == 'Sakit') status-sakit
                                @elseif($attendance->status == 'Alpa') status-alpa
                                @endif">
                                {{ $attendance->status }}
                            </span>
                        </td>
                        <td>{{ $attendance->notes ?? '-' }}</td>
                    </tr>
                @empty
                    <tr>
                        <td colspan="6" class="no-data">
                            Tidak ada data riwayat absensi.
                        </td>
                    </tr>
                @endforelse
            </tbody>
        </table>
        
        {{-- 📅 Footer --}}
        <div class="footer">
            <p>
                <strong>Dicetak pada:</strong> 
                {{ \Carbon\Carbon::now('Asia/Makassar')->isoFormat('dddd, D MMMM Y, HH:mm') }} WITA
            </p>
            <p>Dokumen ini dicetak secara otomatis oleh sistem.</p>
        </div>
        
    </div>
</body>
</html>
```

**🎨 Fitur Template:**
- Header dengan logo dan judul keren
- Info karyawan dalam box abu-abu
- Tabel dengan warna zebra-striping
- Badge berwarna untuk setiap status
- Footer dengan timestamp

---

### 🔄 Langkah 2: Update Class Export

Sekarang kita ubah `MyAttendanceHistoryExport` biar pake template Blade yang baru.

Buka `app/Exports/MyAttendanceHistoryExport.php` dan **ganti semua isinya**:

```php
<?php

namespace App\Exports;

use App\Models\Attendance;
use Illuminate\Contracts\View\View; // 👈 Import View
use Maatwebsite\Excel\Concerns\FromView; // 👈 Ganti jadi FromView
use Illuminate\Support\Facades\Auth;

class MyAttendanceHistoryExport implements FromView
{
    /**
     * Render view Blade dengan data yang diperlukan
     */
    public function view(): View
    {
        // Ambil data karyawan yang lagi login
        $employee = Auth::user()->employee;

        // Ambil semua riwayat absensi karyawan
        $attendances = Attendance::where('employee_id', $employee->id)
                                 ->latest()
                                 ->get();

        // Return view dengan data
        return view('attendance.history_pdf', [
            'attendances' => $attendances,
            'employee' => $employee
        ]);
    }
}
```

**🔑 Perubahan Penting:**
- ❌ Hapus: `FromCollection`, `WithHeadings`, `WithMapping`
- ✅ Ganti dengan: `FromView`
- ❌ Hapus method: `collection()`, `headings()`, `map()`
- ✅ Tambah method: `view()`

---

### ✅ Langkah 3: Testing Final!

**Controller ga perlu diubah!** Method `exportMyHistoryPdf()` otomatis pake template baru.

**Testing Steps:**

1. **Login sebagai karyawan**
2. **Buka "Riwayat Absensi"**
3. **Klik "Export PDF"** 🎯
4. **Buka file PDF yang ke-download**
5. **Cek perubahannya:**
   - ✅ Header berwarna biru kece
   - ✅ Info karyawan dalam box
   - ✅ Tabel dengan warna zebra
   - ✅ Badge status berwarna-warni
   - ✅ Footer dengan timestamp

---

## 🎉 DONE! Project Part 4 Complete!

Selamat bro! Kamu udah berhasil bikin fitur export yang super lengkap:

### ✅ Checklist Achievement:

- ✅ **Export Laporan Harian ke Excel** (Admin)
- ✅ **Laporan Bulanan dengan Rekapitulasi** (Admin)
- ✅ **Export Laporan Bulanan ke Excel** (Admin)
- ✅ **Export Riwayat ke Excel** (Karyawan)
- ✅ **Export Riwayat ke PDF dengan Template Kece** (Karyawan)

---

## 🚀 Next Level Tips

Mau bikin lebih keren lagi? Coba ini:

1. **📧 Email Report:** Kirim laporan otomatis via email
2. **📊 Chart/Grafik:** Tambahin chart di laporan bulanan
3. **🔍 Advanced Filter:** Filter by range date, department, dll
4. **🎨 Custom Template:** Buat template PDF sesuai branding company
5. **📱 WhatsApp Integration:** Notif via WA kalo ada laporan baru

---

## 💪 Troubleshooting

**Problem:** PDF ga ke-download atau error

**Solution:**
```bash
# Pastikan package dompdf udah keinstall
composer require barryvdh/laravel-dompdf

# Clear cache
php artisan config:clear
php artisan cache:clear
```

**Problem:** Export Excel format rusak

**Solution:**
- Cek versi `maatwebsite/excel` minimal 3.1
- Pastikan PHP extension `zip` enabled

---

## 📚 Resources

- [Laravel Excel Docs](https://docs.laravel-excel.com/)
- [DomPDF GitHub](https://github.com/barryvdh/laravel-dompdf)
- [Carbon Date Formatting](https://carbon.nesbot.com/docs/)

---

<div align="center">

### 🎓 Congratulations! 

**Part 4 Complete** ✨

Kamu udah jadi master export data! 🔥

---

**Made with ❤️ by Gen Z Developers**

*"Code is poetry, debugging is... well, also poetry but angrier."* 😅

</div>

### 🔧 Langkah 1: Bikin Class Export Bulanan

Jalankan command Artisan:

```bash
php artisan make:export MonthlyAttendanceExport
```

Buka `app/Exports/MonthlyAttendanceExport.php` dan edit jadi kayak gini:

```php
<?php

namespace App\Exports;

use App\Models\Attendance;
use Maatwebsite\Excel\Concerns\FromCollection;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use Carbon\Carbon;

class MonthlyAttendanceExport implements FromCollection, WithHeadings, WithMapping
{
    protected $employeeId;
    protected $month;
    protected $year;

    public function __construct(int $employeeId, int $month, int $year)
    {
        $this->employeeId = $employeeId;
        $this->month = $month;
        $this->year = $year;
    }

    public function collection()
    {
        return Attendance::with('employee')
                         ->where('employee_id', $this->employeeId)
                         ->whereMonth('date', $this->month)
                         ->whereYear('date', $this->year)
                         ->orderBy('date', 'asc')
                         ->get();
    }

    public function headings(): array
    {
        return [
            'Tanggal',
            'Jam Masuk',
            'Jam Pulang',
            'Status',
            'Keterangan',
        ];
    }

    public function map($attendance): array
    {
        return [
            Carbon::parse($attendance->date)->isoFormat('dddd, D MMMM Y'),
            $attendance->time_in,
            $attendance->time_out,
            $attendance->status,
            $attendance->notes,
        ];
    }
}
```

**💡 Bedanya dengan Daily Export:**
- Nerima 3 parameter: `employeeId`, `month`, `year`
- Filter pake `whereMonth()` dan `whereYear()`
- Format tanggal lebih detail (include hari)

---

### 🛣️ Langkah 2: Tambahin Route & Method

Buka `routes/web.php`:

```php
// routes/web.php

Route::middleware(['auth', 'role:admin'])->group(function () {
    // ... route admin lainnya ...
    
    // 🆕 ROUTE EXPORT LAPORAN BULANAN
    Route::get('/admin/reports/monthly/export', [ReportController::class, 'exportMonthlyExcel'])
            ->name('admin.reports.monthly.export');
});
```

Sekarang buka `app/Http/Controllers/Admin/ReportController.php`:

```php
<?php

namespace App\Http\Controllers\Admin;

// ... use statements lainnya ...
use App\Exports\MonthlyAttendanceExport; // 👈 TAMBAHKAN INI

class ReportController extends Controller
{
    // ... method lain ga berubah ...

    /**
     * Export laporan bulanan ke Excel
     */
    public function exportMonthlyExcel(Request $request)
    {
        $employeeId = $request->input('employee_id');
        $month = $request->input('month', Carbon::now()->month);
        $year = $request->input('year', Carbon::now()->year);

        // Validasi: pastikan employee_id ada
        if (!$employeeId) {
            return redirect()
                ->route('admin.reports.monthly')
                ->with('error', 'Silakan pilih karyawan terlebih dahulu.');
        }
        
        // Ambil data karyawan
        $employee = Employee::findOrFail($employeeId);
        
        // Bikin nama file yang informatif
        $monthName = Carbon::create()->month($month)->isoFormat('MMMM');
        $fileName = 'laporan-bulanan-' 
                  . str_replace(' ', '-', $employee->nama_lengkap) 
                  . '-' . $monthName 
                  . '-' . $year 
                  . '.xlsx';

        return Excel::download(
            new MonthlyAttendanceExport($employeeId, $month, $year), 
            $fileName
        );
    }
}
```

**🔍 Yang Terjadi:**
1. Validasi kalo `employee_id` kosong → redirect with error
2. Ambil data karyawan buat nama file
3. Generate nama file: `laporan-bulanan-John-Doe-Oktober-2025.xlsx`
4. Trigger download

---

### 🎨 Langkah 3: Tambahin Tombol Export

Buka `resources/views/admin/reports/monthly.blade.php` dan update bagian rekapitulasi:

```blade
{{-- Tampilkan hasil kalo ada karyawan yang dipilih --}}
@if ($selectedEmployeeId && !empty($recap))
    
    <div class="mb-6">
        {{-- Header dengan tombol export --}}
        <div class="flex justify-between items-center mb-4">
            <h3 class="text-lg font-semibold">
                📊 Rekapitulasi Bulan {{ \Carbon\Carbon::create()->month($selectedMonth)->isoFormat('MMMM') }} {{ $selectedYear }}
            </h3>
            
            {{-- 🆕 TOMBOL EXPORT BARU --}}
            <a 
                href="{{ route('admin.reports.monthly.export', [
                    'employee_id' => $selectedEmployeeId, 
                    'month' => $selectedMonth, 
                    'year' => $selectedYear
                ]) }}" 
                class="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-green-600 hover:bg-green-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-green-500"
            >
                <i class="fas fa-file-excel mr-2"></i> Export ke Excel
            </a>
        </div>
        
        {{-- Kartu rekapitulasi (ga berubah) --}}
        <div class="grid grid-cols-2 md:grid-cols-5 gap-4 text-center">
            {{-- ... kartu-kartu rekapitulasi ... --}}
        </div>
    </div>
    
    {{-- ... sisa kode ... --}}
@endif
```

**⚠️ Perhatiin Parameter:**
Tombol export passing 3 parameter ke route:
- `employee_id` → Karyawan mana yang mau di-export
- `month` → Bulan berapa
- `year` → Tahun berapa

---

## ✅ Testing Export Bulanan

1. **Login sebagai Admin**
2. **Buka Laporan Bulanan**
3. **Pilih karyawan, bulan, dan tahun**
4. **Klik "Tampilkan Laporan"**
5. **Klik tombol hijau "Export ke Excel"**
6. **File otomatis ke-download** dengan nama kayak:
   - `laporan-bulanan-Budi-Santoso-Oktober-2025.xlsx`
7. **Buka file** dan cek isinya!

---

## 📋 Overview Part 4

Yo, di part 4 ini kita bakal bikin fitur-fitur keren buat export data absensi:

- ✅ **Export Laporan Harian ke Excel** - Buat admin download data harian
- ✅ **Laporan Bulanan** - Rekap absensi per karyawan setiap bulan
- ✅ **Export Rekap Bulanan ke Excel** - Download rekap bulanan dalam format Excel
- ✅ **Export Riwayat Absensi Karyawan** - Karyawan bisa download data mereka sendiri (Excel & PDF)

Let's go! 🔥

---

## 📊 Tahap 13: Export Laporan Harian ke Excel

### 🎯 Tujuan

Kasih admin kemampuan buat download laporan absensi harian dalam format Excel (.xlsx) yang bisa langsung dibuka di Excel atau Google Sheets.

---

### 🛠️ Langkah 1: Install Package Laravel Excel

Pertama-tama, kita butuh package khusus buat handle export Excel di Laravel. Buka terminal kamu dan jalankan:

```bash
composer require maatwebsite/excel
```

> **💡 Pro Tip:** Composer bakal otomatis download dan setup package ini. Tinggal tunggu aja sampe selesai!

---

### 📝 Langkah 2: Bikin Class Export

Laravel Excel kerja pake sistem class yang rapi banget. Setiap jenis export punya class sendiri-sendiri. 

Jalankan command Artisan ini:

```bash
php artisan make:export DailyAttendanceExport
```

Nah, sekarang buka file yang baru dibuat di `app/Exports/DailyAttendanceExport.php` dan edit jadi kayak gini:

```php
<?php

namespace App\Exports;

use App\Models\Attendance;
use Maatwebsite\Excel\Concerns\FromCollection;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use Carbon\Carbon;

class DailyAttendanceExport implements FromCollection, WithHeadings, WithMapping
{
    protected $date;

    // Constructor buat nerima tanggal dari controller
    public function __construct(string $date)
    {
        $this->date = $date;
    }

    /**
     * Ambil data yang mau di-export
     * 
     * @return \Illuminate\Support\Collection
     */
    public function collection()
    {
        // Query data absensi sesuai tanggal yang dipilih
        return Attendance::with('employee')
                         ->whereDate('date', $this->date)
                         ->get();
    }

    /**
     * Bikin header buat file Excel
     * 
     * @return array
     */
    public function headings(): array
    {
        return [
            'NIP',
            'Nama Karyawan',
            'Tanggal',
            'Jam Masuk',
            'Jam Pulang',
            'Status',
            'Keterangan',
        ];
    }

    /**
     * Format tiap baris data yang mau ditampilin
     * 
     * @param mixed $attendance
     * @return array
     */
    public function map($attendance): array
    {
        $nip = $attendance->employee->nip ?? 'N/A';
        
        return [
            "'" . $nip,  // Pake single quote biar di Excel ga auto-format jadi angka
            $attendance->employee->nama_lengkap ?? 'Karyawan Tidak Ditemukan',
            Carbon::parse($attendance->date)->format('d-m-Y'),
            $attendance->time_in,
            $attendance->time_out,
            $attendance->status,
            $attendance->notes,
        ];
    }
}
```

**🔍 Penjelasan Kode:**
- `FromCollection` → Ambil data dari database
- `WithHeadings` → Kasih header di baris pertama Excel
- `WithMapping` → Format data sebelum masuk ke Excel
- `"'" . $nip` → Pake trick single quote biar NIP tetep jadi text, bukan angka

---

### 🛣️ Langkah 3: Tambahin Route Baru

Buka `routes/web.php` dan tambahin route ini di dalam grup admin:

```php
// routes/web.php

Route::middleware(['auth', 'role:admin'])->group(function () {
    // ... route admin lainnya ...
    
    Route::get('/admin/reports/daily', [ReportController::class, 'dailyReport'])
            ->name('admin.reports.daily');
            
    // 🆕 ROUTE BARU BUAT EXPORT EXCEL
    Route::get('/admin/reports/daily/export', [ReportController::class, 'exportExcel'])
            ->name('admin.reports.daily.export');
});
```

---

### 🎮 Langkah 4: Bikin Method di Controller

Sekarang kita tambahin method `exportExcel` di controller. Buka `app/Http/Controllers/Admin/ReportController.php`:

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use App\Models\Attendance;
use Carbon\Carbon;
use App\Exports\DailyAttendanceExport; // 👈 TAMBAHKAN INI
use Maatwebsite\Excel\Facades\Excel;   // 👈 TAMBAHKAN INI

class ReportController extends Controller
{
    public function dailyReport(Request $request)
    {
        // ... method ini ga berubah ...
    }

    /**
     * Handle request export ke Excel
     */
    public function exportExcel(Request $request)
    {
        // Ambil tanggal dari request, kalo ga ada pake hari ini
        $date = $request->input('date') 
                ? Carbon::parse($request->input('date'))->format('Y-m-d') 
                : Carbon::now('Asia/Makassar')->format('Y-m-d');

        // Bikin nama file yang dinamis
        $fileName = 'laporan-absensi-harian-' . $date . '.xlsx';

        // Execute export dan download filenya
        return Excel::download(new DailyAttendanceExport($date), $fileName);
    }
}
```

**💡 Yang Terjadi di Sini:**
1. Ambil parameter `date` dari URL
2. Kalo ga ada date, pake tanggal hari ini
3. Bikin nama file yang informatif (include tanggal)
4. Panggil class Export dan trigger download

---

### 🎨 Langkah 5: Tambahin Tombol Export di View

Terakhir, kita tambahin tombol hijau kece buat trigger export. Buka `resources/views/admin/reports/daily.blade.php`:

```blade
{{-- resources/views/admin/reports/daily.blade.php --}}

<div class="mb-6">
    <form method="GET" action="{{ route('admin.reports.daily') }}">
        <div class="flex items-center space-x-4">
            <!-- Input tanggal -->
            <div>
                <label for="date" class="block text-sm font-medium text-gray-700">
                    Pilih Tanggal:
                </label>
                <input 
                    type="date" 
                    name="date" 
                    id="date" 
                    value="{{ $selectedDate }}" 
                    class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
                >
            </div>
            
            <!-- Tombol-tombol action -->
            <div class="pt-5 flex items-center space-x-2">
                <!-- Tombol tampilkan laporan -->
                <button 
                    type="submit" 
                    class="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
                >
                    📊 Tampilkan Laporan
                </button>
                
                <!-- 🆕 TOMBOL EXPORT BARU -->
                <a 
                    href="{{ route('admin.reports.daily.export', ['date' => $selectedDate]) }}" 
                    class="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-green-600 hover:bg-green-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-green-500"
                >
                    <i class="fas fa-file-excel mr-2"></i> Export ke Excel
                </a>
            </div>
        </div>
    </form>
</div>
```

**⚠️ Penting Banget:**
Parameter `['date' => $selectedDate]` di link export itu krusial! Ini yang bikin controller tau tanggal mana yang mau di-export.

---

## ✅ Testing Time!

Saatnya coba fitur barunya:

1. **Login sebagai Admin** 🔐
2. **Buka halaman Laporan Harian** 📊
3. **Liat tombol hijau "Export ke Excel"** - Harusnya muncul di sebelah tombol "Tampilkan Laporan"
4. **Pilih tanggal** (misalnya hari ini)
5. **Klik "Tampilkan Laporan"** - Pastiin ada datanya
6. **Klik "Export ke Excel"** 🎯
7. **Browser auto-download file** dengan nama kayak `laporan-absensi-harian-2025-10-06.xlsx`
8. **Buka file pake Excel/Google Sheets** - Cek isinya udah bener!

---

**Congrats!** Tahap 13 kelar. Export harian udah jalan lancar!

---

## Tahap 14: Laporan Absensi Bulanan

### Tujuan

Bikin halaman admin buat liat performa kehadiran karyawan dalam satu bulan penuh. Lengkap dengan:
- Dropdown pilih karyawan
- Filter bulan & tahun
- Rekapitulasi total (hadir, terlambat, izin, sakit, alpa)
- Detail absensi per hari

---

### Langkah 1: Tambahin Method di ReportController

Buka `app/Http/Controllers/Admin/ReportController.php` dan tambahin use statement + method baru:

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use App\Models\Attendance;
use App\Models\Employee; // 👈 TAMBAHKAN INI
use Carbon\Carbon;
use App\Exports\DailyAttendanceExport;
use Maatwebsite\Excel\Facades\Excel;

class ReportController extends Controller
{
    // ... method dailyReport dan exportExcel ga berubah ...

    /**
     * Nampilin halaman laporan absensi bulanan
     */
    public function monthlyReport(Request $request)
    {
        // Ambil semua karyawan aktif buat dropdown filter
        $employees = Employee::where('status', 'aktif')
                             ->orderBy('nama_lengkap')
                             ->get();

        // Ambil input dari filter, kalo ga ada pake bulan/tahun sekarang
        $selectedEmployeeId = $request->input('employee_id');
        $selectedMonth = $request->input('month', Carbon::now()->month);
        $selectedYear = $request->input('year', Carbon::now()->year);

        // Bikin collection kosong dulu sebagai default
        $attendances = collect();
        $recap = [];

        // Kalo ada karyawan yang dipilih, baru kita proses datanya
        if ($selectedEmployeeId) {
            // Query absensi karyawan di bulan & tahun yang dipilih
            $attendances = Attendance::where('employee_id', $selectedEmployeeId)
                                     ->whereMonth('date', $selectedMonth)
                                     ->whereYear('date', $selectedYear)
                                     ->orderBy('date', 'asc')
                                     ->get();

            // Hitung rekapitulasi
            $recap = [
                'hadir' => $attendances->where('status', 'Hadir')->count(),
                'terlambat' => $attendances->where('status', 'like', '%Terlambat%')->count(),
                'izin' => $attendances->where('status', 'Izin')->count(),
                'sakit' => $attendances->where('status', 'Sakit')->count(),
                'alpa' => $attendances->where('status', 'Alpa')->count(),
            ];
        }

        return view('admin.reports.monthly', compact(
            'employees',
            'selectedEmployeeId',
            'selectedMonth',
            'selectedYear',
            'attendances',
            'recap'
        ));
    }
}
```

**Penjelasan Logic:**
- Kalo `employee_id` belom dipilih → tampilkan form kosong
- Kalo udah dipilih → query data absensi + hitung rekapitulasi
- Pake `whereMonth` & `whereYear` buat filter by periode

---

### Langkah 2: Bikin Route Baru

Buka `routes/web.php` dan tambahin route laporan bulanan:

```php
// routes/web.php

Route::middleware(['auth', 'role:admin'])->group(function () {
    // ... route admin lainnya ...
    
    // ROUTE BARU LAPORAN BULANAN
    Route::get('/admin/reports/monthly', [ReportController::class, 'monthlyReport'])
            ->name('admin.reports.monthly');
});
```

---

### Langkah 3: Bikin View Laporan Bulanan

Bikin file baru di `resources/views/admin/reports/monthly.blade.php`:

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            Laporan Absensi Bulanan
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">

                    {{-- 🔍 Form Filter --}}
                    <form method="GET" action="{{ route('admin.reports.monthly') }}" class="mb-6">
                        <div class="grid grid-cols-1 md:grid-cols-4 gap-4 items-end">
                            
                            {{-- Dropdown Karyawan --}}
                            <div>
                                <label for="employee_id" class="block text-sm font-medium text-gray-700">
                                    Pilih Karyawan:
                                </label>
                                <select 
                                    name="employee_id" 
                                    id="employee_id" 
                                    class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm rounded-md" 
                                    required
                                >
                                    <option value="">-- Pilih Karyawan --</option>
                                    @foreach($employees as $employee)
                                        <option value="{{ $employee->id }}" 
                                                {{ $selectedEmployeeId == $employee->id ? 'selected' : '' }}>
                                            {{ $employee->nama_lengkap }}
                                        </option>
                                    @endforeach
                                </select>
                            </div>
                            
                            {{-- Dropdown Bulan --}}
                            <div>
                                <label for="month" class="block text-sm font-medium text-gray-700">
                                    Bulan:
                                </label>
                                <select 
                                    name="month" 
                                    id="month" 
                                    class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm rounded-md"
                                >
                                    @for ($m = 1; $m <= 12; $m++)
                                        <option value="{{ $m }}" {{ $selectedMonth == $m ? 'selected' : '' }}>
                                            {{ \Carbon\Carbon::create()->month($m)->isoFormat('MMMM') }}
                                        </option>
                                    @endfor
                                </select>
                            </div>
                            
                            {{-- Input Tahun --}}
                            <div>
                                <label for="year" class="block text-sm font-medium text-gray-700">
                                    Tahun:
                                </label>
                                <input 
                                    type="number" 
                                    name="year" 
                                    id="year" 
                                    value="{{ $selectedYear }}" 
                                    class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                                >
                            </div>
                            
                            {{-- Tombol Submit --}}
                            <div>
                                <button 
                                    type="submit" 
                                    class="w-full inline-flex items-center justify-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
                                >
                                    Tampilkan Laporan
                                </button>
                            </div>
                        </div>
                    </form>

                    {{-- 📊 Tampilkan hasil kalo ada karyawan yang dipilih --}}
                    @if ($selectedEmployeeId && !empty($recap))
                        
                        {{-- Kartu Rekapitulasi --}}
                        <div class="mb-6">
                            <h3 class="text-lg font-semibold mb-4">
                                Rekapitulasi Bulan {{ \Carbon\Carbon::create()->month($selectedMonth)->isoFormat('MMMM') }} {{ $selectedYear }}
                            </h3>
                            
                            <div class="grid grid-cols-2 md:grid-cols-5 gap-4 text-center">
                                {{-- Hadir --}}
                                <div class="p-4 bg-green-100 rounded-lg">
                                    <p class="text-sm text-gray-600">Hadir</p>
                                    <p class="font-bold text-2xl text-green-700">{{ $recap['hadir'] }}</p>
                                </div>
                                
                                {{-- Terlambat --}}
                                <div class="p-4 bg-yellow-100 rounded-lg">
                                    <p class="text-sm text-gray-600">Terlambat</p>
                                    <p class="font-bold text-2xl text-yellow-700">{{ $recap['terlambat'] }}</p>
                                </div>
                                
                                {{-- Izin --}}
                                <div class="p-4 bg-blue-100 rounded-lg">
                                    <p class="text-sm text-gray-600">Izin</p>
                                    <p class="font-bold text-2xl text-blue-700">{{ $recap['izin'] }}</p>
                                </div>
                                
                                {{-- Sakit --}}
                                <div class="p-4 bg-orange-100 rounded-lg">
                                    <p class="text-sm text-gray-600">Sakit</p>
                                    <p class="font-bold text-2xl text-orange-700">{{ $recap['sakit'] }}</p>
                                </div>
                                
                                {{-- Alpa --}}
                                <div class="p-4 bg-red-100 rounded-lg">
                                    <p class="text-sm text-gray-600">Alpa</p>
                                    <p class="font-bold text-2xl text-red-700">{{ $recap['alpa'] }}</p>
                                </div>
                            </div>
                        </div>

                        {{-- 📋 Tabel Detail Absensi --}}
                        <div class="overflow-x-auto">
                            <table class="min-w-full divide-y divide-gray-200">
                                <thead class="bg-gray-50">
                                    <tr>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                            Tanggal
                                        </th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                            Jam Masuk
                                        </th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                            Jam Pulang
                                        </th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                            Status
                                        </th>
                                    </tr>
                                </thead>
                                <tbody class="bg-white divide-y divide-gray-200">
                                    @forelse ($attendances as $attendance)
                                        <tr>
                                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                                                {{ \Carbon\Carbon::parse($attendance->date)->isoFormat('dddd, D MMMM Y') }}
                                            </td>
                                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                                                {{ $attendance->time_in ?? '--:--' }}
                                            </td>
                                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                                                {{ $attendance->time_out ?? '--:--' }}
                                            </td>
                                            <td class="px-6 py-4 whitespace-nowrap">
                                                <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full 
                                                    @if(str_contains($attendance->status, 'Hadir')) 
                                                        bg-green-100 text-green-800 
                                                    @elseif(str_contains($attendance->status, 'Terlambat')) 
                                                        bg-yellow-100 text-yellow-800 
                                                    @elseif($attendance->status == 'Izin') 
                                                        bg-blue-100 text-blue-800
                                                    @elseif($attendance->status == 'Sakit') 
                                                        bg-orange-100 text-orange-800
                                                    @elseif($attendance->status == 'Alpa') 
                                                        bg-red-100 text-red-800
                                                    @endif">
                                                    {{ $attendance->status }}
                                                </span>
                                            </td>
                                        </tr>
                                    @empty
                                        <tr>
                                            <td colspan="4" class="px-6 py-4 text-center text-sm text-gray-500">
                                                Tidak ada data absensi pada periode ini.
                                            </td>
                                        </tr>
                                    @endforelse
                                </tbody>
                            </table>
                        </div>
                        
                    @else
                        {{-- Pesan kalo belom milih karyawan --}}
                        <div class="text-center py-12">
                            <p class="text-gray-500">
                                Silakan pilih karyawan dan periode untuk menampilkan laporan.
                            </p>
                        </div>
                    @endif

                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

**Fitur Keren di View Ini:**
- Form filter yang responsive (grid 4 kolom di desktop, 1 kolom di mobile)
- Kartu rekapitulasi dengan warna berbeda tiap status
- Badge warna di kolom status tabel
- Handling kalo belom ada data

---

### Langkah 4: Tambahin Link di Navigation

Buka `resources/views/layouts/navigation.blade.php` dan tambahin menu baru:

```blade
{{-- Link Laporan Harian --}}
<x-nav-link :href="route('admin.reports.daily')" :active="request()->routeIs('admin.reports.daily')">
    Laporan Harian
</x-nav-link>

{{-- LINK BARU LAPORAN BULANAN --}}
<x-nav-link :href="route('admin.reports.monthly')" :active="request()->routeIs('admin.reports.monthly')">
    Laporan Bulanan
</x-nav-link>

{{-- Link Hari Libur --}}
<x-nav-link :href="route('holidays.index')" :active="request()->routeIs('holidays.*')">
    Hari Libur
</x-nav-link>
```

---

## Testing Laporan Bulanan

1. **Login sebagai Admin**
2. **Klik menu "Laporan Bulanan"** di navigasi
3. **Pilih nama karyawan** dari dropdown
4. **Pilih bulan & tahun** yang diinginkan
5. **Klik "Tampilkan Laporan"**
6. **Cek hasilnya:**
   - Kartu rekapitulasi muncul di atas
   - Tabel detail absensi muncul di bawah
   - Warna badge sesuai status

---
