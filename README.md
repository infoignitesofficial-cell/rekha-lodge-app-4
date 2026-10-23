```react
import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, collection, doc, addDoc, updateDoc, onSnapshot, deleteDoc, setDoc 
} from 'firebase/firestore';
import { 
  ArrowLeft, Home, Users, FileText, IndianRupee, History, Settings as SettingsIcon, 
  WifiOff, Plus, CheckCircle, AlertCircle, Search, Edit2, UserPlus, FilePlus
} from 'lucide-react';

// --- Firebase Initialization ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'rekha-lodge-default';

// --- Helper Functions ---
const formatCurrency = (amount) => `₹${Number(amount).toLocaleString('en-IN')}`;
const formatDate = (dateStr) => {
  if (!dateStr) return 'N/A';
  return new Date(dateStr).toLocaleDateString('en-IN', { day: '2-digit', month: 'short', year: 'numeric' });
};

export default function RekhaLodgeApp() {
  // --- Auth & Network State ---
  const [user, setUser] = useState(null);
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  // --- App Data State ---
  const [rooms, setRooms] = useState([]);
  const [tenants, setTenants] = useState([]);
  const [bills, setBills] = useState([]);
  const [payments, setPayments] = useState([]);
  const [activities, setActivities] = useState([]);
  const [users, setUsers] = useState([]);
  const [settings, setSettings] = useState({ lodgeName: 'Rekha Lodge', defaultElecRate: 9 });
  
  // --- Navigation & UI State ---
  const [navStack, setNavStack] = useState(['dashboard']); // Stack for Back button logic
  const [toast, setToast] = useState(null);
  const [loading, setLoading] = useState(true);

  // Derived State
  const currentView = navStack[navStack.length - 1];
  const currentUserRole = useMemo(() => {
    if (!user) return null;
    const dbUser = users.find(u => u.uid === user.uid);
    if (!dbUser && users.length === 0) return 'owner'; // First user becomes owner
    return dbUser ? dbUser.role : 'unauthorized';
  }, [user, users]);

  const canEdit = currentUserRole === 'owner' || currentUserRole === 'admin';

  // --- Effects ---
  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth Error:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    setLoading(true);

    const getColRef = (colName) => collection(db, 'artifacts', appId, 'public', 'data', colName);

    // Using basic onSnapshot without complex queries (Rule 2 & 3)
    const unsubRooms = onSnapshot(getColRef('rekha_rooms'), (snap) => {
      setRooms(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    }, console.error);

    const unsubTenants = onSnapshot(getColRef('rekha_tenants'), (snap) => {
      setTenants(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    }, console.error);

    const unsubBills = onSnapshot(getColRef('rekha_bills'), (snap) => {
      setBills(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    }, console.error);

    const unsubPayments = onSnapshot(getColRef('rekha_payments'), (snap) => {
      setPayments(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    }, console.error);

    const unsubActivities = onSnapshot(getColRef('rekha_activities'), (snap) => {
      setActivities(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    }, console.error);

    const unsubUsers = onSnapshot(getColRef('rekha_users'), (snap) => {
      setUsers(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    }, console.error);

    const unsubSettings = onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'rekha_settings', 'global'), (docSnap) => {
      if (docSnap.exists()) {
        setSettings(docSnap.data());
      }
      setLoading(false);
    }, console.error);

    return () => {
      unsubRooms(); unsubTenants(); unsubBills(); unsubPayments(); unsubActivities(); unsubUsers(); unsubSettings();
    };
  }, [user]);

  // --- Auto-register Owner if first ---
  useEffect(() => {
    if (user && !loading && users.length === 0) {
      // First user to open the app becomes the Owner
      const registerOwner = async () => {
        try {
          await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'rekha_users'), {
            uid: user.uid,
            name: 'Lodge Owner',
            role: 'owner',
            addedAt: new Date().toISOString()
          });
          await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rekha_settings', 'global'), {
            lodgeName: 'Rekha Lodge',
            defaultElecRate: 9
          });
          logActivity('System', 'Initialized system and created Owner account.');
        } catch (e) { console.error(e); }
      };
      registerOwner();
    }
  }, [user, loading, users.length]);

  // --- Navigation Helpers ---
  const navigate = (view) => setNavStack(prev => [...prev, view]);
  const goBack = () => setNavStack(prev => prev.length > 1 ? prev.slice(0, -1) : prev);
  const showToast = (msg) => {
    setToast(msg);
    setTimeout(() => setToast(null), 3000);
  };

  // --- Database Helpers ---
  const logActivity = async (action, details) => {
    if (!user) return;
    const userName = users.find(u => u.uid === user.uid)?.name || 'Unknown User';
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'rekha_activities'), {
        user: userName,
        action,
        details,
        timestamp: new Date().toISOString()
      });
    } catch (e) { console.error("Activity Log Error:", e); }
  };

  const getCol = (colName) => collection(db, 'artifacts', appId, 'public', 'data', colName);
  const getDocRef = (colName, docId) => doc(db, 'artifacts', appId, 'public', 'data', colName, docId);

  // --- Views ---

  if (loading) return <div className="flex h-screen items-center justify-center bg-gray-50"><div className="text-xl animate-pulse">Loading Rekha Lodge...</div></div>;
  if (currentUserRole === 'unauthorized') return (
    <div className="flex flex-col h-screen items-center justify-center bg-gray-50 p-6 text-center">
      <AlertCircle className="w-16 h-16 text-red-500 mb-4" />
      <h1 className="text-2xl font-bold mb-2">Unauthorized Access</h1>
      <p className="mb-4">You do not have permission to access Rekha Lodge.</p>
      <p className="text-sm bg-gray-200 p-2 rounded">Your ID: <br/><span className="font-mono">{user?.uid}</span></p>
      <p className="text-sm mt-4 text-gray-500">Ask the Owner to add your ID in the Settings.</p>
    </div>
  );

  return (
    <div className="min-h-screen bg-gray-100 flex flex-col font-sans sm:max-w-md md:max-w-2xl lg:max-w-3xl mx-auto shadow-2xl relative">
      {/* Network Status Banner */}
      {!isOnline && (
        <div className="bg-red-500 text-white text-center py-1 text-sm font-semibold flex items-center justify-center gap-2 z-50">
          <WifiOff size={16} /> No Internet - Changes will sync when online
        </div>
      )}

      {/* Top Bar with Back Button */}
      <div className="bg-blue-700 text-white p-4 flex items-center shadow-md sticky top-0 z-40">
        {navStack.length > 1 ? (
          <button onClick={goBack} className="flex items-center gap-2 p-2 -ml-2 rounded-lg hover:bg-blue-600 active:bg-blue-800 transition">
            <ArrowLeft size={24} /> <span className="font-semibold text-lg">Back</span>
          </button>
        ) : (
          <div className="flex items-center gap-2">
            <Home size={28} />
            <h1 className="text-2xl font-bold">{settings.lodgeName}</h1>
          </div>
        )}
        <div className="ml-auto text-sm font-medium opacity-90 bg-blue-800 px-3 py-1 rounded-full">
          {users.find(u => u.uid === user.uid)?.name || 'User'} ({currentUserRole})
        </div>
      </div>

      {/* Main Content Area */}
      <div className="flex-1 overflow-y-auto p-4 pb-24">
        {currentView === 'dashboard' && <DashboardView navigate={navigate} rooms={rooms} tenants={tenants} bills={bills} payments={payments} />}
        {currentView === 'rooms_tenants' && <RoomsTenantsView navigate={navigate} rooms={rooms} tenants={tenants} canEdit={canEdit} />}
        {currentView === 'add_room' && <AddEditRoomView goBack={goBack} getCol={getCol} logActivity={logActivity} />}
        {currentView.startsWith('edit_room_') && <AddEditRoomView goBack={goBack} getCol={getCol} getDocRef={getDocRef} logActivity={logActivity} room={rooms.find(r => r.id === currentView.split('_')[2])} />}
        {currentView === 'add_tenant' && <AddEditTenantView goBack={goBack} getCol={getCol} getDocRef={getDocRef} logActivity={logActivity} rooms={rooms} />}
        {currentView.startsWith('edit_tenant_') && <AddEditTenantView goBack={goBack} getCol={getCol} getDocRef={getDocRef} logActivity={logActivity} rooms={rooms} tenant={tenants.find(t => t.id === currentView.split('_')[2])} />}
        {currentView === 'generate_bills' && <GenerateBillsView navigate={navigate} tenants={tenants} rooms={rooms} bills={bills} settings={settings} getCol={getCol} getDocRef={getDocRef} logActivity={logActivity} goBack={goBack} />}
        {currentView === 'payments_dues' && <PaymentsDuesView navigate={navigate} bills={bills} tenants={tenants} rooms={rooms} payments={payments} getCol={getCol} getDocRef={getDocRef} logActivity={logActivity} canEdit={canEdit} />}
        {currentView === 'activity_history' && <ActivityHistoryView activities={activities} />}
        {currentView === 'settings' && <SettingsView settings={settings} users={users} currentUser={user} role={currentUserRole} getCol={getCol} getDocRef={getDocRef} logActivity={logActivity} />}
      </div>

      {/* Toast Notification */}
      {toast && (
        <div className="fixed bottom-6 left-1/2 transform -translate-x-1/2 bg-gray-800 text-white px-6 py-3 rounded-full shadow-lg z-50 flex items-center gap-2 animate-bounce">
          <CheckCircle size={20} className="text-green-400" />
          {toast}
        </div>
      )}
    </div>
  );
}

// ==========================================
// 1. Dashboard View
// ==========================================
function DashboardView({ navigate, rooms, tenants, bills, payments }) {
  const currentMonth = new Date().toISOString().slice(0, 7); // YYYY-MM
  const occupiedRooms = rooms.filter(r => r.status === 'occupied').length;
  const activeTenants = tenants.filter(t => t.status === 'active').length;
  const billsThisMonth = bills.filter(b => b.periodStart.startsWith(currentMonth)).length;
  const totalDues = bills.reduce((sum, b) => sum + Number(b.outstanding), 0);
  const totalReceived = payments.reduce((sum, p) => sum + Number(p.amount), 0);

  const menuItems = [
    { id: 'rooms_tenants', title: 'Rooms & Tenants', icon: Users, color: 'bg-indigo-500' },
    { id: 'generate_bills', title: 'Generate Bills', icon: FilePlus, color: 'bg-green-500' },
    { id: 'payments_dues', title: 'Payments & Dues', icon: IndianRupee, color: 'bg-yellow-500' },
    { id: 'activity_history', title: 'Activity History', icon: History, color: 'bg-purple-500' },
    { id: 'settings', title: 'Settings', icon: SettingsIcon, color: 'bg-gray-600' },
  ];

  return (
    <div className="space-y-6">
      {/* Stats Grid */}
      <div className="grid grid-cols-2 gap-4">
        <StatCard title="Rooms" value={`${occupiedRooms} / ${rooms.length}`} sub="Occupied" />
        <StatCard title="Active Tenants" value={activeTenants} sub="Total" />
        <StatCard title="Outstanding Dues" value={formatCurrency(totalDues)} sub="Total Unpaid" highlight={totalDues > 0} />
        <StatCard title="Payments Rcvd" value={formatCurrency(totalReceived)} sub="All Time" />
      </div>

      {/* Main Navigation Menu */}
      <h2 className="text-xl font-bold text-gray-800 mt-8 mb-4 border-b pb-2">Main Menu</h2>
      <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
        {menuItems.map(item => (
          <button 
            key={item.id} 
            onClick={() => navigate(item.id)}
            className={`${item.color} text-white p-5 rounded-xl shadow-md hover:opacity-90 active:scale-95 transition flex items-center gap-4`}
          >
            <item.icon size={32} />
            <span className="text-xl font-semibold">{item.title}</span>
          </button>
        ))}
      </div>
    </div>
  );
}

function StatCard({ title, value, sub, highlight }) {
  return (
    <div className={`bg-white p-4 rounded-xl shadow border-l-4 ${highlight ? 'border-red-500' : 'border-blue-500'}`}>
      <div className="text-gray-500 text-sm font-semibold">{title}</div>
      <div className={`text-2xl font-bold my-1 ${highlight ? 'text-red-600' : 'text-gray-800'}`}>{value}</div>
      <div className="text-gray-400 text-xs">{sub}</div>
    </div>
  );
}

// ==========================================
// 2. Rooms & Tenants View
// ==========================================
function RoomsTenantsView({ navigate, rooms, tenants, canEdit }) {
  const [tab, setTab] = useState('rooms');
  const [search, setSearch] = useState('');

  // Sort rooms safely handling non-numeric room names if any
  const sortedRooms = [...rooms].sort((a, b) => String(a.roomNumber).localeCompare(String(b.roomNumber), undefined, { numeric: true }));
  
  const filteredTenants = tenants.filter(t => 
    t.name.toLowerCase().includes(search.toLowerCase()) || 
    t.roomNumber?.toLowerCase().includes(search.toLowerCase())
  );

  return (
    <div className="space-y-4">
      <div className="flex bg-gray-200 rounded-lg p-1">
        <button className={`flex-1 py-3 text-lg font-semibold rounded-md ${tab === 'rooms' ? 'bg-white shadow' : 'text-gray-600'}`} onClick={() => setTab('rooms')}>Rooms</button>
        <button className={`flex-1 py-3 text-lg font-semibold rounded-md ${tab === 'tenants' ? 'bg-white shadow' : 'text-gray-600'}`} onClick={() => setTab('tenants')}>Tenants</button>
      </div>

      {tab === 'rooms' && (
        <div className="space-y-4">
          {canEdit && (
            <button onClick={() => navigate('add_room')} className="w-full bg-blue-600 text-white py-4 rounded-xl font-bold text-lg flex items-center justify-center gap-2 shadow-md">
              <Plus size={24} /> Add New Room
            </button>
          )}
          <div className="grid grid-cols-2 gap-3">
            {sortedRooms.map(room => (
              <div key={room.id} onClick={() => canEdit && navigate(`edit_room_${room.id}`)} className="bg-white p-4 rounded-xl shadow border border-gray-100 flex flex-col items-center justify-center cursor-pointer hover:bg-blue-50 transition">
                <span className="text-2xl font-bold text-gray-800">{room.roomNumber}</span>
                <span className={`text-sm font-semibold mt-1 px-2 py-0.5 rounded-full ${room.status === 'occupied' ? 'bg-green-100 text-green-700' : 'bg-gray-100 text-gray-600'}`}>
                  {room.status.toUpperCase()}
                </span>
                <span className="text-gray-500 text-sm mt-2">{formatCurrency(room.monthlyRent)}/mo</span>
              </div>
            ))}
          </div>
          {rooms.length === 0 && <div className="text-center text-gray-500 p-8">No rooms found. Add a room to start.</div>}
        </div>
      )}

      {tab === 'tenants' && (
        <div className="space-y-4">
          {canEdit && (
            <button onClick={() => navigate('add_tenant')} className="w-full bg-indigo-600 text-white py-4 rounded-xl font-bold text-lg flex items-center justify-center gap-2 shadow-md">
              <UserPlus size={24} /> Add New Tenant
            </button>
          )}
          <div className="relative">
            <Search className="absolute left-3 top-3.5 text-gray-400" size={20} />
            <input type="text" placeholder="Search tenants..." className="w-full p-3 pl-10 border rounded-xl text-lg shadow-sm" value={search} onChange={e => setSearch(e.target.value)} />
          </div>
          <div className="space-y-3">
            {filteredTenants.map(tenant => (
              <div key={tenant.id} onClick={() => canEdit && navigate(`edit_tenant_${tenant.id}`)} className="bg-white p-4 rounded-xl shadow border border-gray-100 cursor-pointer hover:bg-indigo-50 transition flex justify-between items-center">
                <div>
                  <h3 className="text-lg font-bold text-gray-800">{tenant.name}</h3>
                  <p className="text-gray-500">Room: {tenant.roomNumber} | {tenant.phone}</p>
                </div>
                <div className={`text-sm font-bold px-3 py-1 rounded-full ${tenant.status === 'active' ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
                  {tenant.status.toUpperCase()}
                </div>
              </div>
            ))}
            {filteredTenants.length === 0 && <div className="text-center text-gray-500 p-8">No tenants found.</div>}
          </div>
        </div>
      )}
    </div>
  );
}

