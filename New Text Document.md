import React, { useState, useEffect } from 'react';

import { initializeApp } from 'firebase/app';

import { 

&nbsp; getAuth, 

&nbsp; signInWithCustomToken, 

1. &nbsp; signInAnonymously, 

&nbsp; onAuthStateChanged 

} from 'firebase/auth';

import { 

&nbsp; getFirestore, 

&nbsp; collection, 

&nbsp; doc, 

&nbsp; addDoc, 

&nbsp; updateDoc, 

&nbsp; onSnapshot, 

&nbsp; query 

} from 'firebase/firestore';

import { 

&nbsp; Menu, X, Phone, ShieldCheck, Star, Users, ArrowRight, 

&nbsp; CheckCircle2, Plus, LayoutDashboard, Store, ClipboardList, 

&nbsp; TrendingUp, Camera, MapPin, Ruler, MessageCircle, Info, Search, 

&nbsp; Clock, Truck, IndianRupee, Loader2

} from 'lucide-react';



// --- FIREBASE CONFIGURATION ---

const firebaseConfig = JSON.parse(\_\_firebase\_config);

const app = initializeApp(firebaseConfig);

const auth = getAuth(app);

const db = getFirestore(app);

const appId = typeof \_\_app\_id !== 'undefined' ? \_\_app\_id : 'nayan-glass-service';



const COMMISSION\_RATE = 0.05;



// --- MAIN APP COMPONENT ---

export default function App() {

&nbsp; const \[view, setView] = useState('home'); 

&nbsp; const \[mobileMenu, setMobileMenu] = useState(false);

&nbsp; const \[requests, setRequests] = useState(\[]);

&nbsp; const \[vendors, setVendors] = useState(\[]);

&nbsp; const \[user, setUser] = useState(null);

&nbsp; const \[loading, setLoading] = useState(true);

&nbsp; const \[notification, setNotification] = useState(null);



&nbsp; const notify = (msg) => {

&nbsp;   setNotification(msg);

&nbsp;   setTimeout(() => setNotification(null), 3000);

&nbsp; };



&nbsp; const navigate = (to) => {

&nbsp;   setView(to);

&nbsp;   setMobileMenu(false);

&nbsp;   window.scrollTo(0, 0);

&nbsp; };



&nbsp; // 1. Auth Initialization

&nbsp; useEffect(() => {

&nbsp;   const initAuth = async () => {

&nbsp;     try {

&nbsp;       if (typeof \_\_initial\_auth\_token !== 'undefined' \&\& \_\_initial\_auth\_token) {

&nbsp;         await signInWithCustomToken(auth, \_\_initial\_auth\_token);

&nbsp;       } else {

&nbsp;         await signInAnonymously(auth);

&nbsp;       }

&nbsp;     } catch (err) {

&nbsp;       console.error("Auth error:", err);

&nbsp;     }

&nbsp;   };

&nbsp;   initAuth();

&nbsp;   const unsubscribe = onAuthStateChanged(auth, (u) => {

&nbsp;     setUser(u);

&nbsp;     setLoading(false);

&nbsp;   });

&nbsp;   return () => unsubscribe();

&nbsp; }, \[]);



&nbsp; // 2. Data Synchronization (Real-time Firestore Listeners)

&nbsp; useEffect(() => {

&nbsp;   if (!user) return;



&nbsp;   // Fetch Requests (Public Data Path)

&nbsp;   const requestsCol = collection(db, 'artifacts', appId, 'public', 'data', 'requests');

&nbsp;   const unsubRequests = onSnapshot(requestsCol, (snapshot) => {

&nbsp;     const docs = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));

&nbsp;     // Sort locally: newest first

&nbsp;     setRequests(docs.sort((a, b) => (b.createdAt || 0) - (a.createdAt || 0)));

&nbsp;   }, (err) => console.error("Firestore Requests Error:", err));



&nbsp;   // Fetch Vendors (Public Data Path)

&nbsp;   const vendorsCol = collection(db, 'artifacts', appId, 'public', 'data', 'vendors');

&nbsp;   const unsubVendors = onSnapshot(vendorsCol, (snapshot) => {

&nbsp;     const docs = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));

&nbsp;     setVendors(docs);

&nbsp;   }, (err) => console.error("Firestore Vendors Error:", err));



&nbsp;   return () => {

&nbsp;     unsubRequests();

&nbsp;     unsubVendors();

&nbsp;   };

&nbsp; }, \[user]);



&nbsp; // --- ACTIONS ---

&nbsp; const handleRequestSubmit = async (data) => {

&nbsp;   if (!user) return;

&nbsp;   try {

&nbsp;     const requestsCol = collection(db, 'artifacts', appId, 'public', 'data', 'requests');

&nbsp;     await addDoc(requestsCol, {

&nbsp;       ...data,

&nbsp;       status: 'Pending',

&nbsp;       vendorPrice: 0,

&nbsp;       totalPrice: 0,

&nbsp;       createdAt: Date.now(),

&nbsp;       createdBy: user.uid

&nbsp;     });

&nbsp;     notify("Request Sent! We will call you in 30 mins.");

&nbsp;     navigate('home');

&nbsp;   } catch (err) {

&nbsp;     notify("Error submitting request. Try again.");

&nbsp;   }

&nbsp; };



&nbsp; const handleVendorRegister = async (data) => {

&nbsp;   if (!user) return;

&nbsp;   try {

&nbsp;     const vendorsCol = collection(db, 'artifacts', appId, 'public', 'data', 'vendors');

&nbsp;     await addDoc(vendorsCol, {

&nbsp;       ...data,

&nbsp;       activeOrders: 0,

&nbsp;       createdAt: Date.now()

&nbsp;     });

&nbsp;     notify("Application Received! Welcome to the team.");

&nbsp;     navigate('home');

&nbsp;   } catch (err) {

&nbsp;     notify("Error registering. Try again.");

&nbsp;   }

&nbsp; };



