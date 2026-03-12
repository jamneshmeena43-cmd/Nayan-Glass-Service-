import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInWithCustomToken, 
  signInAnonymously, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  addDoc, 
  updateDoc, 
  onSnapshot, 
  query, 
  orderBy,
  serverTimestamp
} from 'firebase/firestore';
import { 
  Menu, X, Phone, ShieldCheck, Star, Users, ArrowRight, 
  CheckCircle2, Plus, LayoutDashboard, Store, ClipboardList, 
  TrendingUp, Camera, MapPin, Ruler, MessageCircle, Info, Search, 
  Clock, Truck, IndianRupee, Loader2
} from 'lucide-react';

// --- FIREBASE CONFIGURATION ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'nayan-glass-service';

const COMMISSION_RATE = 0.05;

// Helper to render Firestore timestamp or number
function formatTimestamp(ts) {
  if (!ts) return '—';
  // Firestore Timestamp has toMillis()
  if (typeof ts.toMillis === 'function') {
    return new Date(ts.toMillis()).toLocaleString();
  }
  // If client used Date.now() numbers
  if (typeof ts === 'number') {
    return new Date(ts).toLocaleString();
  }
  // Fallback
  try {
    return new Date(ts).toLocaleString();
  } catch {
    return String(ts);
  }
}