// --- Add/Edit Room ---
function AddEditRoomView({ goBack, getCol, getDocRef, logActivity, room }) {
  const [formData, setFormData] = useState(room || { roomNumber: '', monthlyRent: '', status: 'vacant', notes: '' });
  const [saving, setSaving] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!formData.roomNumber || !formData.monthlyRent) return alert("Please fill required fields.");
    setSaving(true);
    try {
      if (room) {
        await updateDoc(getDocRef('rekha_rooms', room.id), formData);
        logActivity('Edit Room', `Updated Room ${formData.roomNumber}`);
      } else {
        await addDoc(getCol('rekha_rooms'), formData);
        logActivity('Add Room', `Added Room ${formData.roomNumber}`);
      }
      goBack();
    } catch (err) { console.error(err); alert("Error saving room."); }
    setSaving(false);
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4 bg-white p-6 rounded-xl shadow">
      <h2 className="text-2xl font-bold mb-4">{room ? 'Edit Room' : 'Add New Room'}</h2>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Room Number</label>
        <input type="text" className="w-full p-3 border rounded-lg text-lg" value={formData.roomNumber} onChange={e => setFormData({...formData, roomNumber: e.target.value})} required />
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Monthly Rent (₹)</label>
        <input type="number" className="w-full p-3 border rounded-lg text-lg" value={formData.monthlyRent} onChange={e => setFormData({...formData, monthlyRent: Number(e.target.value)})} required />
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Status</label>
        <select className="w-full p-3 border rounded-lg text-lg" value={formData.status} onChange={e => setFormData({...formData, status: e.target.value})}>
          <option value="vacant">Vacant</option>
          <option value="occupied">Occupied</option>
        </select>
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Notes (Optional)</label>
        <textarea className="w-full p-3 border rounded-lg" rows="3" value={formData.notes} onChange={e => setFormData({...formData, notes: e.target.value})}></textarea>
      </div>
      <button type="submit" disabled={saving} className="w-full bg-blue-600 text-white font-bold text-lg py-4 rounded-xl shadow mt-4 disabled:opacity-50">
        {saving ? 'Saving...' : 'Save Room'}
      </button>
    </form>
  );
}

