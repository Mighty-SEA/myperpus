<template>
  <Head title="Manajemen Buku" />
  <AppLayout :breadcrumbs="breadcrumbs">
    <div id="page-content" class="flex h-full flex-1 flex-col gap-4 p-4">
      <div class="flex justify-between items-center mb-4">
        <h1 class="text-xl md:text-2xl font-bold">Manajemen Buku</h1>
        <div class="flex flex-wrap gap-1 sm:gap-2">
          <Link :href="route('books.recycle')">
            <Button type="button" variant="outline" size="sm" class="text-xs sm:text-sm">
              <span class="mr-1">🗑️</span>
              <span class="hidden sm:inline">Lihat</span> Sampah
            </Button>
          </Link>
          <Dialog>
            <DialogTrigger as-child>
              <Button type="button" size="sm" class="text-xs sm:text-sm">Import</Button>
            </DialogTrigger>
            <DialogContent>
              <DialogHeader>
                <DialogTitle>Import Buku</DialogTitle>
                <DialogDescription>Upload file Excel (.xlsx/.xls) sesuai template export.</DialogDescription>
              </DialogHeader>
              <form :action="route('books.import')" method="POST" enctype="multipart/form-data">
                <input type="hidden" name="_token" :value="csrf" />
                <input type="file" name="file" accept=".xlsx,.xls" required class="mb-4" />
                <DialogFooter>
                  <Button type="submit">Upload</Button>
                  <DialogClose as-child>
                    <Button type="button" variant="outline">Batal</Button>
                  </DialogClose>
                </DialogFooter>
              </form>
            </DialogContent>
          </Dialog>
          <Button type="button" size="sm" class="text-xs sm:text-sm" @click="exportBooks">Export</Button>
          <Link :href="route('books.create')">
            <Button size="sm" class="text-xs sm:text-sm">
              <BookPlus class="mr-1 h-3 w-3 sm:h-4 sm:w-4" />
              <span class="hidden sm:inline">Tambah</span> Buku
            </Button>
          </Link>
        </div>
      </div>
      
      <!-- Komponen Pencarian -->
      <div class="mb-4 relative">
        <div class="flex">
          <div class="relative flex-1">
            <Input 
              v-model="search"
              placeholder="Cari buku berdasarkan judul, penulis, penerbit, no.panggil atau kategori..."
              class="pl-10 pr-10"
            />
            <div class="absolute left-0 top-0 h-full flex items-center pl-3">
              <Search class="h-4 w-4 text-gray-400" />
            </div>
            <button 
              v-if="search"
              @click="clearSearch"
              class="absolute right-0 top-0 h-full flex items-center pr-3"
            >
              <X class="h-4 w-4 text-gray-400" />
            </button>
          </div>
        </div>
        <div v-if="props.books.total > 0" class="text-sm text-gray-500 mt-2">
          Menampilkan {{ props.books.total }} hasil pencarian
        </div>
      </div>
      
      <!-- Bulk Actions -->
      <div v-if="selectedBooks.length > 0" class="flex items-center justify-between mb-4 p-3 bg-background rounded-md border shadow-sm">
        <div class="flex items-center gap-3">
          <span class="font-medium text-sm">{{ selectedBooks.length }} buku dipilih</span>
          <Button v-if="!isAllBooksSelected" size="sm" variant="ghost" @click="selectAllBooks" :disabled="isBulkProcessing || isLoadingAllIds">
            <CheckCircle class="h-4 w-4 mr-1" />
            <span v-if="isLoadingAllIds">Loading...</span>
            <span v-else>Pilih Semua</span>
          </Button>
        </div>
        <div class="flex items-center gap-2">
          <!-- Bulk Update Dialog -->
          <Dialog>
            <DialogTrigger as-child>
              <Button size="sm" variant="secondary">
                Update Eksemplar
              </Button>
            </DialogTrigger>
            <DialogContent>
              <DialogHeader>
                <DialogTitle>Update Eksemplar Buku</DialogTitle>
                <DialogDescription>Set eksemplar untuk {{ selectedBooks.length }} buku yang dipilih.</DialogDescription>
              </DialogHeader>
              <form @submit.prevent="bulkUpdateEksemplar">
                <div class="grid gap-4 py-4">
                  <div class="grid gap-2">
                    <Label for="eksemplar">Ekselampar</Label>
                    <Input id="eksemplar" type="number" v-model="bulkEsemplar" min="0" required />
                  </div>
                </div>
                <DialogFooter>
                  <Button type="submit" :disabled="isBulkProcessing">
                    <LoaderCircle v-if="isBulkProcessing" class="mr-2 h-4 w-4 animate-spin" />
                    Update
                  </Button>
                  <DialogClose as-child>
                    <Button type="button" variant="outline">Batal</Button>
                  </DialogClose>
                </DialogFooter>
              </form>
            </DialogContent>
          </Dialog>
          
          <!-- Bulk Delete Button -->
          <Button 
            size="sm" 
            variant="destructive" 
            @click="confirmBulkDelete = true"
            :disabled="isBulkProcessing"
          >
            <Trash2 class="h-4 w-4 mr-1" />
            Hapus
          </Button>
          
          <!-- Batalkan Pilihan -->
          <Button 
            size="sm" 
            variant="outline" 
            @click="selectedBooks = []"
            :disabled="isBulkProcessing"
          >
            <X class="h-4 w-4 mr-1" />
            Batalkan
          </Button>
        </div>
      </div>
      
      <!-- Mobile scroll hint (only show on small screens) -->
      <div class="sm:hidden text-xs text-gray-500 italic mb-2 px-2">
        Geser ke kanan/kiri untuk melihat selengkapnya
      </div>
      
      <div id="book-table" ref="tableRef" class="rounded-md border overflow-x-auto relative">
        <!-- Loading overlay untuk tabel -->
        <div 
          v-if="isPageLoading" 
          class="absolute inset-0 bg-white/80 backdrop-blur-sm z-10 flex items-center justify-center"
        >
          <div class="flex flex-col items-center">
            <LoaderCircle class="h-8 w-8 text-blue-600 animate-spin" />
            <span class="text-sm text-gray-700 font-medium mt-2">Memuat...</span>
          </div>
        </div>
        
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead class="w-10">
                <input 
                  type="checkbox" 
                  @change="toggleSelectAll" 
                  :checked="isAllSelected" 
                  class="h-4 w-4 rounded border-gray-300"
                  :disabled="!props.books.data || props.books.data.length === 0"
                />
              </TableHead>
              <TableHead class="w-12">No</TableHead>
              <TableHead>Cover</TableHead>
              <TableHead>Judul</TableHead>
              <TableHead class="hidden sm:table-cell">Penulis</TableHead>
              <TableHead class="hidden md:table-cell">Penerbit</TableHead>
              <TableHead class="hidden md:table-cell">Tahun</TableHead>
              <TableHead class="hidden lg:table-cell">Kategori</TableHead>
              <TableHead>No. Panggil</TableHead>
              <TableHead>Asal Koleksi</TableHead>
              <TableHead>Kota Terbit</TableHead>
              <TableHead>Eksemplar</TableHead>
              <TableHead class="text-right">Aksi</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            <TableRow v-for="(book, idx) in props.books.data" :key="book.id">
              <TableCell>
                <input 
                  type="checkbox" 
                  :value="book.id" 
                  v-model="selectedBooks" 
                  class="h-4 w-4 rounded border-gray-300"
                />
              </TableCell>
              <TableCell>{{ idx + 1 + ((props.books.current_page - 1) * props.books.per_page) }}</TableCell>
              <TableCell>
                <img v-if="book.cover_url" :src="book.cover_url" alt="Cover" class="w-12 h-16 object-cover rounded shadow" />
                <span v-else class="text-gray-400 italic">Tidak ada</span>
              </TableCell>
              <TableCell>{{ book.judul }}</TableCell>
              <TableCell class="hidden sm:table-cell">{{ book.penulis }}</TableCell>
              <TableCell class="hidden md:table-cell">{{ book.penerbit }}</TableCell>
              <TableCell class="hidden md:table-cell">{{ book.tahun_terbit }}</TableCell>
              <TableCell class="hidden lg:table-cell">{{ book.kategori }}</TableCell>
              <TableCell>{{ book.no_panggil }}</TableCell>
              <TableCell>{{ book.asal_koleksi }}</TableCell>
              <TableCell>{{ book.kota_terbit }}</TableCell>
              <TableCell>{{ book.eksemplar }}</TableCell>
              <TableCell class="text-right">
                <div class="flex justify-end space-x-2">
                  <Link :href="route('books.edit', book.id)">
                    <Button variant="outline" size="icon">
                      <Edit class="h-4 w-4" />
                    </Button>
                  </Link>
                  <Button variant="destructive" size="icon" @click="deleteBook(book.id)">
                    <Trash2 class="h-4 w-4" />
                  </Button>
                </div>
              </TableCell>
            </TableRow>
            <TableRow v-if="!props.books.data || props.books.data.length === 0">
              <TableCell colspan="10" class="text-center py-6">Tidak ada data buku</TableCell>
            </TableRow>
          </TableBody>
        </Table>
      </div>
      
      <!-- Pagination -->
      <div v-if="props.books.last_page > 1" class="px-2 sm:px-0 overflow-x-auto py-2">
        <Pagination 
          :current-page="props.books.current_page"
          :total-pages="props.books.last_page"
          @page-change="changePage"
        />
      </div>
    </div>
  </AppLayout>
  
  <!-- Konfirmasi Bulk Delete Dialog -->
  <Dialog v-model:open="confirmBulkDelete">
    <DialogContent>
      <DialogHeader>
        <DialogTitle>Konfirmasi Hapus</DialogTitle>
        <DialogDescription>
          Anda yakin ingin menghapus {{ selectedBooks.length }} buku yang dipilih? Tindakan ini tidak dapat dibatalkan.
        </DialogDescription>
      </DialogHeader>
      <DialogFooter>
        <Button variant="destructive" @click="bulkDelete" :disabled="isBulkProcessing">
          <LoaderCircle v-if="isBulkProcessing" class="mr-2 h-4 w-4 animate-spin" />
          <Trash2 v-else class="mr-2 h-4 w-4" />
          Hapus
        </Button>
        <Button variant="outline" @click="confirmBulkDelete = false" :disabled="isBulkProcessing">Batal</Button>
      </DialogFooter>
    </DialogContent>
  </Dialog>
