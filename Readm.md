
# Panduan Migrasi ke TanStack Query

Dokumen ini berisi panduan untuk memigrasikan aplikasi dari penggunaan state lokal dan fetch langsung ke Supabase menjadi menggunakan TanStack Query (React Query).

## Perubahan yang Sudah Dilakukan

1. Instalasi paket TanStack Query:
   ```bash
   npm install @tanstack/react-query @tanstack/react-query-devtools
   ```

2. Setup QueryClient di `src/lib/queryClient.ts`

3. Menambahkan QueryClientProvider di `src/main.tsx`

4. Membuat custom hooks untuk data fetching:
   - `src/lib/hooks/usePayments.ts`
   - `src/lib/hooks/useExpenses.ts`

5. Membuat komponen contoh di `src/components/examples/QueryExample.tsx`

## Cara Migrasi Komponen

### 1. Migrasi AdminDashboard.tsx

Berikut adalah langkah-langkah untuk memigrasikan `AdminDashboard.tsx`:

1. Import hooks yang diperlukan:

```tsx
import { 
  usePayments, 
  useAddPayment, 
  useUpdatePayment, 
  useDeletePayment, 
  useChangePaymentStatus 
} from '../../lib/hooks/usePayments';

import { 
  useExpenses, 
  useAddExpense, 
  useUpdateExpense, 
  useDeleteExpense 
} from '../../lib/hooks/useExpenses';
```

2. Ganti state dan fungsi fetch dengan hooks TanStack Query:

```tsx
// Ganti ini:
const [payments, setPayments] = useState<Payment[]>([]);
const [expenses, setExpenses] = useState<Expense[]>([]);
const [loading, setLoading] = useState(true);

// Dengan ini:
const { 
  data: payments = [], 
  isLoading: isLoadingPayments,
  error: paymentsError
} = usePayments();

const { 
  data: expenses = [], 
  isLoading: isLoadingExpenses,
  error: expensesError
} = useExpenses();

// Gabungkan loading state
const loading = isLoadingPayments || isLoadingExpenses;

// Gabungkan error
useEffect(() => {
  if (paymentsError) {
    setError('Terjadi kesalahan saat mengambil data pembayaran. Silakan coba lagi.');
  } else if (expensesError) {
    setError('Terjadi kesalahan saat mengambil data pengeluaran. Silakan coba lagi.');
  }
}, [paymentsError, expensesError]);
```

3. Ganti fungsi untuk menambah/mengupdate/menghapus data:

```tsx
// Tambahkan hooks mutation
const addPaymentMutation = useAddPayment();
const updatePaymentMutation = useUpdatePayment();
const deletePaymentMutation = useDeletePayment();
const changeStatusMutation = useChangePaymentStatus();

const addExpenseMutation = useAddExpense();
const updateExpenseMutation = useUpdateExpense();
const deleteExpenseMutation = useDeleteExpense();

// Ganti fungsi handlePaymentSubmit
const handlePaymentSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  setError(null);
  
  try {
    // Cek duplikasi nama
    const isDuplicate = await checkDuplicateName(nama, editingPayment?.id);
    if (isDuplicate) {
      setError(`Nama "${nama}" sudah ada dalam daftar. Gunakan nama yang berbeda.`);
      return;
    }

    const paymentData = {
      nama,
      jumlah: parseFloat(jumlah),
      status: parseFloat(jumlah) > 0 ? 'lunas' : 'belum_lunas'
    };

    if (editingPayment) {
      updatePaymentMutation.mutate({
        id: editingPayment.id,
        ...paymentData
      }, {
        onSuccess: () => {
          setNama('');
          setJumlah('');
          setEditingPayment(null);
          setError(null);
        },
        onError: (error: any) => {
          console.error('Error:', error);
          setError(error.message || 'Terjadi kesalahan saat menyimpan pembayaran');
        }
      });
    } else {
      addPaymentMutation.mutate(paymentData, {
        onSuccess: () => {
          setNama('');
          setJumlah('');
          setError(null);
        },
        onError: (error: any) => {
          console.error('Error:', error);
          setError(error.message || 'Terjadi kesalahan saat menyimpan pembayaran');
        }
      });
    }
  } catch (error: any) {
    console.error('Error:', error);
    setError(error.message || 'Terjadi kesalahan saat menyimpan pembayaran');
  }
};

// Ganti fungsi handleExpenseSubmit
const handleExpenseSubmit = async (e: React.FormEvent) => {
  e.preventDefault();

  try {
    // Gunakan tanggal hari ini dalam format ISO
    const today = new Date().toISOString().split('T')[0];
    
    const expenseData = {
      jumlah: parseFloat(jumlah),
      keterangan,
      tanggal: today
    };

    if (editingExpense) {
      updateExpenseMutation.mutate({
        id: editingExpense.id,
        ...expenseData
      }, {
        onSuccess: () => {
          setJumlah('');
          setKeterangan('');
          setEditingExpense(null);
        },
        onError: (error) => {
          console.error('Error:', error);
          alert('Terjadi kesalahan saat menyimpan pengeluaran');
        }
      });
    } else {
      addExpenseMutation.mutate(expenseData, {
        onSuccess: () => {
          setJumlah('');
          setKeterangan('');
        },
        onError: (error) => {
          console.error('Error:', error);
          alert('Terjadi kesalahan saat menyimpan pengeluaran');
        }
      });
    }
  } catch (error) {
    console.error('Error:', error);
    alert('Terjadi kesalahan saat menyimpan pengeluaran');
  }
};

// Ganti fungsi handleDeletePayment
const handleDeletePayment = async (id: string) => {
  if (!confirm('Apakah Anda yakin ingin menghapus pembayaran ini?')) return;
  
  deletePaymentMutation.mutate(id, {
    onError: (error) => {
      console.error('Error:', error);
      alert('Terjadi kesalahan saat menghapus pembayaran');
    }
  });
};

// Ganti fungsi handleDeleteExpense
const handleDeleteExpense = async (id: string) => {
  if (!confirm('Apakah Anda yakin ingin menghapus pengeluaran ini?')) return;
  
  deleteExpenseMutation.mutate(id, {
    onError: (error) => {
      console.error('Error:', error);
      alert('Terjadi kesalahan saat menghapus pengeluaran');
    }
  });
};

// Ganti fungsi setStatusLunas
const setStatusLunas = async (payment: Payment) => {
  try {
    setProcessingId(payment.id);
    setError(null);
    
    console.log('Mengubah status menjadi Lunas:', payment.id);
    
    // Gunakan nilai jumlah yang sudah ada jika lebih dari 0, atau gunakan 10000 sebagai default
    const newAmount = payment.jumlah > 0 ? payment.jumlah : 10000;
    
    changeStatusMutation.mutate({
      id: payment.id,
      status: 'lunas',
      jumlah: newAmount
    }, {
      onSuccess: () => {
        // Tidak perlu melakukan apa-apa, data akan diperbarui otomatis
      },
      onError: (error) => {
        console.error('Error:', error);
        setError('Gagal mengubah status. Silakan coba lagi.');
      },
      onSettled: () => {
        setProcessingId(null);
      }
    });
  } catch (error) {
    console.error('Error:', error);
    setError('Gagal mengubah status. Silakan coba lagi.');
    setProcessingId(null);
  }
};

// Ganti fungsi setStatusBelumLunas
const setStatusBelumLunas = async (payment: Payment) => {
  try {
    setProcessingId(payment.id);
    setError(null);
    
    console.log('Mengubah status menjadi Belum Lunas:', payment.id);
    
    changeStatusMutation.mutate({
      id: payment.id,
      status: 'belum_lunas'
      // Tidak mengubah jumlah untuk menghindari constraint
    }, {
      onSuccess: () => {
        // Tidak perlu melakukan apa-apa, data akan diperbarui otomatis
      },
      onError: (error) => {
        console.error('Error:', error);
        setError('Gagal mengubah status. Silakan coba lagi.');
      },
      onSettled: () => {
        setProcessingId(null);
      }
    });
  } catch (error) {
    console.error('Error:', error);
    setError('Gagal mengubah status. Silakan coba lagi.');
    setProcessingId(null);
  }
};
```

4. Hapus fungsi `fetchData` karena tidak diperlukan lagi

### 2. Migrasi Table.tsx

Berikut adalah langkah-langkah untuk memigrasikan `Table.tsx`:

1. Import hooks yang diperlukan:

```tsx
import { usePayments } from '../lib/hooks/usePayments';
import { useExpenses } from '../lib/hooks/useExpenses';
```

2. Ganti state dan fungsi fetch dengan hooks TanStack Query:

```tsx
// Ganti ini:
const [payments, setPayments] = useState<Payment[]>([]);
const [expenses, setExpenses] = useState<Expense[]>([]);
const [loading, setLoading] = useState(true);

// Dengan ini:
const { 
  data: payments = [], 
  isLoading: isLoadingPayments,
} = usePayments();

const { 
  data: expenses = [], 
  isLoading: isLoadingExpenses,
} = useExpenses();

// Gabungkan loading state
const loading = isLoadingPayments || isLoadingExpenses;
```

3. Hapus fungsi `fetchData` dan kode untuk realtime subscriptions karena TanStack Query akan menangani refetching data

## Keuntungan Menggunakan TanStack Query

1. **Caching Otomatis**: Data di-cache dan diperbarui secara otomatis
2. **Refetching Otomatis**: Data di-refetch saat komponen di-mount ulang atau saat window mendapatkan fokus
3. **Penanganan Loading & Error**: State loading dan error ditangani secara otomatis
4. **Deduplication**: Request yang sama tidak akan dikirim berulang kali
5. **Pagination & Infinite Scrolling**: Dukungan bawaan untuk pagination dan infinite scrolling
6. **Prefetching**: Kemampuan untuk melakukan prefetch data
7. **Devtools**: Alat pengembangan yang memudahkan debugging

## Sumber Daya Tambahan

- [Dokumentasi TanStack Query](https://tanstack.com/query/latest/docs/react/overview)
- [Contoh Penggunaan](https://tanstack.com/query/latest/docs/react/examples/react/basic)
- [Video Tutorial](https://www.youtube.com/watch?v=novnyCaa7To) 