// --- Add/Edit Tenant ---
function AddEditTenantView({ goBack, getCol, getDocRef, logActivity, rooms, tenant }) {
  const [formData, setFormData] = useState(tenant || { name: '', phone: '', roomId: '', roomNumber: '', monthlyRent: '', moveInDate: new Date().toISOString().split('T')[0], status: 'active' });
  const [saving, setSaving] = useState(false);

  const vacantRooms = rooms.filter(r => r.status === 'vacant' || r.id === tenant?.roomId).sort((a,b) => String(a.roomNumber).localeCompare(String(b.roomNumber), undefined, {numeric: true}));

  const handleRoomChange = (roomId) => {
    const selectedRoom = rooms.find(r => r.id === roomId);
    if (selectedRoom) {
      setFormData({...formData, roomId: selectedRoom.id, roomNumber: selectedRoom.roomNumber, monthlyRent: selectedRoom.monthlyRent});
    } else {
      setFormData({...formData, roomId: '', roomNumber: '', monthlyRent: ''});
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!formData.name || !formData.roomId) return alert("Name and Room are required.");
    setSaving(true);
    try {
      if (tenant) {
        await updateDoc(getDocRef('rekha_tenants', tenant.id), formData);
        // Free old room if changed
        if (tenant.roomId !== formData.roomId) {
           await updateDoc(getDocRef('rekha_rooms', tenant.roomId), { status: 'vacant', assignedTenantId: null });
        }
        await updateDoc(getDocRef('rekha_rooms', formData.roomId), { status: 'occupied', assignedTenantId: tenant.id });
        logActivity('Edit Tenant', `Updated Tenant ${formData.name}`);
      } else {
        const docRef = await addDoc(getCol('rekha_tenants'), formData);
        await updateDoc(getDocRef('rekha_rooms', formData.roomId), { status: 'occupied', assignedTenantId: docRef.id });
        logActivity('Add Tenant', `Added Tenant ${formData.name} to Room ${formData.roomNumber}`);
      }
      goBack();
    } catch (err) { console.error(err); alert("Error saving tenant."); }
    setSaving(false);
  };

  const handleDeactivate = async () => {
    if (!window.confirm("Mark tenant as inactive and free the room? Historical records will be kept.")) return;
    setSaving(true);
    try {
      await updateDoc(getDocRef('rekha_tenants', tenant.id), { status: 'inactive', moveOutDate: new Date().toISOString().split('T')[0] });
      await updateDoc(getDocRef('rekha_rooms', tenant.roomId), { status: 'vacant', assignedTenantId: null });
      logActivity('Deactivate Tenant', `Deactivated Tenant ${tenant.name}, freed Room ${tenant.roomNumber}`);
      goBack();
    } catch (err) { console.error(err); }
    setSaving(false);
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4 bg-white p-6 rounded-xl shadow">
      <h2 className="text-2xl font-bold mb-4">{tenant ? 'Edit Tenant' : 'Add New Tenant'}</h2>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Full Name</label>
        <input type="text" className="w-full p-3 border rounded-lg text-lg" value={formData.name} onChange={e => setFormData({...formData, name: e.target.value})} required />
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Phone Number</label>
        <input type="tel" className="w-full p-3 border rounded-lg text-lg" value={formData.phone} onChange={e => setFormData({...formData, phone: e.target.value})} required />
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Assign Room</label>
        <select className="w-full p-3 border rounded-lg text-lg bg-white" value={formData.roomId} onChange={e => handleRoomChange(e.target.value)} required disabled={tenant && formData.status === 'inactive'}>
          <option value="">-- Select Room --</option>
          {vacantRooms.map(r => <option key={r.id} value={r.id}>Room {r.roomNumber} (₹{r.monthlyRent})</option>)}
        </select>
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Agreed Monthly Rent (₹)</label>
        <input type="number" className="w-full p-3 border rounded-lg text-lg" value={formData.monthlyRent} onChange={e => setFormData({...formData, monthlyRent: Number(e.target.value)})} required />
      </div>
      <div>
        <label className="block text-gray-700 font-semibold mb-1">Move-in Date</label>
        <input type="date" className="w-full p-3 border rounded-lg text-lg" value={formData.moveInDate} onChange={e => setFormData({...formData, moveInDate: e.target.value})} required />
      </div>
      
      <button type="submit" disabled={saving} className="w-full bg-indigo-600 text-white font-bold text-lg py-4 rounded-xl shadow mt-4 disabled:opacity-50">
        {saving ? 'Saving...' : 'Save Tenant'}
      </button>

      {tenant && tenant.status === 'active' && (
        <button type="button" onClick={handleDeactivate} className="w-full bg-red-100 text-red-700 font-bold text-lg py-3 rounded-xl mt-4 border border-red-200">
          Mark as Inactive (Moved Out)
        </button>
      )}
    </form>
  );
}