&nbsp; const handleUpdateStatus = async (orderId, newStatus, vPrice = 0) => {

&nbsp;   if (!user) return;

&nbsp;   try {

&nbsp;     const finalPrice = vPrice > 0 ? vPrice + (vPrice \* COMMISSION\_RATE) : 0;

&nbsp;     const orderDoc = doc(db, 'artifacts', appId, 'public', 'data', 'requests', orderId);

&nbsp;     await updateDoc(orderDoc, {

&nbsp;       status: newStatus,

&nbsp;       vendorPrice: vPrice,

&nbsp;       totalPrice: finalPrice,

&nbsp;       updatedAt: Date.now()

&nbsp;     });

&nbsp;     notify(`Order ${newStatus}!`);

&nbsp;   } catch (err) {

&nbsp;     notify("Failed to update order.");

&nbsp;   }

&nbsp; };



&nbsp; if (loading) {

&nbsp;   return (

&nbsp;     <div className="min-h-screen flex flex-col items-center justify-center bg-slate-50">

&nbsp;       <Loader2 className="animate-spin text-blue-600 mb-4" size={48} />

&nbsp;       <p className="font-bold text-slate-500">Connecting to Nayan Glass Cloud...</p>

&nbsp;     </div>

&nbsp;   );

&nbsp; }



&nbsp; return (

&nbsp;   <div className="min-h-screen bg-slate-50 font-sans text-slate-900 selection:bg-blue-100">

&nbsp;     {/\* Navigation \*/}

&nbsp;     <nav className="sticky top-0 z-50 bg-white/90 backdrop-blur-md shadow-sm border-b border-slate-100">

&nbsp;       <div className="max-w-7xl mx-auto px-4 h-16 flex items-center justify-between">

&nbsp;         <div className="flex items-center gap-2 cursor-pointer group" onClick={() => navigate('home')}>

&nbsp;           <div className="bg-blue-600 p-2 rounded-xl group-hover:rotate-6 transition-transform">

&nbsp;             <ShieldCheck className="text-white" size={24} />

&nbsp;           </div>

&nbsp;           <span className="font-bold text-xl tracking-tight text-blue-900">Nayan Glass</span>

&nbsp;         </div>



&nbsp;         <div className="hidden md:flex items-center gap-6">

&nbsp;           <button onClick={() => navigate('home')} className="text-slate-600 hover:text-blue-600 font-semibold transition-colors">होम (Home)</button>

&nbsp;           <button onClick={() => navigate('track')} className="text-slate-600 hover:text-blue-600 font-semibold transition-colors">Track Order</button>

&nbsp;           <button onClick={() => navigate('vendor')} className="text-slate-600 hover:text-blue-600 font-semibold transition-colors">पार्टनर बनें</button>

&nbsp;           <button 

&nbsp;             onClick={() => navigate('admin')} 

&nbsp;             className="bg-slate-100 text-slate-700 px-4 py-2 rounded-xl text-sm font-bold flex items-center gap-2 hover:bg-slate-200 transition-all"

&nbsp;           >

&nbsp;             <LayoutDashboard size={16} /> Admin

&nbsp;           </button>

&nbsp;           <button onClick={() => navigate('request')} className="bg-blue-600 text-white px-6 py-2.5 rounded-xl font-bold hover:bg-blue-700 transition-all shadow-lg shadow-blue-200 active:scale-95">

&nbsp;             Request Service

&nbsp;           </button>

&nbsp;         </div>



&nbsp;         <button className="md:hidden p-2 text-slate-600" onClick={() => setMobileMenu(!mobileMenu)}>

&nbsp;           {mobileMenu ? <X size={28} /> : <Menu size={28} />}

&nbsp;         </button>

&nbsp;       </div>

&nbsp;     </nav>



&nbsp;     {/\* Mobile Menu \*/}

&nbsp;     {mobileMenu \&\& (

&nbsp;       <div className="fixed inset-0 z-40 bg-white pt-20 px-6 flex flex-col gap-6 md:hidden animate-in slide-in-from-right">

&nbsp;         <button onClick={() => navigate('home')} className="text-2xl font-bold border-b pb-4 text-left">Home</button>

&nbsp;         <button onClick={() => navigate('request')} className="text-2xl font-bold border-b pb-4 text-left text-blue-600">Request Service</button>

&nbsp;         <button onClick={() => navigate('track')} className="text-2xl font-bold border-b pb-4 text-left">Track My Order</button>

&nbsp;         <button onClick={() => navigate('vendor')} className="text-2xl font-bold border-b pb-4 text-left">Become a Partner</button>

&nbsp;         <button onClick={() => navigate('admin')} className="text-lg font-bold text-slate-400 mt-auto mb-10 text-left">Admin Login</button>

&nbsp;       </div>

&nbsp;     )}



&nbsp;     {/\* Notification Toast \*/}

&nbsp;     {notification \&\& (

&nbsp;       <div className="fixed top-20 right-4 z-\[100] bg-slate-900 text-white px-6 py-4 rounded-2xl shadow-2xl flex items-center gap-3 animate-in fade-in slide-in-from-top-4">

&nbsp;         <div className="bg-green-500 rounded-full p-1"><CheckCircle2 size={16} /></div>

&nbsp;         <span className="font-bold">{notification}</span>

&nbsp;       </div>

&nbsp;     )}



&nbsp;     <main className="pb-20">

&nbsp;       {view === 'home' \&\& <HomePage onNavigate={navigate} />}

&nbsp;       {view === 'request' \&\& <RequestPage onSubmit={handleRequestSubmit} />}

&nbsp;       {view === 'vendor' \&\& <VendorOnboarding onRegister={handleVendorRegister} />}

&nbsp;       {view === 'track' \&\& <TrackOrder requests={requests} />}

&nbsp;       {view === 'admin' \&\& (

&nbsp;         <AdminDashboard 

&nbsp;           requests={requests} 

&nbsp;           vendors={vendors} 

&nbsp;           onUpdateStatus={handleUpdateStatus} 

&nbsp;         />

&nbsp;       )}

&nbsp;     </main>



&nbsp;     <Footer onNavigate={navigate} />

&nbsp;   </div>

&nbsp; );

}