</template>

<script setup lang="ts">
import AppLayout from '@/layouts/AppLayout.vue';
import { type BreadcrumbItem } from '@/types';
import { Head, Link, router } from '@inertiajs/vue3';
import { Button } from '@/components/ui/button';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Edit, Trash2, BookPlus, X, LoaderCircle, CheckCircle, Search } from 'lucide-vue-next';
import Dialog from '@/components/ui/dialog/Dialog.vue';
import DialogTrigger from '@/components/ui/dialog/DialogTrigger.vue';
import DialogContent from '@/components/ui/dialog/DialogContent.vue';
import DialogHeader from '@/components/ui/dialog/DialogHeader.vue';
import DialogTitle from '@/components/ui/dialog/DialogTitle.vue';
import DialogDescription from '@/components/ui/dialog/DialogDescription.vue';
import DialogFooter from '@/components/ui/dialog/DialogFooter.vue';
import DialogClose from '@/components/ui/dialog/DialogClose.vue';
import Pagination from '@/components/Pagination.vue';
import { Label } from '@/components/ui/label';
import { Input } from '@/components/ui/input';
import { ref, computed, watch } from 'vue';
import axios from 'axios';
import { debounce } from 'lodash';

interface PaginatedBooks {
  data: any[];
  current_page: number;
  last_page: number;
  per_page: number;
  total: number;
  links: any[];
}