// --- MAIN APP COMPONENT ---
export default function App() {
  const [view, setView] = useState('home'); 
  const [mobileMenu, setMobileMenu] = useState(false);
  const [requests, setRequests] = useState([]);
  const [vendors, setVendors] = useState([]);
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [notification, setNotification] = useState(null);

  const notify = (msg) => {
    setNotification(msg);
    setTimeout(() => setNotification(null), 3000);
  };

  const navigate = (to) => {
    setView(to);
    setMobileMenu(false);
    window.scrollTo(0, 0);
  };

  // 1. Auth Initialization
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth error:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  // 2. Data Synchronization (Real-time Firestore Listeners)
  useEffect(() => {
    if (!user) return;

    // Fetch Requests (Public Data Path) ordered by createdAt desc
    const requestsCol = collection(db, 'artifacts', appId, 'public', 'data', 'requests');
    const requestsQuery = query(requestsCol, orderBy('createdAt', 'desc'));
    const unsubRequests = onSnapshot(requestsQuery, (snapshot) => {
      const docs = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      setRequests(docs); // server-side ordered already
    }, (err) => console.error("Firestore Requests Error:", err));

    // Fetch Vendors (Public Data Path) ordered by createdAt
    const vendorsCol = collection(db, 'artifacts', appId, 'public', 'data', 'vendors');
    const vendorsQuery = query(vendorsCol, orderBy('createdAt', 'asc'));
    const unsubVendors = onSnapshot(vendorsQuery, (snapshot) => {
      const docs = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      setVendors(docs);
    }, (err) => console.error("Firestore Vendors Error:", err));

    return () => {
      try { unsubRequests(); } catch {}
      try { unsubVendors(); } catch {}
    };
  }, [user]);

  // --- ACTIONS ---
  const handleRequestSubmit = async (data) => {
    if (!user) return;
    try {
      const requestsCol = collection(db, 'artifacts', appId, 'public', 'data', 'requests');
      await addDoc(requestsCol, {
        ...data,
        status: 'Pending',
        vendorPrice: 0,
        totalPrice: 0,
        createdAt: serverTimestamp(),
        createdBy: user.uid
      });
      notify("Request Sent! We will call you in 30 mins.");
      navigate('home');
    } catch (err) {
      console.error("Request submit error:", err);
      notify("Error submitting request. Try again.");
    }
  };

  const handleVendorRegister = async (data) => {
    if (!user) return;
    try {
      const vendorsCol = collection(db, 'artifacts', appId, 'public', 'data', 'vendors');
      await addDoc(vendorsCol, {
        ...data,
        activeOrders: 0,
        createdAt: serverTimestamp()
      });
      notify("Application Received! Welcome to the team.");
      navigate('home');
    } catch (err) {
      console.error("Vendor register error:", err);
      notify("Error registering. Try again.");
    }
  };

  const handleUpdateStatus = async (orderId, newStatus, vPrice = 0) => {
    if (!user) return;
    try {
      const numericPrice = Number(vPrice) || 0;
      const finalPrice = numericPrice > 0 ? Math.round((numericPrice + (numericPrice * COMMISSION_RATE))) : 0;
      const orderDoc = doc(db, 'artifacts', appId, 'public', 'data', 'requests', orderId);
      await updateDoc(orderDoc, {
        status: newStatus,
        vendorPrice: numericPrice,
        totalPrice: finalPrice,
        updatedAt: serverTimestamp()
      });
      notify(`Order ${newStatus}!`);
    } catch (err) {
      console.error("Update status error:", err);
      notify("Failed to update order.");
    }
  };

  if (loading) {
    return (
      <div className="min-h-screen flex flex-col items-center justify-center bg-slate-50">
        <Loader2 className="animate-spin text-blue-600 mb-4" size={48} />
        <p className="font-bold text-slate-500">Connecting to Nayan Glass Cloud...</p>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-50 font-sans text-slate-900 selection:bg-blue-100">
      {/* Navigation */}
      <nav className="sticky top-0 z-50 bg-white/90 backdrop-blur-md shadow-sm border-b border-slate-100">
        <div className="max-w-7xl mx-auto px-4 h-16 flex items-center justify-between">
          <div className="flex items-center gap-2 cursor-pointer group" onClick={() => navigate('home')}>
            <div className="bg-blue-600 p-2 rounded-xl group-hover:rotate-6 transition-transform">
              <ShieldCheck className="text-white" size={24} />
            </div>
            <span className="font-bold text-xl tracking-tight text-blue-900">Nayan Glass</span>
          </div>

          <div className="hidden md:flex items-center gap-6">
            <button onClick={() => navigate('home')} className="text-slate-600 hover:text-blue-600 font-semibold transition-colors">होम (Home)</button>
            <button onClick={() => navigate('track')} className="text-slate-600 hover:text-blue-600 font-semibold transition-colors">Track Order</button>
            <button onClick={() => navigate('vendor')} className="text-slate-600 hover:text-blue-600 font-semibold transition-colors">पार्टनर बनें</button>
            <button 
              onClick={() => navigate('admin')} 
              className="bg-slate-100 text-slate-700 px-4 py-2 rounded-xl text-sm font-bold flex items-center gap-2 hover:bg-slate-200 transition-all"
            >
              <LayoutDashboard size={16} /> Admin
            </button>
            <button onClick={() => navigate('request')} className="bg-blue-600 text-white px-6 py-2.5 rounded-xl font-bold hover:bg-blue-700 transition-all shadow-lg shadow-blue-200 active:scale-95">
              Request Service
            </button>
          </div>

          <button className="md:hidden p-2 text-slate-600" onClick={() => setMobileMenu(!mobileMenu)}>
            {mobileMenu ? <X size={28} /> : <Menu size={28} />}
          </button>
        </div>
      </nav>

      {/* Mobile Menu */}
      {mobileMenu && (
        <div className="fixed inset-0 z-40 bg-white pt-20 px-6 flex flex-col gap-6 md:hidden animate-in slide-in-from-right">
          <button onClick={() => navigate('home')} className="text-2xl font-bold border-b pb-4 text-left">Home</button>
          <button onClick={() => navigate('request')} className="text-2xl font-bold border-b pb-4 text-left text-blue-600">Request Service</button>
          <button onClick={() => navigate('track')} className="text-2xl font-bold border-b pb-4 text-left">Track My Order</button>
          <button onClick={() => navigate('vendor')} className="text-2xl font-bold border-b pb-4 text-left">Become a Partner</button>
          <button onClick={() => navigate('admin')} className="text-lg font-bold text-slate-400 mt-auto mb-10 text-left">Admin Login</button>
        </div>
      )}

      {/* Notification Toast */}
      {notification && (
        <div className="fixed top-20 right-4 z-[100] bg-slate-900 text-white px-6 py-4 rounded-2xl shadow-2xl flex items-center gap-3 animate-in fade-in slide-in-from-top-4">
          <div className="bg-green-500 rounded-full p-1"><CheckCircle2 size={16} /></div>
          <span className="font-bold">{notification}</span>
        </div>
      )}

      <main className="pb-20">
        {view === 'home' && <HomePage onNavigate={navigate} />}
        {view === 'request' && <RequestPage onSubmit={handleRequestSubmit} />}
        {view === 'vendor' && <VendorOnboarding onRegister={handleVendorRegister} />}
        {view === 'track' && <TrackOrder requests={requests} />}
        {view === 'admin' && (
          <AdminDashboard 
            requests={requests} 
            vendors={vendors} 
            onUpdateStatus={handleUpdateStatus} 
          />
        )}
      </main>

      <Footer onNavigate={navigate} />
    </div>
  );
}

// --- PAGE COMPONENTS ---

function HomePage({ onNavigate }) {
  return (
    <div className="animate-in fade-in duration-700">
      <section className="relative bg-gradient-to-br from-blue-50 to-white pt-12 pb-24 px-4 overflow-hidden">
        <div className="max-w-7xl mx-auto flex flex-col lg:flex-row items-center gap-12">
          <div className="flex-1 text-center lg:text-left z-10">
            <div className="inline-flex items-center gap-2 bg-green-100 text-green-700 px-4 py-2 rounded-full text-sm font-bold mb-8">
              <ShieldCheck size={18} /> Verified Shopkeepers in your area
            </div>
            <h1 className="text-5xl lg:text-7xl font-black text-slate-900 leading-[1.1] mb-6">
              Fast & Trusted <br/>
              <span className="text-blue-600">Glass Service</span> <br/>
              Near You
            </h1>
            <p className="text-xl text-slate-600 mb-10 max-w-xl mx-auto lg:mx-0 leading-relaxed">
              खिड़की, दरवाज़ा और कस्टम ग्लास वर्क—अब घर बैठे कराएं। एक्सपर्ट दुकानदार, किफायती दाम।
            </p>
            <div className="flex flex-col sm:flex-row gap-4 justify-center lg:justify-start">
              <button 
                onClick={() => onNavigate('request')}
                className="bg-blue-600 text-white px-10 py-5 rounded-2xl text-xl font-bold flex items-center justify-center gap-3 hover:bg-blue-700 transition-all shadow-2xl shadow-blue-200 active:scale-95"
              >
                Request Service Now <ArrowRight size={24} />
              </button>
              <div className="flex flex-col items-center lg:items-start justify-center px-4 py-2 bg-white/50 rounded-2xl border border-slate-100">
                <div className="flex -space-x-3 mb-1">
                  {[1,2,3,4,5].map(i => (
                    <div key={i} className="w-10 h-10 rounded-full border-2 border-white bg-slate-200 overflow-hidden shadow-sm">
                        <img src={`https://api.dicebear.com/7.x/avataaars/svg?seed=${i+10}`} alt="user" />
                    </div>
                  ))}
                </div>
                <p className="text-sm font-bold text-slate-700">500+ Happy Customers</p>
              </div>
            </div>
          </div>
          
          <div className="flex-1 relative w-full max-w-xl">
            <div className="relative z-10 rounded-[2.5rem] overflow-hidden shadow-2xl border-[12px] border-white bg-white">
              <div className="grid grid-cols-2 h-72 lg:h-96">
                <div className="bg-slate-200 flex flex-col items-center justify-center relative p-4 text-center">
                  <span className="absolute top-4 left-4 bg-red-500 text-white text-[10px] px-3 py-1 rounded-full uppercase font-black">Before</span>
                  <div className="w-20 h-20 border-4 border-slate-400 opacity-20 rotate-45 mb-4"></div>
                  <p className="text-slate-400 font-bold text-sm">Broken & Dangerous</p>
                </div>
                <div className="bg-blue-50 flex flex-col items-center justify-center relative border-l-4 border-white p-4 text-center">
                  <span className="absolute top-4 right-4 bg-green-500 text-white text-[10px] px-3 py-1 rounded-full uppercase font-black">After</span>
                  <CheckCircle2 className="text-blue-600 mb-4" size={56} strokeWidth={3} />
                  <p className="text-blue-600 font-bold text-sm">Crystal Clear & Secure</p>
                </div>
              </div>
            </div>
            <div className="absolute -bottom-8 -right-8 md:right-0 bg-white p-6 rounded-[2rem] shadow-2xl border border-slate-100 flex items-center gap-4 z-20 animate-bounce">
              <div className="bg-blue-600 p-3 rounded-2xl text-white shadow-lg"><Phone size={24} fill="currentColor"/></div>
              <div>
                <p className="font-black text-slate-900 text-lg leading-tight">Quick Call</p>
                <p className="text-sm text-slate-500 font-medium">Callback in 15 mins</p>
              </div>
            </div>
          </div>
        </div>
      </section>

      <section className="py-24 max-w-7xl mx-auto px-4">
        <div className="text-center mb-16">
          <h2 className="text-4xl font-black mb-4">हमारी सेवाएं (Our Services)</h2>
          <p className="text-slate-500 text-lg max-w-2xl mx-auto">Expert glass solutions delivered at your doorstep by local professionals you can trust.</p>
        </div>
        <div className="grid grid-cols-2 md:grid-cols-4 gap-8">
          {[
            { name: "Window Work", icon: "🪟", color: "bg-blue-100 text-blue-600" },
            { name: "Glass Doors", icon: "🚪", color: "bg-green-100 text-green-600" },
            { name: "Custom Mirrors", icon: "🪞", color: "bg-purple-100 text-purple-600" },
            { name: "Table Tops", icon: "🧊", color: "bg-orange-100 text-orange-600" }
          ].map((s, idx) => (
            <div key={idx} className="bg-white p-10 rounded-[2.5rem] border border-slate-100 text-center hover:border-blue-400 hover:shadow-xl hover:-translate-y-2 transition-all cursor-pointer group">
              <div className={`text-6xl mb-6 mx-auto w-24 h-24 flex items-center justify-center rounded-3xl ${s.color} group-hover:scale-110 transition-transform`}>
                {s.icon}
              </div>
              <h3 className="font-black text-xl text-slate-800">{s.name}</h3>
              <p className="text-slate-400 mt-2 text-sm">Tap to Book</p>
            </div>
          ))}
        </div>
      </section>

      <section className="bg-slate-900 py-24 px-4 overflow-hidden relative">
        <div className="absolute top-0 right-0 w-64 h-64 bg-blue-600 opacity-10 rounded-full -mr-32 -mt-32"></div>
        <div className="max-w-7xl mx-auto grid grid-cols-2 md:grid-cols-4 gap-12 text-center text-white relative z-10">
          <div>
            <div className="text-5xl font-black mb-2 text-blue-400">500+</div>
            <div className="text-slate-400 font-bold">Orders Done</div>
          </div>
          <div>
            <div className="text-5xl font-black mb-2 text-green-400">4.9/5</div>
            <div className="text-slate-400 font-bold">User Rating</div>
          </div>
          <div>
            <div className="text-5xl font-black mb-2 text-purple-400">50+</div>
            <div className="text-slate-400 font-bold">Local Partners</div>
          </div>
          <div>
            <div className="text-5xl font-black mb-2 text-orange-400">0%</div>
            <div className="text-slate-400 font-bold">Fraud Risk</div>
          </div>
        </div>
      </section>
    </div>
  );
}

function RequestPage({ onSubmit }) {
  const [formData, setFormData] = useState({
    name: '', mobile: '', address: '', size: '', type: 'Clear Glass', desc: ''
  });

  return (
    <div className="max-w-2xl mx-auto px-4 py-16 animate-in slide-in-from-bottom-8">
      <div className="bg-white rounded-[2.5rem] shadow-2xl overflow-hidden border border-slate-100">
        <div className="bg-blue-600 p-10 text-white text-center">
          <h1 className="text-3xl font-black mb-2">Request Call Back</h1>
          <p className="text-blue-100 font-medium">अपनी जानकारी भरें, हम आपको 30 मिनट में कॉल करेंगे।</p>
        </div>
        <form className="p-10 space-y-6" onSubmit={(e) => { e.preventDefault(); onSubmit(formData); }}>
          <div className="grid md:grid-cols-2 gap-6">
            <div>
              <label className="block text-sm font-black text-slate-700 mb-2">आपका नाम (Full Name)</label>
              <input 
                required
                className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"
                placeholder="Ex: Rajesh Kumar"
                value={formData.name}
                onChange={e => setFormData({...formData, name: e.target.value})}
              />
            </div>
            <div>
              <label className="block text-sm font-black text-slate-700 mb-2">मोबाइल नंबर (Mobile)</label>
              <input 
                required
                type="tel"
                maxLength="10"
                className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"
                placeholder="10 digit number"
                value={formData.mobile}
                onChange={e => setFormData({...formData, mobile: e.target.value})}
              />
            </div>
          </div>

          <div>
            <label className="block text-sm font-black text-slate-700 mb-2">काम का पता (Address)</label>
            <input 
              required
              className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"
              placeholder="City, Area, House No."
              value={formData.address}
              onChange={e => setFormData({...formData, address: e.target.value})}
            />
          </div>

          <div className="grid md:grid-cols-2 gap-6">
            <div>
              <label className="block text-sm font-black text-slate-700 mb-2 flex items-center gap-2">
                <Ruler size={16} /> कांच का साइज (Size)
              </label>
              <input 
                className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"
                placeholder="Ex: 3ft x 4ft"
                value={formData.size}
                onChange={e => setFormData({...formData, size: e.target.value})}
              />
            </div>
            <div>
              <label className="block text-sm font-black text-slate-700 mb-2">कांच का प्रकार (Type)</label>
              <select 
                className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none bg-white font-medium"
                value={formData.type}
                onChange={e => setFormData({...formData, type: e.target.value})}
              >
                <option>Clear Glass (सादा कांच)</option>
                <option>Toughened Glass (मजबूत कांच)</option>
                <option>Frosted / Design (डिज़ाइनर)</option>
                <option>Mirror (शीशा)</option>
              </select>
            </div>
          </div>

          <button 
            type="submit"
            className="w-full bg-blue-600 text-white py-5 rounded-2xl font-black text-xl hover:bg-blue-700 shadow-xl shadow-blue-100 transition-all active:scale-95"
          >
            Request Call Back
          </button>
        </form>
      </div>
    </div>
  );
}

function TrackOrder({ requests }) {
  const [mobile, setMobile] = useState('');
  const [result, setResult] = useState(null);

  const handleSearch = () => {
    const m = (mobile || '').trim();
    const found = requests.filter(r => (r.mobile || '').trim() === m);
    setResult(found);
  };

  return (
    <div className="max-w-2xl mx-auto px-4 py-16">
      <div className="text-center mb-10">
        <h1 className="text-3xl font-black mb-4">Track Your Order</h1>
        <p className="text-slate-500">अपना मोबाइल नंबर डालकर ऑर्डर स्टेटस चेक करें।</p>
      </div>

      <div className="bg-white p-4 rounded-3xl shadow-xl flex gap-3 border border-slate-100 mb-10">
        <input 
          type="tel" 
          placeholder="Enter Registered Mobile" 
          className="flex-1 px-6 py-4 rounded-2xl border-none focus:ring-0 text-lg font-bold"
          value={mobile}
          onChange={e => setMobile(e.target.value)}
        />
        <button 
          onClick={handleSearch}
          className="bg-blue-600 text-white p-4 rounded-2xl hover:bg-blue-700 shadow-lg shadow-blue-200"
        >
          <Search size={24} />
        </button>
      </div>

      <div className="space-y-6">
        {result?.map(r => (
          <div key={r.id} className="bg-white p-8 rounded-[2rem] shadow-lg border-l-8 border-blue-600 animate-in slide-in-from-top-4">
            <div className="flex justify-between items-start mb-6">
              <div>
                <p className="text-xs font-black text-slate-400 uppercase tracking-widest">Order ID</p>
                <h3 className="text-xl font-black">{r.id.slice(-6).toUpperCase()}</h3>
              </div>
              <span className={`px-4 py-2 rounded-xl text-sm font-black uppercase tracking-tighter ${
                r.status === 'Pending' ? 'bg-orange-100 text-orange-600' : 
                r.status === 'Assigned' ? 'bg-blue-100 text-blue-600' : 'bg-green-100 text-green-600'
              }`}>
                {r.status}
              </span>
            </div>
            
            <div className="space-y-4 mb-8">
               <div className="flex items-center gap-3 text-slate-600 font-bold">
                  <Clock size={18} className="text-blue-500" />
                  <span>Received: {formatTimestamp(r.createdAt)}</span>
               </div>
               <div className="flex items-center gap-3 text-slate-600 font-bold">
                  <Truck size={18} className="text-blue-500" />
                  <span>{r.status === 'Pending' ? 'Admin is finding a shopkeeper...' : 'Assigned to verified local shop.'}</span>
               </div>
            </div>

            {r.totalPrice > 0 && (
              <div className="bg-slate-50 p-6 rounded-2xl flex justify-between items-center border border-slate-100">
                <span className="font-bold text-slate-500">Service Price:</span>
                <span className="text-2xl font-black text-slate-900">₹{Number(r.totalPrice).toLocaleString()}</span>
              </div>
            )}
          </div>
        ))}
        {result && result.length === 0 && <p className="text-center font-bold text-red-500">No order found with this number.</p>}
      </div>
    </div>
  );
}

function AdminDashboard({ requests, vendors, onUpdateStatus }) {
  const [selectedRequest, setSelectedRequest] = useState(null);
  const [activeTab, setActiveTab] = useState('orders');
  const [priceInput, setPriceInput] = useState('');

  useEffect(() => {
    setPriceInput(selectedRequest ? String(selectedRequest.vendorPrice || '') : '');
  }, [selectedRequest]);

  const totalRevenue = requests.reduce((acc, curr) => acc + (Number(curr.totalPrice) || 0), 0);
  const totalCommission = requests.reduce((acc, curr) => acc + (((Number(curr.totalPrice) || 0) - (Number(curr.vendorPrice) || 0)) || 0), 0);

  return (
    <div className="max-w-7xl mx-auto px-4 py-12">
      <div className="flex flex-col md:flex-row md:items-center justify-between mb-12 gap-6">
        <h1 className="text-3xl font-black flex items-center gap-3">
          <LayoutDashboard className="text-blue-600" /> Control Center
        </h1>
        <div className="flex bg-white p-1.5 rounded-2xl shadow-sm border border-slate-100">
          <button onClick={() => setActiveTab('orders')} className={`px-6 py-3 rounded-xl font-black transition-all ${activeTab === 'orders' ? 'bg-blue-600 text-white' : 'text-slate-500'}`}>Orders</button>
          <button onClick={() => setActiveTab('shops')} className={`px-6 py-3 rounded-xl font-black transition-all ${activeTab === 'shops' ? 'bg-blue-600 text-white' : 'text-slate-500'}`}>Partners</button>
        </div>
      </div>

      <div className="grid md:grid-cols-4 gap-6 mb-12">
        <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">
          <p className="text-slate-400 text-xs font-black uppercase mb-2">New Leads</p>
          <p className="text-4xl font-black">{requests.filter(r=>r.status === 'Pending').length}</p>
        </div>
        <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">
          <p className="text-slate-400 text-xs font-black uppercase mb-2">Active Partners</p>
          <p className="text-4xl font-black text-blue-600">{vendors.length}</p>
        </div>
        <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">
          <p className="text-slate-400 text-xs font-black uppercase mb-2">Total Volume</p>
          <p className="text-4xl font-black text-green-600">₹{totalRevenue.toLocaleString()}</p>
        </div>
        <div className="bg-blue-600 p-8 rounded-3xl shadow-xl text-white">
          <p className="text-blue-200 text-xs font-black uppercase mb-2">My Commission (5%)</p>
          <p className="text-4xl font-black text-white">₹{totalCommission.toFixed(2)}</p>
        </div>
      </div>

      {activeTab === 'orders' ? (
        <div className="bg-white rounded-[2rem] shadow-xl border border-slate-100 overflow-x-auto">
          <table className="w-full text-left min-w-[800px]">
            <thead className="bg-slate-50/50 border-b">
              <tr>
                <th className="p-6 font-black text-slate-400 text-xs uppercase">Customer</th>
                <th className="p-6 font-black text-slate-400 text-xs uppercase">Work Details</th>
                <th className="p-6 font-black text-slate-400 text-xs uppercase">Price Breakdown</th>
                <th className="p-6 font-black text-slate-400 text-xs uppercase">Status</th>
                <th className="p-6 font-black text-slate-400 text-xs uppercase text-right">Action</th>
              </tr>
            </thead>
            <tbody>
              {requests.map(r => (
                <tr key={r.id} className="border-b last:border-0 hover:bg-slate-50/50">
                  <td className="p-6">
                    <p className="font-black text-slate-900">{r.name}</p>
                    <p className="text-sm font-bold text-slate-400">{r.mobile}</p>
                    <p className="text-xs text-slate-400 mt-1 truncate max-w-[200px]">{r.address}</p>
                  </td>
                  <td className="p-6">
                    <p className="font-bold text-slate-700">{r.type}</p>
                    <p className="text-xs font-bold text-blue-500 uppercase">{r.size || 'No Size'}</p>
                  </td>
                  <td className="p-6">
                    {Number(r.vendorPrice) > 0 ? (
                      <div className="space-y-1">
                        <p className="text-xs text-slate-400">Shop: ₹{Number(r.vendorPrice).toLocaleString()}</p>
                        <p className="text-xs text-green-600 font-bold">Comm: ₹{(Number(r.totalPrice || 0) - Number(r.vendorPrice || 0)).toFixed(0)}</p>
                        <p className="text-sm font-black text-slate-900 border-t pt-1">Total: ₹{Number(r.totalPrice).toLocaleString()}</p>
                      </div>
                    ) : (
                      <span className="text-slate-300 italic text-sm">Waiting for price...</span>
                    )}
                  </td>
                  <td className="p-6">
                    <span className={`px-4 py-2 rounded-xl text-[10px] font-black uppercase ${
                      r.status === 'Pending' ? 'bg-orange-100 text-orange-600' : 
                      r.status === 'Assigned' ? 'bg-blue-100 text-blue-600' : 'bg-green-100 text-green-600'
                    }`}>
                      {r.status}
                    </span>
                  </td>
                  <td className="p-6 text-right">
                    <button 
                      onClick={() => setSelectedRequest(r)}
                      className="bg-slate-900 text-white px-5 py-2 rounded-xl font-bold text-sm hover:scale-105 transition-transform"
                    >
                      Manage
                    </button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
          {requests.length === 0 && <div className="p-20 text-center font-bold text-slate-300">No requests yet.</div>}
        </div>
      ) : (
        <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-8">
          {vendors.map(v => (
            <div key={v.id} className="bg-white p-8 rounded-[2rem] border border-slate-100 shadow-lg relative group overflow-hidden">
               <div className="absolute top-0 right-0 w-24 h-24 bg-blue-50 rounded-full -mr-12 -mt-12 group-hover:scale-110 transition-transform"></div>
               <h3 className="text-xl font-black mb-1">{v.shopName}</h3>
               <p className="text-slate-400 font-bold mb-4">{v.owner} • {v.location}</p>
               <div className="flex gap-4 items-center">
                  <div className="bg-slate-50 px-4 py-2 rounded-xl text-xs font-black text-slate-500">Exp: {v.experience}</div>
                  <div className="bg-blue-50 px-4 py-2 rounded-xl text-xs font-black text-blue-600">{v.activeOrders} Active</div>
               </div>
               <button className="mt-6 w-full bg-slate-100 text-slate-900 py-3 rounded-xl font-black flex items-center justify-center gap-2 hover:bg-slate-900 hover:text-white transition-all">
                  <Phone size={18} /> Contact Shop
               </button>
            </div>
          ))}
          {vendors.length === 0 && <div className="col-span-full py-20 text-center font-bold text-slate-300">No partner shops registered.</div>}
        </div>
      )}

      {selectedRequest && (
        <div className="fixed inset-0 z-[100] bg-slate-900/60 backdrop-blur-md flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-lg rounded-[2.5rem] shadow-2xl overflow-hidden animate-in zoom-in-95 duration-200">
            <div className="p-10 border-b flex justify-between items-center bg-blue-600 text-white">
              <h3 className="font-black text-2xl">Manage Order</h3>
              <button onClick={() => setSelectedRequest(null)} className="hover:rotate-90 transition-transform"><X /></button>
            </div>
            <div className="p-10 space-y-8">
              <div>
                <label className="block text-xs font-black uppercase text-slate-400 mb-3">Assign Shopkeeper</label>
                <select className="w-full p-4 rounded-2xl border border-slate-200 bg-white font-bold" defaultValue="">
                  <option value="" disabled>Select shop / Self Service</option>
                  {vendors.map(v => <option key={v.id} value={v.id}>{v.shopName}</option>)}
                  <option value="self">Self Service (Nayan Glass)</option>
                </select>
              </div>

              <div>
                <label className="block text-xs font-black uppercase text-slate-400 mb-3">Shopkeeper Price (₹)</label>
                <div className="relative">
                  <IndianRupee className="absolute left-4 top-4 text-slate-300" size={20} />
                  <input 
                    id="price-input"
                    type="number" 
                    className="w-full pl-12 p-4 rounded-2xl border border-slate-200 font-black text-xl" 
                    placeholder="Enter basic work cost" 
                    value={priceInput}
                    onChange={(e) => setPriceInput(e.target.value)}
                  />
                  <p className="mt-2 text-xs text-green-600 font-bold italic">+ 5% commission will be added for customer automatically</p>
                </div>
              </div>

              <div className="grid grid-cols-2 gap-4 pt-4">
                <button 
                  onClick={() => {
                    const vPrice = Number(priceInput) || 0;
                    onUpdateStatus(selectedRequest.id, 'Assigned', vPrice);
                    setSelectedRequest(null);
                  }}
                  className="bg-slate-100 text-slate-900 py-5 rounded-2xl font-black hover:bg-slate-200"
                >
                  Save Status
                </button>
                <button 
                  onClick={() => {
                    const vPrice = Number(priceInput) || 0;
                    onUpdateStatus(selectedRequest.id, 'Completed', vPrice);
                    setSelectedRequest(null);
                  }}
                  className="bg-green-600 text-white py-5 rounded-2xl font-black hover:bg-green-700 shadow-xl shadow-green-100"
                >
                  Mark Finished
                </button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

function VendorOnboarding({ onRegister }) {
  const [formData, setFormData] = useState({ shopName: '', owner: '', mobile: '', location: '', experience: '1-3 Years' });

  return (
    <div className="max-w-3xl mx-auto px-4 py-16 animate-in slide-in-from-right-8">
       <div className="bg-white rounded-[2.5rem] shadow-2xl border border-slate-100 p-12 text-center">
          <div className="bg-green-100 w-24 h-24 rounded-3xl flex items-center justify-center text-green-600 mx-auto mb-8">
            <Store size={48} strokeWidth={2.5}/>
          </div>
          <h1 className="text-4xl font-black mb-4">Partner with Nayan Glass</h1>
          <p className="text-slate-500 text-lg mb-12">ग्लास वर्क का काम करते हैं? हमारे साथ जुड़ें और रोज़ाना 10+ नए आर्डर पाएं।</p>

          <form className="grid md:grid-cols-2 gap-8 text-left" onSubmit={(e) => { e.preventDefault(); onRegister(formData); }}>
            <div className="md:col-span-2 bg-blue-50 p-6 rounded-3xl border border-blue-100 flex items-start gap-4 mb-4">
               <Info className="text-blue-500 mt-1 shrink-0" size={24} />
               <div>
                  <h4 className="font-black text-blue-900">How it works?</h4>
                  <p className="text-sm text-blue-700">We send customers. You set the price. We take 5% commission. No monthly fees.</p>
               </div>
            </div>
            
            <div className="space-y-6">
              <div>
                <label className="block text-sm font-black mb-2 text-slate-700">दुकान का नाम (Shop Name)</label>
                <input required className="w-full p-4 rounded-2xl border border-slate-200" placeholder="Ganesh Glass House" onChange={e => setFormData({...formData, shopName: e.target.value})} />
              </div>
              <div>
                <label className="block text-sm font-black mb-2 text-slate-700">मोबाइल नंबर (Mobile)</label>
                <input required type="tel" className="w-full p-4 rounded-2xl border border-slate-200" placeholder="99999-XXXXX" onChange={e => setFormData({...formData, mobile: e.target.value})} />
              </div>
            </div>
            <div className="space-y-6">
              <div>
                <label className="block text-sm font-black mb-2 text-slate-700">अनुभव (Experience)</label>
                <select className="w-full p-4 rounded-2xl border border-slate-200 bg-white" onChange={e => setFormData({...formData, experience: e.target.value})}>
                  <option>1-3 Years</option>
                  <option>3-7 Years</option>
                  <option>7+ Years</option>
                </select>
              </div>
              <div>
                <label className="block text-sm font-black mb-2 text-slate-700">शहर / जिला (City)</label>
                <input required className="w-full p-4 rounded-2xl border border-slate-200" placeholder="Ex: Lucknow" onChange={e => setFormData({...formData, location: e.target.value})} />
              </div>
            </div>
            <button type="submit" className="md:col-span-2 bg-slate-900 text-white py-5 rounded-2xl font-black text-xl hover:bg-black transition-all shadow-2xl">
              Become a Partner Now
            </button>
          </form>
       </div>
    </div>
  );
}

function Footer({ onNavigate }) {
  return (
    <footer className="bg-slate-900 text-slate-400 py-20 px-4">
      <div className="max-w-7xl mx-auto grid grid-cols-1 md:grid-cols-4 gap-12">
        <div className="md:col-span-2">
          <div className="flex items-center gap-2 text-white mb-6">
            <ShieldCheck size={32} className="text-blue-500" />
            <span className="font-black text-2xl tracking-tighter uppercase">Nayan Glass Service</span>
          </div>
          <p className="text-lg leading-relaxed max-w-sm mb-8">भारत की सबसे भरोसेमंद ग्लास सर्विस प्लेटफॉर्म। सुरक्षित काम, सीधा दुकानदार से संपर्क।</p>
          <div className="flex gap-4">
             <div className="w-12 h-12 bg-slate-800 rounded-xl flex items-center justify-center text-white cursor-pointer hover:bg-blue-600 transition-colors"><Phone size={20} /></div>
             <div className="w-12 h-12 bg-slate-800 rounded-xl flex items-center justify-center text-white cursor-pointer hover:bg-blue-600 transition-colors"><MessageCircle size={20} /></div>
          </div>
        </div>
        <div>
          <h4 className="text-white font-black mb-6 uppercase tracking-widest text-xs">Navigation</h4>
          <ul className="space-y-4 font-bold">
            <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('home')}>Home</li>
            <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('request')}>Service Booking</li>
            <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('track')}>Track Order</li>
            <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('vendor')}>Partner Login</li>
          </ul>
        </div>
        <div>
          <h4 className="text-white font-black mb-6 uppercase tracking-widest text-xs">Working Areas</h4>
          <ul className="space-y-4 font-bold">
            <li>Noida / Greater Noida</li>
            <li>New Delhi / Ghaziabad</li>
            <li>Lucknow / Kanpur</li>
            <li>Mumbai / Pune</li>
          </ul>
        </div>
      </div>
      <div className="max-w-7xl mx-auto mt-20 pt-8 border-t border-slate-800 text-sm text-center font-bold">
        © 2024 Nayan Glass Service. Made for India 🇮🇳
      </div>
    </footer>
  );
}