// --- PAGE COMPONENTS ---



function HomePage({ onNavigate }) {

&nbsp; return (

&nbsp;   <div className="animate-in fade-in duration-700">

&nbsp;     <section className="relative bg-gradient-to-br from-blue-50 to-white pt-12 pb-24 px-4 overflow-hidden">

&nbsp;       <div className="max-w-7xl mx-auto flex flex-col lg:flex-row items-center gap-12">

&nbsp;         <div className="flex-1 text-center lg:text-left z-10">

&nbsp;           <div className="inline-flex items-center gap-2 bg-green-100 text-green-700 px-4 py-2 rounded-full text-sm font-bold mb-8">

&nbsp;             <ShieldCheck size={18} /> Verified Shopkeepers in your area

&nbsp;           </div>

&nbsp;           <h1 className="text-5xl lg:text-7xl font-black text-slate-900 leading-\[1.1] mb-6">

&nbsp;             Fast \& Trusted <br/>

&nbsp;             <span className="text-blue-600">Glass Service</span> <br/>

&nbsp;             Near You

&nbsp;           </h1>

&nbsp;           <p className="text-xl text-slate-600 mb-10 max-w-xl mx-auto lg:mx-0 leading-relaxed">

&nbsp;             खिड़की, दरवाज़ा और कस्टम ग्लास वर्क—अब घर बैठे कराएं। एक्सपर्ट दुकानदार, किफायती दाम।

&nbsp;           </p>

&nbsp;           <div className="flex flex-col sm:flex-row gap-4 justify-center lg:justify-start">

&nbsp;             <button 

&nbsp;               onClick={() => onNavigate('request')}

&nbsp;               className="bg-blue-600 text-white px-10 py-5 rounded-2xl text-xl font-bold flex items-center justify-center gap-3 hover:bg-blue-700 transition-all shadow-2xl shadow-blue-200 active:scale-95"

&nbsp;             >

&nbsp;               Request Service Now <ArrowRight size={24} />

&nbsp;             </button>

&nbsp;             <div className="flex flex-col items-center lg:items-start justify-center px-4 py-2 bg-white/50 rounded-2xl border border-slate-100">

&nbsp;               <div className="flex -space-x-3 mb-1">

&nbsp;                 {\[1,2,3,4,5].map(i => (

&nbsp;                   <div key={i} className="w-10 h-10 rounded-full border-2 border-white bg-slate-200 overflow-hidden shadow-sm">

&nbsp;                       <img src={`https://api.dicebear.com/7.x/avataaars/svg?seed=${i+10}`} alt="user" />

&nbsp;                   </div>

&nbsp;                 ))}

&nbsp;               </div>

&nbsp;               <p className="text-sm font-bold text-slate-700">500+ Happy Customers</p>

&nbsp;             </div>

&nbsp;           </div>

&nbsp;         </div>

&nbsp;         

&nbsp;         <div className="flex-1 relative w-full max-w-xl">

&nbsp;           <div className="relative z-10 rounded-\[2.5rem] overflow-hidden shadow-2xl border-\[12px] border-white bg-white">

&nbsp;             <div className="grid grid-cols-2 h-72 lg:h-96">

&nbsp;               <div className="bg-slate-200 flex flex-col items-center justify-center relative p-4 text-center">

&nbsp;                 <span className="absolute top-4 left-4 bg-red-500 text-white text-\[10px] px-3 py-1 rounded-full uppercase font-black">Before</span>

&nbsp;                 <div className="w-20 h-20 border-4 border-slate-400 opacity-20 rotate-45 mb-4"></div>

&nbsp;                 <p className="text-slate-400 font-bold text-sm">Broken \& Dangerous</p>

&nbsp;               </div>

&nbsp;               <div className="bg-blue-50 flex flex-col items-center justify-center relative border-l-4 border-white p-4 text-center">

&nbsp;                 <span className="absolute top-4 right-4 bg-green-500 text-white text-\[10px] px-3 py-1 rounded-full uppercase font-black">After</span>

&nbsp;                 <CheckCircle2 className="text-blue-600 mb-4" size={56} strokeWidth={3} />

&nbsp;                 <p className="text-blue-600 font-bold text-sm">Crystal Clear \& Secure</p>

&nbsp;               </div>

&nbsp;             </div>

&nbsp;           </div>

&nbsp;           <div className="absolute -bottom-8 -right-8 md:right-0 bg-white p-6 rounded-\[2rem] shadow-2xl border border-slate-100 flex items-center gap-4 z-20 animate-bounce">

&nbsp;             <div className="bg-blue-600 p-3 rounded-2xl text-white shadow-lg"><Phone size={24} fill="currentColor"/></div>

&nbsp;             <div>

&nbsp;               <p className="font-black text-slate-900 text-lg leading-tight">Quick Call</p>

&nbsp;               <p className="text-sm text-slate-500 font-medium">Callback in 15 mins</p>

&nbsp;             </div>

&nbsp;           </div>

&nbsp;         </div>

&nbsp;       </div>

&nbsp;     </section>



&nbsp;     <section className="py-24 max-w-7xl mx-auto px-4">

&nbsp;       <div className="text-center mb-16">

&nbsp;         <h2 className="text-4xl font-black mb-4">हमारी सेवाएं (Our Services)</h2>

&nbsp;         <p className="text-slate-500 text-lg max-w-2xl mx-auto">Expert glass solutions delivered at your doorstep by local professionals you can trust.</p>

&nbsp;       </div>

&nbsp;       <div className="grid grid-cols-2 md:grid-cols-4 gap-8">

&nbsp;         {\[

&nbsp;           { name: "Window Work", icon: "🪟", color: "bg-blue-100 text-blue-600" },

&nbsp;           { name: "Glass Doors", icon: "🚪", color: "bg-green-100 text-green-600" },

&nbsp;           { name: "Custom Mirrors", icon: "🪞", color: "bg-purple-100 text-purple-600" },

&nbsp;           { name: "Table Tops", icon: "🧊", color: "bg-orange-100 text-orange-600" }

&nbsp;         ].map((s, idx) => (

&nbsp;           <div key={idx} className="bg-white p-10 rounded-\[2.5rem] border border-slate-100 text-center hover:border-blue-400 hover:shadow-xl hover:-translate-y-2 transition-all cursor-pointer group">

&nbsp;             <div className={`text-6xl mb-6 mx-auto w-24 h-24 flex items-center justify-center rounded-3xl ${s.color} group-hover:scale-110 transition-transform`}>

&nbsp;               {s.icon}

&nbsp;             </div>

&nbsp;             <h3 className="font-black text-xl text-slate-800">{s.name}</h3>

&nbsp;             <p className="text-slate-400 mt-2 text-sm">Tap to Book</p>

&nbsp;           </div>

&nbsp;         ))}

&nbsp;       </div>

&nbsp;     </section>



&nbsp;     <section className="bg-slate-900 py-24 px-4 overflow-hidden relative">

&nbsp;       <div className="absolute top-0 right-0 w-64 h-64 bg-blue-600 opacity-10 rounded-full -mr-32 -mt-32"></div>

&nbsp;       <div className="max-w-7xl mx-auto grid grid-cols-2 md:grid-cols-4 gap-12 text-center text-white relative z-10">

&nbsp;         <div>

&nbsp;           <div className="text-5xl font-black mb-2 text-blue-400">500+</div>

&nbsp;           <div className="text-slate-400 font-bold">Orders Done</div>

&nbsp;         </div>

&nbsp;         <div>

&nbsp;           <div className="text-5xl font-black mb-2 text-green-400">4.9/5</div>

&nbsp;           <div className="text-slate-400 font-bold">User Rating</div>

&nbsp;         </div>

&nbsp;         <div>

&nbsp;           <div className="text-5xl font-black mb-2 text-purple-400">50+</div>

&nbsp;           <div className="text-slate-400 font-bold">Local Partners</div>

&nbsp;         </div>

&nbsp;         <div>

&nbsp;           <div className="text-5xl font-black mb-2 text-orange-400">0%</div>

&nbsp;           <div className="text-slate-400 font-bold">Fraud Risk</div>

&nbsp;         </div>

&nbsp;       </div>

&nbsp;     </section>

&nbsp;   </div>

&nbsp; );

}