const props = defineProps<{ 
  books: PaginatedBooks,
  filters: {
    search: string;
  }
}>();

const isPageLoading = ref(false);
const tableRef = ref<HTMLElement | null>(null);
const search = ref('');

// Mengisi nilai awal pencarian dari filter
search.value = props.filters.search || '';

// Debounce pencarian
const debouncedSearch = debounce(() => {
  router.get(route('books.index'), { search: search.value }, {
    preserveState: true,
    preserveScroll: true,
    replace: true
  });
}, 500);

// Watch perubahan pada search
watch(search, () => {
  debouncedSearch();
});

// Fungsi untuk menghapus pencarian
const clearSearch = () => {
  search.value = '';
  router.get(route('books.index'), {}, {
    preserveState: true,
    preserveScroll: true,
    replace: true
  });
};

const breadcrumbs: BreadcrumbItem[] = [
  {
    title: 'Dashboard',
    href: route('dashboard'),
  },
  {
    title: 'Manajemen Buku',
    href: route('books.index'),
  },
];

const deleteBook = (id: number) => {
  if (confirm('Yakin ingin menghapus buku ini?')) {
    router.delete(route('books.destroy', id));
  }
};

let csrf = '';
const meta = document.querySelector('meta[name="csrf-token"]');
if (meta) {
  csrf = meta.getAttribute('content') || '';
}

function exportBooks() {
  window.location.href = "/admin/export-books";
}

// Fungsi untuk pindah halaman
function changePage(page: number) {
  isPageLoading.value = true;
  
  router.get(route('books.index'), { page, search: search.value }, {
    preserveState: true,
    preserveScroll: true,
    onFinish: () => {
      isPageLoading.value = false;
    },
    onError: () => {
      isPageLoading.value = false;
      // Jika terjadi error, tampilkan pesan
      alert('Terjadi kesalahan saat memuat halaman. Silakan coba lagi.');
    }
  });
}