// ==========================================
// 3. Generate Bills View
// ==========================================
function GenerateBillsView({ navigate, tenants, rooms, bills, settings, getCol, logActivity, goBack }) {
  const activeTenants = tenants.filter(t => t.status === 'active');
  const [selectedTenantId, setSelectedTenantId] = useState('');
  const [formData, setFormData] = useState({
    periodStart: '', periodEnd: '', rentAmount: '', prevMeter: '', currentMeter: '', additionalCharges: 0, dueDate: ''
  });
  const [preview, setPreview] = useState(null);
  const [saving, setSaving] = useState(false);

  // Auto-fill form when tenant selected
  useEffect(() => {
    if (!selectedTenantId) return;
    const t = activeTenants.find(t => t.id === selectedTenantId);
    if (!t) return;

    // Find previous bill for meter reading
    const tenantBills = bills.filter(b => b.tenantId === t.id).sort((a,b) => new Date(b.periodEnd) - new Date(a.periodEnd));
    const lastBill = tenantBills[0];

    const today = new Date();
    const firstDay = new Date(today.getFullYear(), today.getMonth(), 1).toISOString().split('T')[0];
    const lastDay = new Date(today.getFullYear(), today.getMonth() + 1, 0).toISOString().split('T')[0];
    const dueDay = new Date(today.getFullYear(), today.getMonth() + 1, 5).toISOString().split('T')[0]; // Default 5th of next month

    setFormData({
      periodStart: firstDay,
      periodEnd: lastDay,
      rentAmount: t.monthlyRent || 0,
      prevMeter: lastBill ? lastBill.currentMeter : 0,
      currentMeter: '',
      additionalCharges: 0,
      dueDate: dueDay
    });
    setPreview(null);
  }, [selectedTenantId]);

  const handlePreview = () => {
    const t = activeTenants.find(t => t.id === selectedTenantId);
    if (!t || !formData.periodStart || !formData.periodEnd || formData.currentMeter === '') return;

    const current = Number(formData.currentMeter);
    const prev = Number(formData.prevMeter);
    if (current < prev) return alert("Current meter reading cannot be less than previous.");

    // Check duplicate bill for same period
    const isDuplicate = bills.some(b => b.tenantId === t.id && b.periodStart === formData.periodStart && b.periodEnd === formData.periodEnd);
    if (isDuplicate) {
      if (!window.confirm("A bill for this exact period already exists. Are you sure you want to create another one?")) return;
    }

    const units = current - prev;
    const elecRate = Number(settings.defaultElecRate);
    const elecCharges = units * elecRate;
    const rent = Number(formData.rentAmount);
    const addl = Number(formData.additionalCharges);
    const total = rent + elecCharges + addl;

    setPreview({
      tenantId: t.id,
      tenantName: t.name,
      roomId: t.roomId,
      roomNumber: t.roomNumber,
      periodStart: formData.periodStart,
      periodEnd: formData.periodEnd,
      rentAmount: rent,
      prevMeter: prev,
      currentMeter: current,
      units,
      elecRate,
      elecCharges,
      additionalCharges: addl,
      totalBill: total,
      outstanding: total,
      issueDate: new Date().toISOString(),
      dueDate: formData.dueDate,
      status: 'unpaid'
    });
  };

  const handleGenerate = async () => {
    if (!preview) return;
    setSaving(true);
    try {
      await addDoc(getCol('rekha_bills'), preview);
      logActivity('Generate Bill', `Generated Bill of ${formatCurrency(preview.totalBill)} for ${preview.tenantName} (Room ${preview.roomNumber})`);
      goBack();
    } catch (err) { console.error(err); alert("Failed to generate bill."); }
    setSaving(false);
  };

  return (
    <div className="space-y-6">
      <div className="bg-white p-6 rounded-xl shadow border border-gray-100">
        <h2 className="text-2xl font-bold mb-4 flex items-center gap-2"><FilePlus /> Generate New Bill</h2>
        
        <div className="space-y-4">
          <div>
            <label className="block text-gray-700 font-semibold mb-1">Select Tenant</label>
            <select className="w-full p-3 border rounded-lg text-lg bg-white" value={selectedTenantId} onChange={e => setSelectedTenantId(e.target.value)}>
              <option value="">-- Choose Active Tenant --</option>
              {activeTenants.map(t => <option key={t.id} value={t.id}>{t.name} (Room {t.roomNumber})</option>)}
            </select>
          </div>

          {selectedTenantId && (
            <>
              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="block text-gray-700 font-semibold mb-1 text-sm">Period Start</label>
                  <input type="date" className="w-full p-3 border rounded-lg" value={formData.periodStart} onChange={e => setFormData({...formData, periodStart: e.target.value})} />
                </div>
                <div>
                  <label className="block text-gray-700 font-semibold mb-1 text-sm">Period End</label>
                  <input type="date" className="w-full p-3 border rounded-lg" value={formData.periodEnd} onChange={e => setFormData({...formData, periodEnd: e.target.value})} />
                </div>
              </div>

              <div>
                <label className="block text-gray-700 font-semibold mb-1">Rent Amount (₹)</label>
                <input type="number" className="w-full p-3 border rounded-lg text-lg" value={formData.rentAmount} onChange={e => setFormData({...formData, rentAmount: e.target.value})} />
                <p className="text-xs text-gray-500 mt-1">Edit this for prorated rent if needed.</p>
              </div>

              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="block text-gray-700 font-semibold mb-1 text-sm">Prev Meter Reading</label>
                  <input type="number" className="w-full p-3 border rounded-lg" value={formData.prevMeter} onChange={e => setFormData({...formData, prevMeter: e.target.value})} />
                </div>
                <div>
                  <label className="block text-gray-700 font-semibold mb-1 text-sm">Current Meter</label>
                  <input type="number" className="w-full p-3 border rounded-lg" value={formData.currentMeter} onChange={e => setFormData({...formData, currentMeter: e.target.value})} />
                </div>
              </div>

              <div>
                <label className="block text-gray-700 font-semibold mb-1">Additional Charges (₹)</label>
                <input type="number" className="w-full p-3 border rounded-lg text-lg" value={formData.additionalCharges} onChange={e => setFormData({...formData, additionalCharges: e.target.value})} />
              </div>

              <div>
                <label className="block text-gray-700 font-semibold mb-1">Due Date</label>
                <input type="date" className="w-full p-3 border rounded-lg text-lg" value={formData.dueDate} onChange={e => setFormData({...formData, dueDate: e.target.value})} />
              </div>

              <button type="button" onClick={handlePreview} className="w-full bg-gray-800 text-white font-bold text-lg py-3 rounded-xl mt-2">
                Preview Bill
              </button>
            </>
          )}
        </div>
      </div>

      {preview && (
        <div className="bg-yellow-50 p-6 rounded-xl shadow border border-yellow-200">
          <h3 className="text-xl font-bold mb-4 text-yellow-900 border-b border-yellow-300 pb-2">Bill Preview</h3>
          <div className="space-y-2 text-yellow-800 text-lg">
            <div className="flex justify-between"><span>Rent:</span> <strong>{formatCurrency(preview.rentAmount)}</strong></div>
            <div className="flex justify-between text-sm"><span>Electricity ({preview.units} units @ ₹{preview.elecRate}):</span> <span>{formatCurrency(preview.elecCharges)}</span></div>
            {preview.additionalCharges > 0 && <div className="flex justify-between text-sm"><span>Additional:</span> <span>{formatCurrency(preview.additionalCharges)}</span></div>}
            <div className="flex justify-between text-xl font-bold border-t border-yellow-300 pt-2 mt-2">
              <span>Total Bill:</span> <span>{formatCurrency(preview.totalBill)}</span>
            </div>
          </div>
          <button onClick={handleGenerate} disabled={saving} className="w-full bg-green-600 text-white font-bold text-xl py-4 rounded-xl shadow mt-6 disabled:opacity-50">
            {saving ? 'Saving...' : 'Confirm & Generate Bill'}
          </button>
        </div>
      )}
    </div>
  );
}