function RequestPage({ onSubmit }) {

&nbsp; const \[formData, setFormData] = useState({

&nbsp;   name: '', mobile: '', address: '', size: '', type: 'Clear Glass', desc: ''

&nbsp; });



&nbsp; return (

&nbsp;   <div className="max-w-2xl mx-auto px-4 py-16 animate-in slide-in-from-bottom-8">

&nbsp;     <div className="bg-white rounded-\[2.5rem] shadow-2xl overflow-hidden border border-slate-100">

&nbsp;       <div className="bg-blue-600 p-10 text-white text-center">

&nbsp;         <h1 className="text-3xl font-black mb-2">Request Call Back</h1>

&nbsp;         <p className="text-blue-100 font-medium">अपनी जानकारी भरें, हम आपको 30 मिनट में कॉल करेंगे।</p>

&nbsp;       </div>

&nbsp;       <form className="p-10 space-y-6" onSubmit={(e) => { e.preventDefault(); onSubmit(formData); }}>

&nbsp;         <div className="grid md:grid-cols-2 gap-6">

&nbsp;           <div>

&nbsp;             <label className="block text-sm font-black text-slate-700 mb-2">आपका नाम (Full Name)</label>

&nbsp;             <input 

&nbsp;               required

&nbsp;               className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"

&nbsp;               placeholder="Ex: Rajesh Kumar"

&nbsp;               value={formData.name}

&nbsp;               onChange={e => setFormData({...formData, name: e.target.value})}

&nbsp;             />

&nbsp;           </div>

&nbsp;           <div>

&nbsp;             <label className="block text-sm font-black text-slate-700 mb-2">मोबाइल नंबर (Mobile)</label>

&nbsp;             <input 

&nbsp;               required

&nbsp;               type="tel"

&nbsp;               maxLength="10"

&nbsp;               className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"

&nbsp;               placeholder="10 digit number"

&nbsp;               value={formData.mobile}

&nbsp;               onChange={e => setFormData({...formData, mobile: e.target.value})}

&nbsp;             />

&nbsp;           </div>

&nbsp;         </div>



&nbsp;         <div>

&nbsp;           <label className="block text-sm font-black text-slate-700 mb-2">काम का पता (Address)</label>

&nbsp;           <input 

&nbsp;             required

&nbsp;             className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"

&nbsp;             placeholder="City, Area, House No."

&nbsp;             value={formData.address}

&nbsp;             onChange={e => setFormData({...formData, address: e.target.value})}

&nbsp;           />

&nbsp;         </div>



&nbsp;         <div className="grid md:grid-cols-2 gap-6">

&nbsp;           <div>

&nbsp;             <label className="block text-sm font-black text-slate-700 mb-2 flex items-center gap-2">

&nbsp;               <Ruler size={16} /> कांच का साइज (Size)

&nbsp;             </label>

&nbsp;             <input 

&nbsp;               className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none transition-all font-medium"

&nbsp;               placeholder="Ex: 3ft x 4ft"

&nbsp;               value={formData.size}

&nbsp;               onChange={e => setFormData({...formData, size: e.target.value})}

&nbsp;             />

&nbsp;           </div>

&nbsp;           <div>

&nbsp;             <label className="block text-sm font-black text-slate-700 mb-2">कांच का प्रकार (Type)</label>

&nbsp;             <select 

&nbsp;               className="w-full px-5 py-4 rounded-2xl border border-slate-200 focus:ring-4 focus:ring-blue-100 focus:border-blue-500 outline-none bg-white font-medium"

&nbsp;               value={formData.type}

&nbsp;               onChange={e => setFormData({...formData, type: e.target.value})}

&nbsp;             >

&nbsp;               <option>Clear Glass (सादा कांच)</option>

&nbsp;               <option>Toughened Glass (मजबूत कांच)</option>

&nbsp;               <option>Frosted / Design (डिज़ाइनर)</option>

&nbsp;               <option>Mirror (शीशा)</option>

&nbsp;             </select>

&nbsp;           </div>

&nbsp;         </div>



&nbsp;         <button 

&nbsp;           type="submit"

&nbsp;           className="w-full bg-blue-600 text-white py-5 rounded-2xl font-black text-xl hover:bg-blue-700 shadow-xl shadow-blue-100 transition-all active:scale-95"

&nbsp;         >

&nbsp;           Request Call Back

&nbsp;         </button>

&nbsp;       </form>

&nbsp;     </div>

&nbsp;   </div>

&nbsp; );

}



