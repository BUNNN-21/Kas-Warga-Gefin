  // Kas-Warga-gefin
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Kas RT 005 / RW 021 - Perum. Gesya Infinity Nongsa</title>
  
  <!-- PWA & Meta Tags -->
  <meta name="theme-color" content="#2563eb">
  <meta name="description" content="Aplikasi Kas Warga RT 005 / RW 021 Perum. Gesya Infinity Nongsa">
  <link rel="apple-touch-icon" href="https://cdn-icons-png.flaticon.com/512/6082/6082802.png">

  <!-- Tailwind CSS CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  
  <!-- React & Lucide Icons -->
  <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
  <script src="https://unpkg.com/lucide@latest"></script>

  <!-- Firebase SDK -->
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
</head>
<body class="bg-slate-100 text-slate-800 font-sans pb-24">

  <div id="root"></div>

  <script type="text/babel">
    // CONFIGURATION FIREBASE (Ganti dengan config milik Anda dari Firebase Console)
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "kas-gesya-rt005.firebaseapp.com",
      projectId: "kas-gesya-rt005",
      storageBucket: "kas-gesya-rt005.appspot.com",
      messagingSenderId: "1234567890",
      appId: "1:1234567890:web:abcdef123456"
    };

    // Initialize Firebase
    if (!firebase.apps.length) {
      firebase.initializeApp(firebaseConfig);
    }
    const db = firebase.firestore();

    const { useState, useEffect } = React;

    function App() {
      const [tab, setTab] = useState('ringkasan');
      const [isBendahara, setIsBendahara] = useState(false);
      const [pinInput, setPinInput] = useState('');
      const [showPinModal, setShowPinModal] = useState(false);
      
      const [wargaList, setWargaList] = useState([]);
      const [transaksi, setTransaksi] = useState([]);
      
      const [nominalIuran, setNominalIuran] = useState(50000);
      const [namaWargaBaru, setNamaWargaBaru] = useState('');
      const [blokBaru, setBlokBaru] = useState('');

      const [pengeluaranKet, setPengeluaranKet] = useState('');
      const [pengeluaranNom, setPengeluaranNom] = useState('');

      // Fetch Realtime Data from Firebase
      useEffect(() => {
        const unsubWarga = db.collection('warga').onSnapshot(snapshot => {
          const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
          setWargaList(data.sort((a, b) => a.blok.localeCompare(b.blok)));
        }, err => console.log("Firebase not connected yet, using offline mode"));

        const unsubTx = db.collection('transaksi').orderBy('tanggal', 'desc').onSnapshot(snapshot => {
          const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
          setTransaksi(data);
        }, err => console.log("Firebase not connected yet"));

        return () => {
          unsubWarga();
          unsubTx();
        };
      }, []);

      useEffect(() => {
        if (window.lucide) window.lucide.createIcons();
      });

      const totalPemasukan = transaksi.filter(t => t.tipe === 'masuk').reduce((acc, t) => acc + Number(t.nominal), 0);
      const totalPengeluaran = transaksi.filter(t => t.tipe === 'keluar').reduce((acc, t) => acc + Number(t.nominal), 0);
      const saldoKas = totalPemasukan - totalPengeluaran;

      const handleLoginBendahara = (e) => {
        e.preventDefault();
        if (pinInput === '1234') { // PIN standar Bendahara: 1234
          setIsBendahara(true);
          setShowPinModal(false);
          setPinInput('');
          alert('Login Bendahara Berhasil!');
        } else {
          alert('PIN Salah! Masukkan PIN yang benar.');
        }
      };

      const handleBayarIuran = async (warga) => {
        if (!isBendahara) return alert('Hanya Bendahara yang dapat mengutip iuran.');
        
        const bulanIni = new Date().toLocaleString('id-ID', { month: 'long', year: 'numeric' });
        
        try {
          await db.collection('warga').doc(warga.id).update({
            statusBayar: true,
            terakhirBayar: bulanIni
          });

          await db.collection('transaksi').add({
            keterangan: `Iuran Bulan Ini - ${warga.nama} (${warga.blok})`,
            nominal: nominalIuran,
            tipe: 'masuk',
            tanggal: new Date().toISOString()
          });

          alert(`Pembayaran atas nama ${warga.nama} berhasil dicatat!`);
        } catch (e) {
          alert('Gagal update database. Pastikan Firebase Config sudah diisi.');
        }
      };

      const handleTambahWarga = async (e) => {
        e.preventDefault();
        if (!namaWargaBaru || !blokBaru) return;
        try {
          await db.collection('warga').add({
            nama: namaWargaBaru,
            blok: blokBaru,
            statusBayar: false,
            terakhirBayar: '-'
          });
          setNamaWargaBaru('');
          setBlokBaru('');
          alert('Data warga berhasil ditambahkan!');
        } catch (e) {
          alert('Gagal menambahkan warga.');
        }
      };

      const handleTambahPengeluaran = async (e) => {
        e.preventDefault();
        if (!pengeluaranKet || !pengeluaranNom) return;
        try {
          await db.collection('transaksi').add({
            keterangan: pengeluaranKet,
            nominal: Number(pengeluaranNom),
            tipe: 'keluar',
            tanggal: new Date().toISOString()
          });
          setPengeluaranKet('');
          setPengeluaranNom('');
          alert('Catatan pengeluaran berhasil disimpan!');
        } catch (e) {
          alert('Gagal mencatat pengeluaran.');
        }
      };

      return (
        <div className="max-w-md mx-auto min-h-screen bg-slate-50 shadow-xl relative">
          
          <!-- Header -->
          <header className="bg-blue-600 text-white p-5 rounded-b-2xl shadow-lg">
            <div className="flex justify-between items-start">
              <div>
                <span className="text-xs font-semibold bg-blue-700 px-2 py-1 rounded-full border border-blue-400">
                  Perum. Gesya Infinity Nongsa
                </span>
                <h1 className="text-2xl font-bold mt-1">Kas RT 005 / RW 021</h1>
              </div>
              <button 
                onClick={() => isBendahara ? setIsBendahara(false) : setShowPinModal(true)}
                className={`text-xs px-3 py-1.5 rounded-lg font-bold transition ${isBendahara ? 'bg-amber-400 text-slate-900' : 'bg-blue-800 text-white hover:bg-blue-700'}`}
              >
                {isBendahara ? 'Mode: Bendahara' : 'Masuk Bendahara'}
              </button>
            </div>

            <!-- Saldo Card -->
            <div className="mt-5 bg-white/10 backdrop-blur-md p-4 rounded-xl border border-white/20">
              <p className="text-xs text-blue-100">Total Saldo Kas RT saat ini:</p>
              <p className="text-3xl font-black mt-1">Rp {saldoKas.toLocaleString('id-ID')}</p>
              <div className="flex justify-between mt-3 text-xs pt-2 border-t border-white/20">
                <span>Pemasukan: <b className="text-green-300">Rp {totalPemasukan.toLocaleString('id-ID')}</b></span>
                <span>Pengeluaran: <b className="text-red-300">Rp {totalPengeluaran.toLocaleString('id-ID')}</b></span>
              </div>
            </div>
          </header>

          <!-- Content Body -->
          <main className="p-4">
            
            {/* TAB RINGKASAN */}
            {tab === 'ringkasan' && (
              <div className="space-y-4">
                <div className="bg-white p-4 rounded-xl shadow-sm border border-slate-200">
                  <h3 className="font-bold text-slate-700 mb-2 flex items-center gap-2">
                    <i data-lucide="info" className="w-4 h-4 text-blue-600"></i> Informasi Iuran
                  </h3>
                  <p className="text-sm text-slate-600">Iuran Warga Bulanan: <b>Rp {nominalIuran.toLocaleString('id-ID')} / KK</b></p>
                  <p className="text-xs text-slate-500 mt-1">Pengutipan dilakukan rutin oleh pengurus RT 005 setiap awal bulan.</p>
                </div>

                <div className="bg-white p-4 rounded-xl shadow-sm border border-slate-200">
                  <h3 className="font-bold text-slate-700 mb-3">Riwayat Transaksi Terakhir</h3>
                  <div className="space-y-2 max-h-60 overflow-y-auto">
                    {transaksi.length === 0 ? (
                      <p className="text-xs text-slate-400 text-center py-4">Belum ada riwayat transaksi</p>
                    ) : (
                      transaksi.map((t, idx) => (
                        <div key={idx} className="flex justify-between items-center text-xs p-2.5 bg-slate-50 rounded-lg border border-slate-100">
                          <div>
                            <p className="font-semibold text-slate-800">{t.keterangan}</p>
                            <p className="text-[10px] text-slate-400">{new Date(t.tanggal).toLocaleDateString('id-ID')}</p>
                          </div>
                          <span className={`font-bold ${t.tipe === 'masuk' ? 'text-emerald-600' : 'text-rose-600'}`}>
                            {t.tipe === 'masuk' ? '+' : '-'} Rp {Number(t.nominal).toLocaleString('id-ID')}
                          </span>
                        </div>
                      ))
                    )}
                  </div>
                </div>
              </div>
            )}

            {/* TAB PENGUTIPAN / STATUS WARGA */}
            {tab === 'kutip' && (
              <div className="space-y-3">
                <div className="flex justify-between items-center">
                  <h2 className="font-bold text-slate-700">Daftar Iuran Warga</h2>
                  <span className="text-xs bg-slate-200 px-2 py-1 rounded text-slate-600 font-medium">
                    Total: {wargaList.length} KK
                  </span>
                </div>

                {wargaList.length === 0 ? (
                  <div className="bg-white p-6 text-center rounded-xl border border-dashed border-slate-300">
                    <p className="text-xs text-slate-500">Belum ada data warga terdaftar.</p>
                  </div>
                ) : (
                  wargaList.map((w) => (
                    <div key={w.id} className="bg-white p-3.5 rounded-xl border border-slate-200 shadow-sm flex items-center justify-between">
                      <div>
                        <div className="flex items-center gap-2">
                          <span className="font-bold text-sm text-slate-800">{w.nama}</span>
                          <span className="text-[10px] bg-slate-100 px-1.5 py-0.5 rounded text-slate-600 font-mono">Blok {w.blok}</span>
                        </div>
                        <p className="text-[11px] text-slate-400 mt-0.5">Ket: {w.terakhirBayar || 'Belum Lunas'}</p>
                      </div>

                      {w.statusBayar ? (
                        <span className="text-xs bg-emerald-100 text-emerald-700 px-2.5 py-1 rounded-full font-bold">
                          ✓ Lunas
                        </span>
                      ) : (
                        <button
                          onClick={() => handleBayarIuran(w)}
                          disabled={!isBendahara}
                          className={`text-xs px-3 py-1.5 rounded-lg font-bold ${isBendahara ? 'bg-blue-600 hover:bg-blue-700 text-white shadow-sm' : 'bg-slate-200 text-slate-400 cursor-not-allowed'}`}
                        >
                          {isBendahara ? 'Kutip' : 'Belum'}
                        </button>
                      )}
                    </div>
                  ))
                )}
              </div>
            )}

            {/* TAB INPUT BENDAHARA */}
            {tab === 'admin' && (
              <div className="space-y-4">
                {!isBendahara ? (
                  <div className="bg-amber-50 border border-amber-200 p-4 rounded-xl text-center">
                    <p className="text-xs text-amber-800 font-medium">Anda harus masuk sebagai Bendahara untuk mengakses halaman ini.</p>
                    <button 
                      onClick={() => setShowPinModal(true)}
                      className="mt-3 text-xs bg-amber-500 text-white font-bold px-4 py-2 rounded-lg"
                    >
                      Masukkan PIN Bendahara
                    </button>
                  </div>
                ) : (
                  <>
                    <!-- Form Pengeluaran -->
                    <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm">
                      <h3 className="font-bold text-sm text-slate-800 mb-3">Catat Pengeluaran Kas</h3>
                      <form onSubmit={handleTambahPengeluaran} className="space-y-2.5">
                        <input 
                          type="text" 
                          placeholder="Keterangan (misal: Beli Lampu Jalan)" 
                          value={pengeluaranKet} 
                          onChange={e => setPengeluaranKet(e.target.value)}
                          className="w-full text-xs p-2.5 border rounded-lg focus:outline-blue-500"
                          required
                        />
                        <input 
                          type="number" 
                          placeholder="Nominal (Rp)" 
                          value={pengeluaranNom} 
                          onChange={e => setPengeluaranNom(e.target.value)}
                          className="w-full text-xs p-2.5 border rounded-lg focus:outline-blue-500"
                          required
                        />
                        <button type="submit" className="w-full bg-rose-600 text-white text-xs font-bold py-2.5 rounded-lg">
                          Simpan Pengeluaran
                        </button>
                      </form>
                    </div>

                    <!-- Form Tambah Warga -->
                    <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm">
                      <h3 className="font-bold text-sm text-slate-800 mb-3">Tambah Data Warga Baru</h3>
                      <form onSubmit={handleTambahWarga} className="space-y-2.5">
                        <input 
                          type="text" 
                          placeholder="Nama Kepala Keluarga" 
                          value={namaWargaBaru} 
                          onChange={e => setNamaWargaBaru(e.target.value)}
                          className="w-full text-xs p-2.5 border rounded-lg focus:outline-blue-500"
                          required
                        />
                        <input 
                          type="text" 
                          placeholder="Nomor Rumah / Blok (contoh: A1 No. 05)" 
                          value={blokBaru} 
                          onChange={e => setBlokBaru(e.target.value)}
                          className="w-full text-xs p-2.5 border rounded-lg focus:outline-blue-500"
                          required
                        />
                        <button type="submit" className="w-full bg-blue-600 text-white text-xs font-bold py-2.5 rounded-lg">
                          Tambah Warga
                        </button>
                      </form>
                    </div>
                  </>
                )}
              </div>
            )}
          </main>

          <!-- Bottom Navigation Bar -->
          <nav className="fixed bottom-0 left-0 right-0 max-w-md mx-auto bg-white border-t border-slate-200 flex justify-around p-2 z-10 shadow-lg">
            <button 
              onClick={() => setTab('ringkasan')} 
              className={`flex flex-col items-center gap-1 text-xs font-semibold py-1 px-3 ${tab === 'ringkasan' ? 'text-blue-600' : 'text-slate-400'}`}
            >
              <i data-lucide="wallet" className="w-5 h-5"></i>
              <span>Ringkasan</span>
            </button>
            <button 
              onClick={() => setTab('kutip')} 
              className={`flex flex-col items-center gap-1 text-xs font-semibold py-1 px-3 ${tab === 'kutip' ? 'text-blue-600' : 'text-slate-400'}`}
            >
              <i data-lucide="users" className="w-5 h-5"></i>
              <span>Iuran Warga</span>
            </button>
            <button 
              onClick={() => setTab('admin')} 
              className={`flex flex-col items-center gap-1 text-xs font-semibold py-1 px-3 ${tab === 'admin' ? 'text-blue-600' : 'text-slate-400'}`}
            >
              <i data-lucide="settings" className="w-5 h-5"></i>
              <span>Pengaturan</span>
            </button>
          </nav>

          <!-- Modal Login PIN -->
          {showPinModal && (
            <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4 z-50">
              <div className="bg-white w-full max-w-xs p-5 rounded-2xl shadow-xl text-center">
                <h3 className="font-bold text-slate-800">Akses Pengurus RT</h3>
                <p className="text-xs text-slate-500 mt-1 mb-4">Masukkan PIN Khusus Bendahara/Pengutip</p>
                <form onSubmit={handleLoginBendahara}>
                  <input 
                    type="password" 
                    placeholder="PIN (Default: 1234)" 
                    value={pinInput}
                    onChange={e => setPinInput(e.target.value)}
                    className="w-full text-center text-lg font-bold letter-spacing-2 p-2 border rounded-lg mb-3 focus:outline-blue-500"
                    autoFocus
                  />
                  <div className="flex gap-2">
                    <button 
                      type="button" 
                      onClick={() => setShowPinModal(false)}
                      className="w-1/2 text-xs py-2 bg-slate-100 text-slate-600 font-bold rounded-lg"
                    >
                      Batal
                    </button>
                    <button 
                      type="submit" 
                      className="w-1/2 text-xs py-2 bg-blue-600 text-white font-bold rounded-lg"
                    >
                      Masuk
                    </button>
                  </div>
                </form>
              </div>
            </div>
          )}

        </div>
      );
    }

    ReactDOM.render(<App />, document.getElementById('root'));
  </script>
</body>
</html>