// Bulk actions state
const selectedBooks = ref<number[]>([]);
const confirmBulkDelete = ref(false);
const bulkEsemplar = ref<number>(1);
const isBulkProcessing = ref(false);
const isLoadingAllIds = ref(false);
const allBookIds = ref<number[]>([]);
const isAllBooksSelected = computed(() => {
  return allBookIds.value.length > 0 && 
         allBookIds.value.length === selectedBooks.value.length && 
         allBookIds.value.every(id => selectedBooks.value.includes(id));
});

// Check if all books in current page are selected
const isAllSelected = computed(() => {
  if (!props.books.data || props.books.data.length === 0) return false;
  return props.books.data.every(book => selectedBooks.value.includes(book.id));
});

// Toggle select all books in current page
function toggleSelectAll(event: Event) {
  const target = event.target as HTMLInputElement;
  if (target.checked) {
    selectedBooks.value = props.books.data.map(book => book.id);
  } else {
    selectedBooks.value = [];
  }
}

// Ambil semua ID buku
function selectAllBooks() {
  if (isLoadingAllIds.value) return;
  
  if (allBookIds.value.length > 0) {
    // Jika ID sudah tersedia, langsung pilih semua
    selectedBooks.value = [...allBookIds.value];
  } else {
    // Ambil semua ID dari server menggunakan URL absolut
    isLoadingAllIds.value = true;
    fetch('/api/books/all-ids')
      .then(response => response.json())
      .then(data => {
        allBookIds.value = data.ids || [];
        selectedBooks.value = [...allBookIds.value];
        isLoadingAllIds.value = false;
        
        // Jika data dibatasi, tampilkan pesan
        if (data.limited) {
          alert(`Hanya ${data.ids.length} dari total ${data.total} buku yang dipilih. Sistem membatasi jumlah maksimal yang dapat dipilih untuk kinerja yang lebih baik.`);
        }
      })
      .catch(error => {
        console.error('Error fetching all book IDs:', error);
        isLoadingAllIds.value = false;
        alert('Gagal mengambil semua ID buku. Silakan coba lagi.');
      });
  }
}

// Bulk delete books
function bulkDelete() {
  if (selectedBooks.value.length === 0) return;
  
  isBulkProcessing.value = true;
  
  // Menggunakan URL absolut dengan batasan limit waktu (timeout)
  axios({
    method: 'delete',
    url: '/admin/books/bulk-delete',
    data: {
      ids: selectedBooks.value
    },
    headers: {
      'X-CSRF-TOKEN': csrf,
      'Content-Type': 'application/json',
      'Accept': 'application/json'
    },
    timeout: 30000 // 30 detik timeout
  })
  .then(response => {
    selectedBooks.value = [];
    confirmBulkDelete.value = false;
    // Tampilkan notifikasi sukses
    alert(response.data.message || 'Buku berhasil dihapus.');
    // Refresh halaman setelah operasi berhasil
    window.location.reload();
  })
  .catch(error => {
    console.error('Error deleting books:', error);
    console.log('Error details:', error.response || error);
    alert('Gagal menghapus buku. Silakan coba lagi. ' + 
          (error.response?.data?.message || 
          (error.code === 'ECONNABORTED' ? 'Waktu request habis, server mungkin sibuk.' : '')));
  })
  .finally(() => {
    isBulkProcessing.value = false;
  });
}

// Bulk update eksemplar
function bulkUpdateEksemplar() {
  if (selectedBooks.value.length === 0) return;
  
  isBulkProcessing.value = true;
  
  axios.post('/admin/books/bulk-update-eksemplar', {
    ids: selectedBooks.value,
    eksemplar: bulkEsemplar.value
  }, {
    headers: {
      'X-CSRF-TOKEN': csrf
    },
    timeout: 30000 // 30 detik timeout
  })
  .then(() => {
    selectedBooks.value = [];
    // Refresh halaman setelah operasi berhasil
    window.location.reload();
  })
  .catch(error => {
    console.error('Error updating book quantity:', error);
    alert('Gagal mengupdate eksemplar buku. Silakan coba lagi. ' + 
          (error.code === 'ECONNABORTED' ? 'Waktu request habis, server mungkin sibuk.' : ''));
  })
  .finally(() => {
    isBulkProcessing.value = false;
  });
}
</script>

<style scoped>
/* Hapus kode yang tidak digunakan */
</style> 