function TrackOrder({ requests }) {

&nbsp; const \[mobile, setMobile] = useState('');

&nbsp; const \[result, setResult] = useState(null);



&nbsp; const handleSearch = () => {

&nbsp;   const found = requests.filter(r => r.mobile === mobile);

&nbsp;   setResult(found);

&nbsp; };



&nbsp; return (

&nbsp;   <div className="max-w-2xl mx-auto px-4 py-16">

&nbsp;     <div className="text-center mb-10">

&nbsp;       <h1 className="text-3xl font-black mb-4">Track Your Order</h1>

&nbsp;       <p className="text-slate-500">अपना मोबाइल नंबर डालकर ऑर्डर स्टेटस चेक करें।</p>

&nbsp;     </div>



&nbsp;     <div className="bg-white p-4 rounded-3xl shadow-xl flex gap-3 border border-slate-100 mb-10">

&nbsp;       <input 

&nbsp;         type="tel" 

&nbsp;         placeholder="Enter Registered Mobile" 

&nbsp;         className="flex-1 px-6 py-4 rounded-2xl border-none focus:ring-0 text-lg font-bold"

&nbsp;         value={mobile}

&nbsp;         onChange={e => setMobile(e.target.value)}

&nbsp;       />

&nbsp;       <button 

&nbsp;         onClick={handleSearch}

&nbsp;         className="bg-blue-600 text-white p-4 rounded-2xl hover:bg-blue-700 shadow-lg shadow-blue-200"

&nbsp;       >

&nbsp;         <Search size={24} />

&nbsp;       </button>

&nbsp;     </div>



&nbsp;     <div className="space-y-6">

&nbsp;       {result?.map(r => (

&nbsp;         <div key={r.id} className="bg-white p-8 rounded-\[2rem] shadow-lg border-l-8 border-blue-600 animate-in slide-in-from-top-4">

&nbsp;           <div className="flex justify-between items-start mb-6">

&nbsp;             <div>

&nbsp;               <p className="text-xs font-black text-slate-400 uppercase tracking-widest">Order ID</p>

&nbsp;               <h3 className="text-xl font-black">{r.id.slice(-6).toUpperCase()}</h3>

&nbsp;             </div>

&nbsp;             <span className={`px-4 py-2 rounded-xl text-sm font-black uppercase tracking-tighter ${

&nbsp;               r.status === 'Pending' ? 'bg-orange-100 text-orange-600' : 

&nbsp;               r.status === 'Assigned' ? 'bg-blue-100 text-blue-600' : 'bg-green-100 text-green-600'

&nbsp;             }`}>

&nbsp;               {r.status}

&nbsp;             </span>

&nbsp;           </div>

&nbsp;           

&nbsp;           <div className="space-y-4 mb-8">

&nbsp;              <div className="flex items-center gap-3 text-slate-600 font-bold">

&nbsp;                 <Clock size={18} className="text-blue-500" />

&nbsp;                 <span>Received: {new Date(r.createdAt).toLocaleString()}</span>

&nbsp;              </div>

&nbsp;              <div className="flex items-center gap-3 text-slate-600 font-bold">

&nbsp;                 <Truck size={18} className="text-blue-500" />

&nbsp;                 <span>{r.status === 'Pending' ? 'Admin is finding a shopkeeper...' : 'Assigned to verified local shop.'}</span>

&nbsp;              </div>

&nbsp;           </div>



&nbsp;           {r.totalPrice > 0 \&\& (

&nbsp;             <div className="bg-slate-50 p-6 rounded-2xl flex justify-between items-center border border-slate-100">

&nbsp;               <span className="font-bold text-slate-500">Service Price:</span>

&nbsp;               <span className="text-2xl font-black text-slate-900">₹{r.totalPrice}</span>

&nbsp;             </div>

&nbsp;           )}

&nbsp;         </div>

&nbsp;       ))}

&nbsp;       {result \&\& result.length === 0 \&\& <p className="text-center font-bold text-red-500">No order found with this number.</p>}

&nbsp;     </div>

&nbsp;   </div>

&nbsp; );

}



