<div align="center">

<!-- <p> <b>Visitors Count 👁️</b> </p>
<img src="https://profile-counter.deno.dev/part4-panduan-absensi-project/count.svg" alt="Profile Counter Repo :: Visitor's Count" /> -->

</div>

# part4-panduan-absensi-project

pada part 4 ini kita akan membuat fitur export data harian, rekap bulanan absen 

Fitur ini akan menambahkan sebuah tombol di halaman Laporan Harian admin, yang saat diklik akan men-download rekap absensi dalam format file Excel (.xlsx).

## Tahap 13: Export Laporan ke Excel
🎯 Tujuan:
Mengintegrasikan package Laravel Excel untuk meng-export data dari Laporan Harian Admin.

Langkah 1: Install Package Laravel Excel
Langkah pertama adalah menambahkan pustaka maatwebsite/excel ke dalam proyek kita. Buka terminal dan jalankan perintah Composer berikut:

```Bash
composer require maatwebsite/excel
```
Composer akan secara otomatis mengunduh dan mengkonfigurasi package ini untuk Anda.

Langkah 2: Membuat Class Export
Laravel Excel bekerja dengan menggunakan class khusus untuk setiap jenis export, ini membuat kode kita sangat rapi. Mari kita buat satu class untuk mengekspor laporan harian.

Jalankan perintah Artisan ini di terminal:

```Bash
php artisan make:export DailyAttendanceExport
```
Perintah ini akan membuat file baru di app/Exports/DailyAttendanceExport.php. Buka file tersebut dan modifikasi isinya menjadi seperti ini:

```PHP
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

    // Gunakan constructor untuk menerima tanggal dari controller
    public function __construct(string $date)
    {
        $this->date = $date;
    }

    /**
    * @return \Illuminate\Support\Collection
    */
    public function collection()
    {
        // Query data yang akan diexport, sama seperti di controller laporan
        return Attendance::with('employee')
                         ->whereDate('date', $this->date)
                         ->get();
    }

    /**
     * Menentukan header untuk file Excel.
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
     * Memetakan data untuk setiap baris di Excel.
     *
     * @param mixed $attendance
     * @return array
     */
    public function map($attendance): array
    {
        $nip = $attendance->employee->nip ?? 'N/A';
        return [
            "'" . $nip,  // Prepend single quote untuk memaksa sebagai teks di Excel
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
Langkah 3: Menambahkan Rute untuk Download
Kita butuh satu rute baru yang akan memicu proses download. Buka routes/web.php dan tambahkan rute GET berikut di dalam grup rute admin.

```PHP
// routes/web.php

Route::middleware(['auth', 'role:admin'])->group(function () {
    // ... rute admin lainnya ...
    Route::get('/admin/reports/daily', [ReportController::class, 'dailyReport'])
            ->name('admin.reports.daily');
            
    // RUTE BARU UNTUK EXPORT EXCEL
    Route::get('/admin/reports/daily/export', [ReportController::class, 'exportExcel'])
            ->name('admin.reports.daily.export');
});
```
Langkah 4: Membuat Method di Controller
Sekarang, kita buat method exportExcel di ReportController yang akan memanggil class Export yang sudah kita buat.

Buka app/Http/Controllers/Admin/ReportController.php dan tambahkan use statement serta method baru berikut:

```PHP
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use App\Models\Attendance;
use Carbon\Carbon;
use App\Exports\DailyAttendanceExport; // <-- TAMBAHKAN INI
use Maatwebsite\Excel\Facades\Excel;   // <-- TAMBAHKAN INI

class ReportController extends Controller
{
    public function dailyReport(Request $request)
    {
        // ... (method ini tidak berubah) ...
    }

    /**
     * Menangani permintaan export ke Excel.
     */
    public function exportExcel(Request $request)
    {
        // Ambil tanggal dari request, jika tidak ada, gunakan hari ini
        $date = $request->input('date') ? Carbon::parse($request->input('date'))->format('Y-m-d') : Carbon::now('Asia/Makassar')->format('Y-m-d');

        // Buat nama file yang dinamis
        $fileName = 'laporan-absensi-harian-' . $date . '.xlsx';

        // Panggil class Export dan download filenya
        return Excel::download(new DailyAttendanceExport($date), $fileName);
    }
}
```
Langkah 5: Menambahkan Tombol Export di View
Langkah terakhir adalah menambahkan tombol "Export ke Excel" di halaman laporan harian agar admin bisa mengkliknya.

Buka resources/views/admin/reports/daily.blade.php dan tambahkan sebuah link (<a>) di sebelah tombol "Tampilkan Laporan".

```Blade
{{-- resources/views/admin/reports/daily.blade.php --}}
{{-- ... --}}
<div class="mb-6">
    <form method="GET" action="{{ route('admin.reports.daily') }}">
        <div class="flex items-center space-x-4">
            <div>
                <label for="date" class="block text-sm font-medium text-gray-700">Pilih Tanggal:</label>
                <input type="date" name="date" id="date" value="{{ $selectedDate }}" class="mt-1 block w-full ... rounded-md">
            </div>
            <div class="pt-5 flex items-center space-x-2">
                <button type="submit" class="inline-flex items-center px-4 py-2 ... bg-indigo-600 hover:bg-indigo-700">
                    Tampilkan Laporan
                </button>
                
                {{-- TOMBOL EXPORT BARU --}}
                <a href="{{ route('admin.reports.daily.export', ['date' => $selectedDate]) }}" class="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-green-600 hover:bg-green-700">
                    <i class="fas fa-file-excel mr-2"></i> Export ke Excel
                </a>
            </div>
        </div>
    </form>
