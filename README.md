<div align="center">

<!-- <p> <b>Visitors Count 👁️</b> </p>
<img src="https://profile-counter.deno.dev/part4-panduan-absensi-project/count.svg" alt="Profile Counter Repo :: Visitor's Count" /> -->

</div>

# part4-panduan-absensi-project

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

Sekarang, klik tombol "Export ke Excel".

Browser Anda akan secara otomatis men-download sebuah file .xlsx dengan nama seperti laporan-absensi-harian-2025-10-06.xlsx.

Buka file tersebut dengan Microsoft Excel, Google Sheets, atau aplikasi sejenis. Pastikan datanya sesuai dengan yang ditampilkan di halaman web.