function AdminDashboard({ requests, vendors, onUpdateStatus }) {

&nbsp; const \[selectedRequest, setSelectedRequest] = useState(null);

&nbsp; const \[activeTab, setActiveTab] = useState('orders');



&nbsp; const totalRevenue = requests.reduce((acc, curr) => acc + (curr.totalPrice || 0), 0);

&nbsp; const totalCommission = requests.reduce((acc, curr) => acc + (curr.totalPrice - curr.vendorPrice), 0);



&nbsp; return (

&nbsp;   <div className="max-w-7xl mx-auto px-4 py-12">

&nbsp;     <div className="flex flex-col md:flex-row md:items-center justify-between mb-12 gap-6">

&nbsp;       <h1 className="text-3xl font-black flex items-center gap-3">

&nbsp;         <LayoutDashboard className="text-blue-600" /> Control Center

&nbsp;       </h1>

&nbsp;       <div className="flex bg-white p-1.5 rounded-2xl shadow-sm border border-slate-100">

&nbsp;         <button onClick={() => setActiveTab('orders')} className={`px-6 py-3 rounded-xl font-black transition-all ${activeTab === 'orders' ? 'bg-blue-600 text-white' : 'text-slate-500'}`}>Orders</button>

&nbsp;         <button onClick={() => setActiveTab('shops')} className={`px-6 py-3 rounded-xl font-black transition-all ${activeTab === 'shops' ? 'bg-blue-600 text-white' : 'text-slate-500'}`}>Partners</button>

&nbsp;       </div>

&nbsp;     </div>



&nbsp;     <div className="grid md:grid-cols-4 gap-6 mb-12">

&nbsp;       <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">

&nbsp;         <p className="text-slate-400 text-xs font-black uppercase mb-2">New Leads</p>

&nbsp;         <p className="text-4xl font-black">{requests.filter(r=>r.status === 'Pending').length}</p>

&nbsp;       </div>

&nbsp;       <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">

&nbsp;         <p className="text-slate-400 text-xs font-black uppercase mb-2">Active Partners</p>

&nbsp;         <p className="text-4xl font-black text-blue-600">{vendors.length}</p>

&nbsp;       </div>

&nbsp;       <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">

&nbsp;         <p className="text-slate-400 text-xs font-black uppercase mb-2">Total Volume</p>

&nbsp;         <p className="text-4xl font-black text-green-600">₹{totalRevenue.toLocaleString()}</p>

&nbsp;       </div>

&nbsp;       <div className="bg-blue-600 p-8 rounded-3xl shadow-xl text-white">

&nbsp;         <p className="text-blue-200 text-xs font-black uppercase mb-2">My Commission (5%)</p>

&nbsp;         <p className="text-4xl font-black text-white">₹{totalCommission.toFixed(2)}</p>

&nbsp;       </div>

&nbsp;     </div>



&nbsp;     {activeTab === 'orders' ? (

&nbsp;       <div className="bg-white rounded-\[2rem] shadow-xl border border-slate-100 overflow-x-auto">

&nbsp;         <table className="w-full text-left min-w-\[800px]">

&nbsp;           <thead className="bg-slate-50/50 border-b">

&nbsp;             <tr>

&nbsp;               <th className="p-6 font-black text-slate-400 text-xs uppercase">Customer</th>

&nbsp;               <th className="p-6 font-black text-slate-400 text-xs uppercase">Work Details</th>

&nbsp;               <th className="p-6 font-black text-slate-400 text-xs uppercase">Price Breakdown</th>

&nbsp;               <th className="p-6 font-black text-slate-400 text-xs uppercase">Status</th>

&nbsp;               <th className="p-6 font-black text-slate-400 text-xs uppercase text-right">Action</th>

&nbsp;             </tr>

&nbsp;           </thead>

&nbsp;           <tbody>

&nbsp;             {requests.map(r => (

&nbsp;               <tr key={r.id} className="border-b last:border-0 hover:bg-slate-50/50">

&nbsp;                 <td className="p-6">

&nbsp;                   <p className="font-black text-slate-900">{r.name}</p>

&nbsp;                   <p className="text-sm font-bold text-slate-400">{r.mobile}</p>

&nbsp;                   <p className="text-xs text-slate-400 mt-1 truncate max-w-\[200px]">{r.address}</p>

&nbsp;                 </td>

&nbsp;                 <td className="p-6">

&nbsp;                   <p className="font-bold text-slate-700">{r.type}</p>

&nbsp;                   <p className="text-xs font-bold text-blue-500 uppercase">{r.size || 'No Size'}</p>

&nbsp;                 </td>

&nbsp;                 <td className="p-6">

&nbsp;                   {r.vendorPrice > 0 ? (

&nbsp;                     <div className="space-y-1">

&nbsp;                       <p className="text-xs text-slate-400">Shop: ₹{r.vendorPrice}</p>

&nbsp;                       <p className="text-xs text-green-600 font-bold">Comm: ₹{(r.totalPrice - r.vendorPrice).toFixed(0)}</p>

&nbsp;                       <p className="text-sm font-black text-slate-900 border-t pt-1">Total: ₹{r.totalPrice}</p>

&nbsp;                     </div>

&nbsp;                   ) : (

&nbsp;                     <span className="text-slate-300 italic text-sm">Waiting for price...</span>

&nbsp;                   )}

&nbsp;                 </td>

&nbsp;                 <td className="p-6">

&nbsp;                   <span className={`px-4 py-2 rounded-xl text-\[10px] font-black uppercase ${

&nbsp;                     r.status === 'Pending' ? 'bg-orange-100 text-orange-600' : 

&nbsp;                     r.status === 'Assigned' ? 'bg-blue-100 text-blue-600' : 'bg-green-100 text-green-600'

&nbsp;                   }`}>

&nbsp;                     {r.status}

&nbsp;                   </span>

&nbsp;                 </td>

&nbsp;                 <td className="p-6 text-right">

&nbsp;                   <button 

&nbsp;                     onClick={() => setSelectedRequest(r)}

&nbsp;                     className="bg-slate-900 text-white px-5 py-2 rounded-xl font-bold text-sm hover:scale-105 transition-transform"

&nbsp;                   >

&nbsp;                     Manage

&nbsp;                   </button>

&nbsp;                 </td>

&nbsp;               </tr>

&nbsp;             ))}

&nbsp;           </tbody>

&nbsp;         </table>

&nbsp;         {requests.length === 0 \&\& <div className="p-20 text-center font-bold text-slate-300">No requests yet.</div>}

&nbsp;       </div>

&nbsp;     ) : (

&nbsp;       <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-8">

&nbsp;         {vendors.map(v => (

&nbsp;           <div key={v.id} className="bg-white p-8 rounded-\[2rem] border border-slate-100 shadow-lg relative group overflow-hidden">

&nbsp;              <div className="absolute top-0 right-0 w-24 h-24 bg-blue-50 rounded-full -mr-12 -mt-12 group-hover:scale-110 transition-transform"></div>

&nbsp;              <h3 className="text-xl font-black mb-1">{v.shopName}</h3>

&nbsp;              <p className="text-slate-400 font-bold mb-4">{v.owner} • {v.location}</p>

&nbsp;              <div className="flex gap-4 items-center">

&nbsp;                 <div className="bg-slate-50 px-4 py-2 rounded-xl text-xs font-black text-slate-500">Exp: {v.experience}</div>

&nbsp;                 <div className="bg-blue-50 px-4 py-2 rounded-xl text-xs font-black text-blue-600">{v.activeOrders} Active</div>

&nbsp;              </div>

&nbsp;              <button className="mt-6 w-full bg-slate-100 text-slate-900 py-3 rounded-xl font-black flex items-center justify-center gap-2 hover:bg-slate-900 hover:text-white transition-all">

&nbsp;                 <Phone size={18} /> Contact Shop

&nbsp;              </button>

&nbsp;           </div>

&nbsp;         ))}

&nbsp;         {vendors.length === 0 \&\& <div className="col-span-full py-20 text-center font-bold text-slate-300">No partner shops registered.</div>}

&nbsp;       </div>

&nbsp;     )}



&nbsp;     {selectedRequest \&\& (

&nbsp;       <div className="fixed inset-0 z-\[100] bg-slate-900/60 backdrop-blur-md flex items-center justify-center p-4">

&nbsp;         <div className="bg-white w-full max-w-lg rounded-\[2.5rem] shadow-2xl overflow-hidden animate-in zoom-in-95 duration-200">

&nbsp;           <div className="p-10 border-b flex justify-between items-center bg-blue-600 text-white">

&nbsp;             <h3 className="font-black text-2xl">Manage Order</h3>

&nbsp;             <button onClick={() => setSelectedRequest(null)} className="hover:rotate-90 transition-transform"><X /></button>

&nbsp;           </div>

&nbsp;           <div className="p-10 space-y-8">

&nbsp;             <div>

&nbsp;               <label className="block text-xs font-black uppercase text-slate-400 mb-3">Assign Shopkeeper</label>

&nbsp;               <select className="w-full p-4 rounded-2xl border border-slate-200 bg-white font-bold">

&nbsp;                 {vendors.map(v => <option key={v.id}>{v.shopName}</option>)}

&nbsp;                 <option>Self Service (Nayan Glass)</option>

&nbsp;               </select>

&nbsp;             </div>



&nbsp;             <div>

&nbsp;               <label className="block text-xs font-black uppercase text-slate-400 mb-3">Shopkeeper Price (₹)</label>

&nbsp;               <div className="relative">

&nbsp;                 <IndianRupee className="absolute left-4 top-4 text-slate-300" size={20} />

&nbsp;                 <input 

&nbsp;                   id="price-input"

&nbsp;                   type="number" 

&nbsp;                   className="w-full pl-12 p-4 rounded-2xl border border-slate-200 font-black text-xl" 

&nbsp;                   placeholder="Enter basic work cost" 

&nbsp;                   defaultValue={selectedRequest.vendorPrice}

&nbsp;                 />

&nbsp;                 <p className="mt-2 text-xs text-green-600 font-bold italic">+ 5% commission will be added for customer automatically</p>

&nbsp;               </div>

&nbsp;             </div>



&nbsp;             <div className="grid grid-cols-2 gap-4 pt-4">

&nbsp;               <button 

&nbsp;                 onClick={() => {

&nbsp;                   const vPrice = parseInt(document.getElementById('price-input').value || 0);

&nbsp;                   onUpdateStatus(selectedRequest.id, 'Assigned', vPrice);

&nbsp;                   setSelectedRequest(null);

&nbsp;                 }}

&nbsp;                 className="bg-slate-100 text-slate-900 py-5 rounded-2xl font-black hover:bg-slate-200"

&nbsp;               >

&nbsp;                 Save Status

&nbsp;               </button>

&nbsp;               <button 

&nbsp;                 onClick={() => {

&nbsp;                   const vPrice = parseInt(document.getElementById('price-input').value || 0);

&nbsp;                   onUpdateStatus(selectedRequest.id, 'Completed', vPrice);

&nbsp;                   setSelectedRequest(null);

&nbsp;                 }}

&nbsp;                 className="bg-green-600 text-white py-5 rounded-2xl font-black hover:bg-green-700 shadow-xl shadow-green-100"

&nbsp;               >

&nbsp;                 Mark Finished

&nbsp;               </button>

&nbsp;             </div>

&nbsp;           </div>

&nbsp;         </div>

&nbsp;       </div>

&nbsp;     )}

&nbsp;   </div>

&nbsp; );

}