</div>
{{-- ... --}}
```
Poin Kunci: Perhatikan bahwa href pada link export menyertakan parameter date (['date' => $selectedDate]). Ini sangat penting agar controller tahu data tanggal berapa yang harus diekspor.

## ✅ Uji Coba
Login sebagai Admin dan buka halaman Laporan Harian.

Anda akan melihat tombol hijau baru "Export ke Excel".

Pilih tanggal tertentu (misalnya hari ini) dan klik "Tampilkan Laporan" untuk memastikan ada datanya.

---

Fitur ini akan memberikan admin kemampuan untuk melihat performa kehadiran seorang karyawan dalam rentang satu bulan penuh, lengkap dengan total rekapitulasi (jumlah hadir, terlambat, alpa, dll).

## Tahap 14: Laporan Absensi Bulanan
🎯 Tujuan:
Membuat halaman admin untuk memilih karyawan dan periode (bulan/tahun), lalu menampilkan rekapitulasi dan rincian absensi bulanan karyawan tersebut.

Langkah 1: Menambahkan Method di ReportController
Kita akan menambahkan method baru monthlyReport di controller yang sudah ada, yaitu app/Http/Controllers/Admin/ReportController.php.

Buka file tersebut dan tambahkan use App\Models\Employee; di atas, lalu tambahkan method baru di bawah ini:

```PHP
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use App\Models\Attendance;
use App\Models\Employee; // <-- TAMBAHKAN INI
use Carbon\Carbon;
use App\Exports\DailyAttendanceExport;
use Maatwebsite\Excel\Facades\Excel;

class ReportController extends Controller
{
    // ... (method dailyReport dan exportExcel tidak berubah) ...