// ==========================================
// 4. Payments & Dues View
// ==========================================
function PaymentsDuesView({ bills, payments, getCol, getDocRef, logActivity, canEdit }) {
  const [activeTab, setActiveTab] = useState('unpaid');
  const [selectedBill, setSelectedBill] = useState(null);
  const [payAmount, setPayAmount] = useState('');
  const [payMethod, setPayMethod] = useState('Cash');
  const [saving, setSaving] = useState(false);
  const [showDemand, setShowDemand] = useState(null);

  // Sort bills newest first
  const sortedBills = [...bills].sort((a,b) => new Date(b.issueDate) < new Date(a.issueDate) ? 1 : -1);
  const unpaidBills = sortedBills.filter(b => b.outstanding > 0);
  const paidBills = sortedBills.filter(b => b.outstanding <= 0);

  const displayBills = activeTab === 'unpaid' ? unpaidBills : paidBills;

  const handleRecordPayment = async (e) => {
    e.preventDefault();
    if (!selectedBill || !payAmount) return;
    const amount = Number(payAmount);
    if (amount <= 0 || amount > selectedBill.outstanding) return alert("Invalid amount. Cannot exceed outstanding balance.");
    
    setSaving(true);
    try {
      const newOutstanding = selectedBill.outstanding - amount;
      const status = newOutstanding === 0 ? 'paid' : 'partial';
      
      // 1. Record payment
      await addDoc(getCol('rekha_payments'), {
        billId: selectedBill.id,
        tenantId: selectedBill.tenantId,
        tenantName: selectedBill.tenantName,
        roomNumber: selectedBill.roomNumber,
        amount: amount,
        method: payMethod,
        date: new Date().toISOString()
      });

      // 2. Update Bill
      await updateDoc(getDocRef('rekha_bills', selectedBill.id), {
        outstanding: newOutstanding,
        status: status
      });

      logActivity('Record Payment', `Received ${formatCurrency(amount)} via ${payMethod} from ${selectedBill.tenantName}`);
      setSelectedBill(null);
      setPayAmount('');
    } catch (err) { console.error(err); alert("Failed to record payment."); }
    setSaving(false);
  };

  return (
    <div className="space-y-4">
      <div className="flex bg-gray-200 rounded-lg p-1">
        <button className={`flex-1 py-3 text-lg font-semibold rounded-md ${activeTab === 'unpaid' ? 'bg-white shadow text-red-600' : 'text-gray-600'}`} onClick={() => setActiveTab('unpaid')}>Dues ({unpaidBills.length})</button>
        <button className={`flex-1 py-3 text-lg font-semibold rounded-md ${activeTab === 'paid' ? 'bg-white shadow text-green-600' : 'text-gray-600'}`} onClick={() => setActiveTab('paid')}>Paid History</button>
      </div>

      <div className="space-y-4">
        {displayBills.map(bill => (
          <div key={bill.id} className="bg-white p-5 rounded-xl shadow border border-gray-200">
            <div className="flex justify-between items-start mb-2">
              <div>
                <h3 className="text-lg font-bold text-gray-900">{bill.tenantName}</h3>
                <p className="text-gray-600 text-sm">Room {bill.roomNumber} | Period: {formatDate(bill.periodStart)} to {formatDate(bill.periodEnd)}</p>
              </div>
              <div className="text-right">
                <div className="text-xl font-bold text-gray-800">{formatCurrency(bill.totalBill)}</div>
                <div className={`text-sm font-semibold ${bill.outstanding > 0 ? 'text-red-600' : 'text-green-600'}`}>
                  {bill.outstanding > 0 ? `Due: ${formatCurrency(bill.outstanding)}` : 'PAID'}
                </div>
              </div>
            </div>
            
            {bill.outstanding > 0 && canEdit && (
              <div className="mt-4 flex gap-2">
                <button onClick={() => { setSelectedBill(bill); setPayAmount(bill.outstanding); }} className="flex-1 bg-green-600 text-white py-2 rounded-lg font-semibold hover:bg-green-700">
                  Record Payment
                </button>
                <button onClick={() => setShowDemand(bill)} className="flex-1 bg-orange-100 text-orange-800 border border-orange-300 py-2 rounded-lg font-semibold hover:bg-orange-200">
                  Send Demand
                </button>
              </div>
            )}
          </div>
        ))}
        {displayBills.length === 0 && <div className="text-center text-gray-500 py-8">No bills found in this category.</div>}
      </div>

      {/* Record Payment Modal */}
      {selectedBill && (
        <div className="fixed inset-0 bg-black/60 flex items-center justify-center p-4 z-50">
          <div className="bg-white rounded-2xl p-6 w-full max-w-md shadow-2xl">
            <h2 className="text-2xl font-bold mb-4">Record Payment</h2>
            <p className="mb-4 text-gray-600">Tenant: {selectedBill.tenantName} (Room {selectedBill.roomNumber})<br/>Outstanding: <strong className="text-red-600">{formatCurrency(selectedBill.outstanding)}</strong></p>
            
            <form onSubmit={handleRecordPayment} className="space-y-4">
              <div>
                <label className="block text-gray-700 font-semibold mb-1">Amount Received (₹)</label>
                <input type="number" className="w-full p-3 border rounded-lg text-xl font-bold" value={payAmount} onChange={e => setPayAmount(e.target.value)} max={selectedBill.outstanding} required />
              </div>
              <div>
                <label className="block text-gray-700 font-semibold mb-1">Payment Method</label>
                <select className="w-full p-3 border rounded-lg text-lg bg-white" value={payMethod} onChange={e => setPayMethod(e.target.value)}>
                  <option value="Cash">Cash</option>
                  <option value="UPI">UPI (Outside App)</option>
                  <option value="Bank Transfer">Bank Transfer</option>
                </select>
              </div>
              <div className="flex gap-3 pt-2">
                <button type="button" onClick={() => setSelectedBill(null)} className="flex-1 py-3 bg-gray-200 text-gray-800 font-bold rounded-xl">Cancel</button>
                <button type="submit" disabled={saving} className="flex-1 py-3 bg-green-600 text-white font-bold rounded-xl disabled:opacity-50">Confirm Payment</button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* Demand Modal */}
      {showDemand && (
        <div className="fixed inset-0 bg-black/60 flex items-center justify-center p-4 z-50">
          <div className="bg-white rounded-2xl p-6 w-full max-w-md shadow-2xl">
            <h2 className="text-2xl font-bold mb-4 text-orange-600">Payment Demand Notice</h2>
            <div className="bg-orange-50 p-4 border border-orange-200 rounded-lg text-orange-900 mb-6 font-medium whitespace-pre-wrap">
              Dear {showDemand.tenantName} (Room {showDemand.roomNumber}),
              {'\n\n'}
              This is a reminder that your rent/electricity bill of {formatCurrency(showDemand.totalBill)} is partially unpaid.
              {'\n\n'}
              Payments Received: {formatCurrency(showDemand.totalBill - showDemand.outstanding)}
              {'\n'}
              Current Outstanding Balance: {formatCurrency(showDemand.outstanding)}
              {'\n\n'}
              Please clear your dues by {formatDate(showDemand.dueDate)}.
              {'\n\n'}
              Regards,
              {'\n'}Rekha Lodge Management
            </div>
            <p className="text-xs text-gray-500 text-center mb-4 italic">* This preview is for viewing only. Copy/paste to your preferred messaging app to send to the tenant.</p>
            <button onClick={() => setShowDemand(null)} className="w-full py-3 bg-gray-800 text-white font-bold rounded-xl">Close Preview</button>
          </div>
        </div>
      )}
    </div>
  );
}

// ==========================================
// 5. Activity History View
// ==========================================
function ActivityHistoryView({ activities }) {
  const sorted = [...activities].sort((a,b) => new Date(b.timestamp) < new Date(a.timestamp) ? -1 : 1);
  return (
    <div className="space-y-3">
      {sorted.map(act => (
        <div key={act.id} className="bg-white p-4 rounded-xl shadow-sm border border-gray-200">
          <div className="flex justify-between items-start mb-1">
            <span className="font-bold text-blue-800">{act.user}</span>
            <span className="text-xs text-gray-500">{new Date(act.timestamp).toLocaleString('en-IN')}</span>
          </div>
          <div className="text-sm font-semibold text-gray-700">{act.action}</div>
          <div className="text-sm text-gray-600 mt-1">{act.details}</div>
        </div>
      ))}
      {sorted.length === 0 && <div className="text-center text-gray-500 p-8">No recent activity.</div>}
    </div>
  );
}

// ==========================================
// 6. Settings View
// ==========================================
function SettingsView({ settings, users, currentUser, role, getCol, getDocRef, logActivity }) {
  const [elecRate, setElecRate] = useState(settings.defaultElecRate || 9);
  const [saving, setSaving] = useState(false);
  const [newUserId, setNewUserId] = useState('');
  const [newUserRole, setNewUserRole] = useState('viewer');

  const handleSaveSettings = async () => {
    if (role !== 'owner') return alert("Only owner can change settings.");
    setSaving(true);
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rekha_settings', 'global'), {
        ...settings,
        defaultElecRate: Number(elecRate)
      });
      logActivity('Update Settings', `Changed default electricity rate to ₹${elecRate}`);
      alert("Settings saved successfully.");
    } catch (err) { console.error(err); alert("Failed to save."); }
    setSaving(false);
  };

  const handleAddUser = async (e) => {
    e.preventDefault();
    if (role !== 'owner') return;
    if (!newUserId) return;
    setSaving(true);
    try {
      await addDoc(getCol('rekha_users'), {
        uid: newUserId.trim(),
        name: `Family Member (${newUserRole})`,
        role: newUserRole,
        addedAt: new Date().toISOString()
      });
      logActivity('Manage Access', `Granted ${newUserRole} access to UID ${newUserId.substring(0,6)}...`);
      setNewUserId('');
    } catch (err) { console.error(err); alert("Failed to add user."); }
    setSaving(false);
  };

  const handleRemoveUser = async (userDocId) => {
    if (role !== 'owner') return;
    if (!window.confirm("Remove this user's access?")) return;
    try {
      await deleteDoc(getDocRef('rekha_users', userDocId));
      logActivity('Manage Access', `Revoked access for a user.`);
    } catch(e) { console.error(e); }
  };

  return (
    <div className="space-y-6">
      <div className="bg-white p-6 rounded-xl shadow border border-gray-100">
        <h2 className="text-xl font-bold mb-4 border-b pb-2">Your Account Info</h2>
        <p className="text-sm text-gray-600 mb-1">Role: <strong className="uppercase">{role}</strong></p>
        <p className="text-sm text-gray-600 mb-2">Your UID (Copy this to give to the Owner if needed):</p>
        <div className="bg-gray-100 p-3 rounded font-mono text-xs break-all border border-gray-300 select-all">
          {currentUser?.uid}
        </div>
      </div>

      {role === 'owner' && (
        <>
          <div className="bg-white p-6 rounded-xl shadow border border-gray-100">
            <h2 className="text-xl font-bold mb-4 border-b pb-2">Business Settings</h2>
            <div>
              <label className="block text-gray-700 font-semibold mb-1">Electricity Rate (₹ per unit)</label>
              <input type="number" className="w-full p-3 border rounded-lg text-lg mb-2" value={elecRate} onChange={e => setElecRate(e.target.value)} />
              <p className="text-xs text-gray-500 mb-4">Note: Changing this only affects future bills. Historical bills will remain unchanged.</p>
              <button onClick={handleSaveSettings} disabled={saving} className="bg-blue-600 text-white font-bold py-3 px-6 rounded-xl shadow">
                {saving ? 'Saving...' : 'Save Settings'}
              </button>
            </div>
          </div>

          <div className="bg-white p-6 rounded-xl shadow border border-gray-100">
            <h2 className="text-xl font-bold mb-4 border-b pb-2">Family Access Management</h2>
            <p className="text-sm text-gray-600 mb-4">Ask your family member to open this app and copy their "UID" from this settings page. Paste it below to grant them access.</p>
            
            <form onSubmit={handleAddUser} className="mb-6 bg-gray-50 p-4 rounded-lg border border-gray-200">
              <div className="mb-3">
                <input type="text" placeholder="Paste UID here..." className="w-full p-3 border rounded-lg" value={newUserId} onChange={e => setNewUserId(e.target.value)} required />
              </div>
              <div className="mb-3">
                <select className="w-full p-3 border rounded-lg" value={newUserRole} onChange={e => setNewUserRole(e.target.value)}>
                  <option value="viewer">Viewer (Read Only)</option>
                  <option value="admin">Admin (Can Edit/Bill)</option>
                </select>
              </div>
              <button type="submit" disabled={saving} className="w-full bg-indigo-600 text-white font-bold py-3 rounded-xl shadow">Add User</button>
            </form>

            <h3 className="font-bold text-gray-800 mb-3">Authorized Users</h3>
            <div className="space-y-3">
              {users.map(u => (
                <div key={u.id} className="flex justify-between items-center bg-gray-100 p-3 rounded-lg">
                  <div>
                    <div className="font-bold">{u.name} <span className="text-xs font-normal text-gray-500 uppercase ml-2 px-2 py-0.5 bg-gray-300 rounded">{u.role}</span></div>
                    <div className="text-xs text-gray-500 font-mono mt-1">{u.uid.substring(0,10)}...</div>
                  </div>
                  {u.uid !== currentUser.uid && (
                    <button onClick={() => handleRemoveUser(u.id)} className="text-red-600 text-sm font-bold bg-red-100 px-3 py-1 rounded">Remove</button>
                  )}
                </div>
              ))}
            </div>
          </div>
        </>
      )}
    </div>
  );
}
```