function VendorOnboarding({ onRegister }) {

&nbsp; const \[formData, setFormData] = useState({ shopName: '', owner: '', mobile: '', location: '', experience: '1-3 Years' });



&nbsp; return (

&nbsp;   <div className="max-w-3xl mx-auto px-4 py-16 animate-in slide-in-from-right-8">

&nbsp;      <div className="bg-white rounded-\[2.5rem] shadow-2xl border border-slate-100 p-12 text-center">

&nbsp;         <div className="bg-green-100 w-24 h-24 rounded-3xl flex items-center justify-center text-green-600 mx-auto mb-8">

&nbsp;           <Store size={48} strokeWidth={2.5}/>

&nbsp;         </div>

&nbsp;         <h1 className="text-4xl font-black mb-4">Partner with Nayan Glass</h1>

&nbsp;         <p className="text-slate-500 text-lg mb-12">ग्लास वर्क का काम करते हैं? हमारे साथ जुड़ें और रोज़ाना 10+ नए आर्डर पाएं।</p>



&nbsp;         <form className="grid md:grid-cols-2 gap-8 text-left" onSubmit={(e) => { e.preventDefault(); onRegister(formData); }}>

&nbsp;           <div className="md:col-span-2 bg-blue-50 p-6 rounded-3xl border border-blue-100 flex items-start gap-4 mb-4">

&nbsp;              <Info className="text-blue-500 mt-1 shrink-0" size={24} />

&nbsp;              <div>

&nbsp;                 <h4 className="font-black text-blue-900">How it works?</h4>

&nbsp;                 <p className="text-sm text-blue-700">We send customers. You set the price. We take 5% commission. No monthly fees.</p>

&nbsp;              </div>

&nbsp;           </div>

&nbsp;           

&nbsp;           <div className="space-y-6">

&nbsp;             <div>

&nbsp;               <label className="block text-sm font-black mb-2 text-slate-700">दुकान का नाम (Shop Name)</label>

&nbsp;               <input required className="w-full p-4 rounded-2xl border border-slate-200" placeholder="Ganesh Glass House" onChange={e => setFormData({...formData, shopName: e.target.value})} />

&nbsp;             </div>

&nbsp;             <div>

&nbsp;               <label className="block text-sm font-black mb-2 text-slate-700">मोबाइल नंबर (Mobile)</label>

&nbsp;               <input required type="tel" className="w-full p-4 rounded-2xl border border-slate-200" placeholder="99999-XXXXX" onChange={e => setFormData({...formData, mobile: e.target.value})} />

&nbsp;             </div>

&nbsp;           </div>

&nbsp;           <div className="space-y-6">

&nbsp;             <div>

&nbsp;               <label className="block text-sm font-black mb-2 text-slate-700">अनुभव (Experience)</label>

&nbsp;               <select className="w-full p-4 rounded-2xl border border-slate-200 bg-white" onChange={e => setFormData({...formData, experience: e.target.value})}>

&nbsp;                 <option>1-3 Years</option>

&nbsp;                 <option>3-7 Years</option>

&nbsp;                 <option>7+ Years</option>

&nbsp;               </select>

&nbsp;             </div>

&nbsp;             <div>

&nbsp;               <label className="block text-sm font-black mb-2 text-slate-700">शहर / जिला (City)</label>

&nbsp;               <input required className="w-full p-4 rounded-2xl border border-slate-200" placeholder="Ex: Lucknow" onChange={e => setFormData({...formData, location: e.target.value})} />

&nbsp;             </div>

&nbsp;           </div>

&nbsp;           <button type="submit" className="md:col-span-2 bg-slate-900 text-white py-5 rounded-2xl font-black text-xl hover:bg-black transition-all shadow-2xl">

&nbsp;             Become a Partner Now

&nbsp;           </button>

&nbsp;         </form>

&nbsp;      </div>

&nbsp;   </div>

&nbsp; );

}