    /**
     * Menampilkan halaman laporan absensi bulanan.
     */
    public function monthlyReport(Request $request)
    {
        // Ambil daftar karyawan aktif untuk dropdown filter
        $employees = Employee::where('status', 'aktif')->orderBy('nama_lengkap')->get();

        // Ambil input dari filter, jika tidak ada, gunakan bulan dan tahun saat ini
        $selectedEmployeeId = $request->input('employee_id');
        $selectedMonth = $request->input('month', Carbon::now()->month);
        $selectedYear = $request->input('year', Carbon::now()->year);

        $attendances = collect(); // Buat collection kosong sebagai default
        $recap = [];

        // Jika seorang karyawan sudah dipilih, baru kita proses datanya
        if ($selectedEmployeeId) {
            // Query untuk mengambil semua data absensi karyawan di bulan dan tahun yang dipilih
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
Langkah 2: Menambahkan Rute Baru
Sekarang, kita buat rute untuk halaman laporan bulanan di routes/web.php.

```PHP
// routes/web.php
Route::middleware(['auth'])->group(function () { 
    // ... rute admin lainnya ...
    
    // RUTE BARU UNTUK LAPORAN BULANAN
    Route::get('/admin/reports/monthly', [ReportController::class, 'monthlyReport'])
            ->name('admin.reports.monthly');
});
```
Langkah 3: Membuat View Laporan Bulanan
Buat file view baru di resources/views/admin/reports/monthly.blade.php dan isi dengan kode berikut:

```Blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Laporan Absensi Bulanan') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">

                    {{-- Filter Form --}}
                    <form method="GET" action="{{ route('admin.reports.monthly') }}" class="mb-6">
                        <div class="grid grid-cols-1 md:grid-cols-4 gap-4 items-end">
                            <div>
                                <label for="employee_id" class="block text-sm font-medium text-gray-700">Pilih Karyawan:</label>
                                <select name="employee_id" id="employee_id" class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm rounded-md" required>
                                    <option value="">-- Semua Karyawan --</option>
                                    @foreach($employees as $employee)
                                        <option value="{{ $employee->id }}" {{ $selectedEmployeeId == $employee->id ? 'selected' : '' }}>
                                            {{ $employee->nama_lengkap }}
                                        </option>
                                    @endforeach
                                </select>
                            </div>
                            <div>
                                <label for="month" class="block text-sm font-medium text-gray-700">Bulan:</label>
                                <select name="month" id="month" class="mt-1 block w-full pl-3 pr-10 py-2 ... rounded-md">
                                    @for ($m = 1; $m <= 12; $m++)
                                        <option value="{{ $m }}" {{ $selectedMonth == $m ? 'selected' : '' }}>
                                            {{ \Carbon\Carbon::create()->month($m)->isoFormat('MMMM') }}
                                        </option>
                                    @endfor
                                </select>
                            </div>
                            <div>
                                <label for="year" class="block text-sm font-medium text-gray-700">Tahun:</label>
                                <input type="number" name="year" id="year" value="{{ $selectedYear }}" class="mt-1 block w-full ... rounded-md">
                            </div>
                            <div>
                                <button type="submit" class="w-full inline-flex items-center justify-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-indigo-600 hover:bg-indigo-700">
                                    Tampilkan Laporan
                                </button>
                            </div>
                        </div>
                    </form>

                    {{-- Tampilkan hasil hanya jika karyawan dipilih --}}
                    @if ($selectedEmployeeId && !empty($recap))
                        <div class="mb-6">
                            <h3 class="text-lg font-semibold">Rekapitulasi untuk Bulan {{ \Carbon\Carbon::create()->month($selectedMonth)->isoFormat('MMMM') }} {{ $selectedYear }}</h3>
                            <div class="grid grid-cols-2 md:grid-cols-5 gap-4 mt-2 text-center">
                                <div class="p-4 bg-green-100 rounded-lg"><p class="text-sm">Hadir</p><p class="font-bold text-2xl">{{ $recap['hadir'] }}</p></div>
                                <div class="p-4 bg-yellow-100 rounded-lg"><p class="text-sm">Terlambat</p><p class="font-bold text-2xl">{{ $recap['terlambat'] }}</p></div>
                                <div class="p-4 bg-blue-100 rounded-lg"><p class="text-sm">Izin</p><p class="font-bold text-2xl">{{ $recap['izin'] }}</p></div>
                                <div class="p-4 bg-orange-100 rounded-lg"><p class="text-sm">Sakit</p><p class="font-bold text-2xl">{{ $recap['sakit'] }}</p></div>
                                <div class="p-4 bg-red-100 rounded-lg"><p class="text-sm">Alpa</p><p class="font-bold text-2xl">{{ $recap['alpa'] }}</p></div>
                            </div>
                        </div>

                        <div class="overflow-x-auto">
                            <table class="min-w-full divide-y divide-gray-200">
                                <thead class="bg-gray-50">
                                    <tr>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Tanggal</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Jam Masuk</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Jam Pulang</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Status</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    @forelse ($attendances as $attendance)
                                        <tr class="bg-white">
                                            <td class="px-6 py-4">{{ \Carbon\Carbon::parse($attendance->date)->isoFormat('dddd, D MMMM Y') }}</td>
                                            <td class="px-6 py-4">{{ $attendance->time_in ?? '--:--' }}</td>
                                            <td class="px-6 py-4">{{ $attendance->time_out ?? '--:--' }}</td>
                                            <td class="px-6 py-4">
                                                 <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full 
                                                    @if(str_contains($attendance->status, 'Hadir')) bg-green-100 text-green-800 
                                                    @elseif(str_contains($attendance->status, 'Terlambat')) bg-yellow-100 text-yellow-800 
                                                    @elseif($attendance->status == 'Izin') bg-blue-100 text-blue-800
                                                    @elseif($attendance->status == 'Sakit') bg-orange-100 text-orange-800
                                                    @elseif($attendance->status == 'Alpa') bg-red-100 text-red-800
                                                    @endif">
                                                    {{ $attendance->status }}
                                                </span>
                                            </td>
                                        </tr>
                                    @empty
                                        <tr>
                                            <td colspan="4" class="px-6 py-4 text-center">Tidak ada data absensi pada periode ini.</td>
                                        </tr>
                                    @endforelse
                                </tbody>
                            </table>
                        </div>
                    @else
                        <p class="text-center text-gray-500">Silakan pilih karyawan dan periode untuk menampilkan laporan.</p>
                    @endif

                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```
Langkah 4: Menambahkan Link Navigasi
Terakhir, tambahkan link "Laporan Bulanan" di menu navigasi admin pada file resources/views/layouts/navigation.blade.php.

```HTML
<x-nav-link :href="route('admin.reports.daily')" :active="request()->routeIs('admin.reports.daily')">
    {{ __('Laporan Harian') }}
</x-nav-link>

{{-- LINK BARU UNTUK LAPORAN BULANAN --}}
<x-nav-link :href="route('admin.reports.monthly')" :active="request()->routeIs('admin.reports.monthly')">
    {{ __('Laporan Bulanan') }}
</x-nav-link>

<x-nav-link :href="route('holidays.index')" :active="request()->routeIs('holidays.*')">
    {{ __('Hari Libur') }}
</x-nav-link>
```
## ✅ Uji Coba
Login sebagai Admin.

Klik menu navigasi baru "Laporan Bulanan".

Pilih salah satu nama karyawan dari dropdown.

Pilih bulan dan tahun yang diinginkan.

Klik tombol "Tampilkan Laporan".

Sistem akan menampilkan kartu rekapitulasi di bagian atas dan tabel rincian absensi di bawahnya untuk karyawan dan periode yang Anda pilih.

Sekarang, klik tombol "Export ke Excel".

Browser Anda akan secara otomatis men-download sebuah file .xlsx dengan nama seperti laporan-absensi-harian-2025-10-06.xlsx.

Buka file tersebut dengan Microsoft Excel, Google Sheets, atau aplikasi sejenis. Pastikan datanya sesuai dengan yang ditampilkan di halaman web.

---

## 1. Export Excel Rekap Bulanan (Sisi Admin)
Tujuannya adalah menambahkan tombol "Export Excel" di halaman laporan bulanan yang akan men-download rekapitulasi dan rincian data yang sedang ditampilkan.

Langkah 1: Membuat Class Export Bulanan
Kita buat class Export baru yang akan menangani logika untuk laporan bulanan.

Jalankan perintah Artisan ini di terminal:

```Bash
php artisan make:export MonthlyAttendanceExport
```
Buka file app/Exports/MonthlyAttendanceExport.php dan modifikasi isinya. Kita akan menerima employeeId, month, dan year untuk mengambil data yang relevan.

```PHP
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
Langkah 2: Menambahkan Rute & Method Controller
Buka routes/web.php dan tambahkan rute untuk memicu download.

```PHP
// routes/web.php
Route::middleware(['auth'])->group(function () {
    // ... rute admin lainnya ...
    // RUTE BARU UNTUK EXPORT LAPORAN BULANAN
    Route::get('/admin/reports/monthly/export', [ReportController::class, 'exportMonthlyExcel'])
            ->name('admin.reports.monthly.export');
});
```
Selanjutnya, buka app/Http/Controllers/Admin/ReportController.php dan tambahkan use statement serta method exportMonthlyExcel.

```PHP
<?php
// app/Http/Controllers/Admin/ReportController.php
namespace App\Http\Controllers\Admin;

// ... use statements lainnya ...
use App\Exports\MonthlyAttendanceExport; // <-- TAMBAHKAN INI

class ReportController extends Controller
{
    // ... (method dailyReport & monthlyReport tidak berubah) ...
    // ... (method exportExcel tidak berubah) ...

    public function exportMonthlyExcel(Request $request)
    {
        $employeeId = $request->input('employee_id');
        $month = $request->input('month', Carbon::now()->month);
        $year = $request->input('year', Carbon::now()->year);

        // Validasi: pastikan employee_id ada
        if (!$employeeId) {
            return redirect()->route('admin.reports.monthly')->with('error', 'Silakan pilih karyawan terlebih dahulu.');
        }
        
        $employee = Employee::findOrFail($employeeId);
        $monthName = Carbon::create()->month($month)->isoFormat('MMMM');
        $fileName = 'laporan-bulanan-' . str_replace(' ', '-', $employee->nama_lengkap) . '-' . $monthName . '-' . $year . '.xlsx';

        return Excel::download(new MonthlyAttendanceExport($employeeId, $month, $year), $fileName);
    }
}
```
Langkah 3: Menambahkan Tombol di View
Buka resources/views/admin/reports/monthly.blade.php dan tambahkan tombol export. Tombol ini hanya akan muncul jika sebuah laporan sedang ditampilkan.

```Blade

{{-- ... di dalam file resources/views/admin/reports/monthly.blade.php --}}

{{-- Tampilkan hasil hanya jika karyawan dipilih --}}
@if ($selectedEmployeeId && !empty($recap))
    <div class="mb-6">
        <div class="flex justify-between items-center">
            <h3 class="text-lg font-semibold">
                Rekapitulasi untuk Bulan {{ \Carbon\Carbon::create()->month($selectedMonth)->isoFormat('MMMM') }} {{ $selectedYear }}
            </h3>
            {{-- TOMBOL EXPORT BARU --}}
            <a href="{{ route('admin.reports.monthly.export', ['employee_id' => $selectedEmployeeId, 'month' => $selectedMonth, 'year' => $selectedYear]) }}" class="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-green-600 hover:bg-green-700">
                <i class="fas fa-file-excel mr-2"></i> Export ke Excel
            </a>
        </div>
        {{-- ... sisa kode rekapitulasi ... --}}
    </div>
    {{-- ... sisa kode ... --}}
@endif
```