function Footer({ onNavigate }) {

&nbsp; return (

&nbsp;   <footer className="bg-slate-900 text-slate-400 py-20 px-4">

&nbsp;     <div className="max-w-7xl mx-auto grid grid-cols-1 md:grid-cols-4 gap-12">

&nbsp;       <div className="md:col-span-2">

&nbsp;         <div className="flex items-center gap-2 text-white mb-6">

&nbsp;           <ShieldCheck size={32} className="text-blue-500" />

&nbsp;           <span className="font-black text-2xl tracking-tighter uppercase">Nayan Glass Service</span>

&nbsp;         </div>

&nbsp;         <p className="text-lg leading-relaxed max-w-sm mb-8">भारत की सबसे भरोसेमंद ग्लास सर्विस प्लेटफॉर्म। सुरक्षित काम, सीधा दुकानदार से संपर्क।</p>

&nbsp;         <div className="flex gap-4">

&nbsp;            <div className="w-12 h-12 bg-slate-800 rounded-xl flex items-center justify-center text-white cursor-pointer hover:bg-blue-600 transition-colors"><Phone size={20} /></div>

&nbsp;            <div className="w-12 h-12 bg-slate-800 rounded-xl flex items-center justify-center text-white cursor-pointer hover:bg-blue-600 transition-colors"><MessageCircle size={20} /></div>

&nbsp;         </div>

&nbsp;       </div>

&nbsp;       <div>

&nbsp;         <h4 className="text-white font-black mb-6 uppercase tracking-widest text-xs">Navigation</h4>

&nbsp;         <ul className="space-y-4 font-bold">

&nbsp;           <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('home')}>Home</li>

&nbsp;           <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('request')}>Service Booking</li>

&nbsp;           <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('track')}>Track Order</li>

&nbsp;           <li className="cursor-pointer hover:text-white transition-colors" onClick={() => onNavigate('vendor')}>Partner Login</li>

&nbsp;         </ul>

&nbsp;       </div>

&nbsp;       <div>

&nbsp;         <h4 className="text-white font-black mb-6 uppercase tracking-widest text-xs">Working Areas</h4>

&nbsp;         <ul className="space-y-4 font-bold">

&nbsp;           <li>Noida / Greater Noida</li>

&nbsp;           <li>New Delhi / Ghaziabad</li>

&nbsp;           <li>Lucknow / Kanpur</li>

&nbsp;           <li>Mumbai / Pune</li>

&nbsp;         </ul>

&nbsp;       </div>

&nbsp;     </div>

&nbsp;     <div className="max-w-7xl mx-auto mt-20 pt-8 border-t border-slate-800 text-sm text-center font-bold">

&nbsp;       © 2024 Nayan Glass Service. Made for India 🇮🇳

&nbsp;     </div>

&nbsp;   </footer>

&nbsp; );

}

