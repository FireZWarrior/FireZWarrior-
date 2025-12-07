import React, { useState, useCallback, useMemo, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { 
    getFirestore, 
    doc, 
    setDoc, 
    onSnapshot, 
    Firestore, 
    collection, 
    query, 
    where,
    addDoc, 
    getDocs, 
    updateDoc 
} from 'firebase/firestore';

// GLUE CODE: Global variables for Firebase configuration (provided by environment)
declare const __app_id: string;
declare const __firebase_config: string;
declare const __initial_auth_token: string;

// Sub-collection name for transaction history
const TRANSACTION_COLLECTION = "transactions"; 

// Define Admin IDs (Only the user's ID is used as requested)
// အက်ဒမင် ID များ - တောင်းဆိုချက်အရ အက်ဒမင် တစ်ဦးတည်းသာ ထည့်သွင်းထားသည်
const ADMIN_IDS = ["11374268089904132258"]; 

// Constants for the new point system
const POINTS_PER_AD = 10;
const EXCHANGE_PACKAGES = [
    { diamonds: 5, cost: 500, label: "5 Diamond (500 Points)" },
    { diamonds: 10, cost: 1000, label: "10 Diamond (1000 Points)" },
    { diamonds: 20, cost: 2000, label: "20 Diamond (2000 Points)" },
];

// Define component state types
type Step = 1 | 2 | 3;
type View = 'USER_FLOW' | 'ADMIN_DASHBOARD';
type AdminTab = 'PENDING' | 'HISTORY'; // NEW: Admin view tabs

// NEW: Interface for the global app settings document
interface AppSettings {
    nextResetTimeISO: string | null;
}

interface ExchangeTransaction {
    id: string; // Document ID
    userId: string; // NEW: Store the user ID for Admin history viewing
    accountName: string; // NEW: Store the account name for Admin history viewing
    gameId: string; // NEW: Store the game ID for Admin history viewing
    timestamp: string; // Time of request
    diamonds: number;
    cost: number;
    isFulfilled: boolean;
    fulfilledTime?: string; // Time of fulfillment
}

interface MLBBProfile {
    accountName: string;
    gameId: string;
    serverId: string;
    lastDiamondReward: number; 
    currentPoints: number; 
    lastUpdate: string;
    currentStep: Step;
    isFulfilled: boolean; 
    pendingTransactionId?: string | null; 
    // NEW FIELD FOR RESET LOGIC
    lastResetCheck: string; // ISO string of the last time the user checked the global reset time.
}

// Icon SVGs (Inline for single-file requirement)
const UserIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2"/><circle cx="12" cy="7" r="4"/></svg>
);
const ZapIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/></svg>
);
const GamepadIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M6 12H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h16a2 2 0 0 1 2 2v6a2 2 0 0 1-2 2h-2"/><path d="M12 22v-4"/><path d="M12 18h.01"/><path d="M16 16h-8"/><path d="M16 12h-8"/><path d="M12 8v-.01"/><path d="M16 18h-8"/></svg>
);
const RefreshCwIcon = (props: React.SVGProps<SVGSVGElement>) => (
    <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M23 4v6h-6"/><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"/></svg>
);
const HistoryIcon = (props: React.SVGProps<SVGSVGElement>) => (
    <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 2v10l4-4"/><circle cx="12" cy="12" r="10"/></svg>
);
const LoadingSpinner = () => (
    <svg className="animate-spin h-6 w-6 text-gray-700" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
        <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
        <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
    </svg>
);
const DiamondIcon = () => (
    <span role="img" aria-label="diamond" className="text-4xl">💎</span>
);
const PointIcon = (props: React.SVGProps<SVGSVGElement>) => (
    <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><circle cx="12" cy="12" r="10"/><path d="M12 16a4 4 0 0 0 0-8"/><path d="M12 2a2 2 0 0 1 2 2 2 2 0 0 1-4 0 2 2 0 0 1 2-2z"/><path d="M22 12a2 2 0 0 1-2 2 2 2 0 0 1 0-4 2 2 0 0 1 2 2z"/><path d="M2 12a2 2 0 0 1 2 2 2 2 0 0 1 0-4 2 2 0 0 1-2 2z"/><path d="M12 22a2 2 0 0 1 2-2 2 2 0 0 1-4 0 2 2 0 0 1 2 2z"/></svg>
);

// --- New Time Display Component ---
const TimeDisplay: React.FC<{ time: Date }> = ({ time }) => {
    // Custom formatting to show "ည" (night/PM) or "နံနက်" (morning/AM) and MMT
    const formatOptions: Intl.DateTimeFormatOptions = {
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit',
        hour12: true,
        timeZone: 'Asia/Yangon', // Explicitly setting Myanmar Time (MMT)
    };
    
    // Format Time
    const formattedTime = time.toLocaleTimeString('en-US', formatOptions);
    const [timePart, ampm] = formattedTime.split(' ');
    
    const burmeseAmpm = ampm === 'PM' ? 'ည' : 'နံနက်';
    
    // Format Date
    const formattedDate = time.toLocaleDateString('my-MM', {
        day: '2-digit',
        month: '2-digit',
        year: 'numeric',
        timeZone: 'Asia/Yangon',
    });

    return (
        <div className="text-center text-xs sm:text-sm text-yellow-300 font-mono p-3 bg-gray-700 rounded-xl shadow-inner border border-red-700 w-full max-w-md mx-auto mb-6">
            <p className="font-bold text-base">App Server အချိန် (MMT - 1 မိနစ်နောက်ကျ)</p>
            <p className="mt-1 text-2xl font-extrabold text-white">
                {timePart} <span className="text-sm font-normal text-red-400 ml-1">{burmeseAmpm}</span>
            </p>
            <p className="text-gray-400">{formattedDate}</p>
        </div>
    );
};
// --- End New Time Display Component ---


const StepIndicator: React.FC<{ currentStep: Step }> = ({ currentStep }) => {
    const steps = [
        { id: 1, name: 'အကောင့်', icon: UserIcon },
        { id: 2, name: 'ဂိမ်း ID', icon: GamepadIcon },
        { id: 3, name: 'လဲလှယ်ရန်', icon: RefreshCwIcon },
    ];

    return (
        <div className="flex justify-between items-center w-full max-w-md mx-auto mb-10">
            {steps.map((step) => {
                const isActive = step.id === currentStep;
                const isCompleted = step.id < currentStep;
                const baseClasses = "flex flex-col items-center transition-colors duration-300";
                const colorClasses = isActive ? "text-yellow-400" : isCompleted ? "text-green-400" : "text-gray-500";

                return (
                    <React.Fragment key={step.id}>
                        <div className={`${baseClasses} ${colorClasses}`}>
                            <div className={`p-3 rounded-full ${isActive ? 'bg-yellow-800 border-2 border-yellow-400 shadow-xl' : isCompleted ? 'bg-green-800' : 'bg-gray-700'} transition-all`}>
                                <step.icon className="w-6 h-6"/>
                            </div>
                            <p className="mt-2 text-sm font-medium">{step.name}</p>
                        </div>
                        {step.id < 3 && (
                            <div className={`flex-1 h-1 mx-2 ${isCompleted || isActive ? 'bg-red-500' : 'bg-gray-700'} rounded-full transition-colors duration-300`}></div>
                        )}
                    </React.Fragment>
                );
            })}
        </div>
    );
};

// --- Step 1: Account Input ---
const AccountStep: React.FC<{ profile: MLBBProfile | null, onNext: (name: string) => void }> = ({ profile, onNext }) => {
    const [accountName, setAccountName] = useState(profile?.accountName || '');
    const [error, setError] = useState('');

    useEffect(() => {
        if (profile?.accountName) {
            setAccountName(profile.accountName);
        }
    }, [profile]);

    const handleSubmit = useCallback(() => {
        if (accountName.trim().length > 2) {
            onNext(accountName.trim());
        } else {
            setError('အကောင့်နာမည်ကို မှန်ကန်စွာ ထည့်သွင်းပေးပါ။');
        }
    }, [accountName, onNext]);

    return (
        <div className="space-y-6">
            <h2 className="text-2xl font-bold text-center text-white">အဆင့် ၁: အကောင့်ထည့်သွင်းရန်</h2>
            <p className="text-gray-400 text-center">သင့်ရဲ့ လဲလှယ်မှုအတွက် အကောင့်နာမည်ကို ဖန်တီးပါ သို့မဟုတ် ပြင်ဆင်ပါ။</p>
            
            <div className="relative">
                <input
                    type="text"
                    placeholder="အကောင့် နာမည် (ဥပမာ: MLBBPro)"
                    value={accountName}
                    onChange={(e) => { setAccountName(e.target.value); setError(''); }}
                    className="w-full p-4 pl-12 bg-gray-700 border border-gray-600 rounded-xl text-white placeholder-gray-400 focus:ring-yellow-500 focus:border-yellow-500 transition duration-200 shadow-inner"
                    required
                />
                <UserIcon className="absolute left-3 top-1/2 transform -translate-y-1/2 w-6 h-6 text-yellow-400"/>
            </div>
            
            {error && <p className="text-red-400 text-sm text-center">{error}</p>}
            
            <button
                onClick={handleSubmit}
                disabled={accountName.trim().length <= 2}
                className="w-full py-3 bg-red-600 text-white font-bold rounded-xl shadow-lg hover:bg-red-700 transition duration-300 disabled:bg-red-900 disabled:opacity-70 transform hover:scale-[1.01]"
            >
                နောက်တစ်ဆင့်သို့
            </button>
        </div>
    );
};

// --- Step 2: Game ID Input ---
const GameIdStep: React.FC<{ profile: MLBBProfile | null, onNext: (gameId: string, serverId: string) => void, onBack: () => void }> = ({ profile, onNext, onBack }) => {
    const [gameId, setGameId] = useState(profile?.gameId || '');
    const [serverId, setServerId] = useState(profile?.serverId || '');
    const [error, setError] = useState('');

    useEffect(() => {
        if (profile) {
            setGameId(profile.gameId || '');
            setServerId(profile.serverId || '');
        }
    }, [profile]);

    const handleSubmit = useCallback(() => {
        if (gameId.trim() && serverId.trim()) {
            onNext(gameId.trim(), serverId.trim());
        } else {
            setError('ဂိမ်း ID နှင့် Server ID နှစ်ခုလုံးကို ဖြည့်သွင်းပေးပါ။');
        }
    }, [gameId, serverId, onNext]);

    return (
        <div className="space-y-6">
            <h2 className="text-2xl font-bold text-center text-white">အဆင့် ၂: ဂိမ်းအချက်အလက်</h2>
            <p className="text-gray-400 text-center">သင့်ရဲ့ MLBB ဂိမ်း ID နှင့် Server ID ကို ထည့်သွင်းပါ သို့မဟုတ် ပြင်ဆင်ပါ။</p>
            
            <div className="flex flex-col space-y-4">
                {/* Game ID Input */}
                <div className="relative">
                    <input
                        type="number"
                        placeholder="ဂိမ်း ID (ဥပမာ: 12345678)"
                        value={gameId}
                        onChange={(e) => { setGameId(e.target.value); setError(''); }}
                        className="w-full p-4 pl-12 bg-gray-700 border border-gray-600 rounded-xl text-white placeholder-gray-400 focus:ring-yellow-500 focus:border-yellow-500 transition duration-200 shadow-inner"
                        required
                    />
                    <GamepadIcon className="absolute left-3 top-1/2 transform -translate-y-1/2 w-6 h-6 text-yellow-400"/>
                </div>

                {/* Server ID Input */}
                <div className="relative">
                    <input
                        type="number"
                        placeholder="ဆာဗာ ID (ဥပမာ: 9876)"
                        value={serverId}
                        onChange={(e) => { setServerId(e.target.value); setError(''); }}
                        className="w-full p-4 pl-12 bg-gray-700 border border-gray-600 rounded-xl text-white placeholder-gray-400 focus:ring-yellow-500 focus:border-yellow-500 transition duration-200 shadow-inner"
                        required
                    />
                    <ZapIcon className="absolute left-3 top-1/2 transform -translate-y-1/2 w-6 h-6 text-yellow-400"/>
                </div>
            </div>

            {error && <p className="text-red-400 text-sm text-center">{error}</p>}
            
            <div className="flex space-x-4">
                <button
                    onClick={onBack}
                    className="w-1/3 py-3 bg-gray-600 text-white font-bold rounded-xl shadow-lg hover:bg-gray-700 transition duration-300 transform hover:scale-[1.01]"
                >
                    နောက်သို့
                </button>
                <button
                    onClick={handleSubmit}
                    disabled={!gameId.trim() || !serverId.trim()}
                    className="w-2/3 py-3 bg-red-600 text-white font-bold rounded-xl shadow-lg hover:bg-red-700 transition duration-300 disabled:bg-red-900 disabled:opacity-70 transform hover:scale-[1.01]"
                >
                    အချက်အလက်အတည်ပြုရန်
                </button>
            </div>
        </div>
    );
};

// --- Admin Dashboard Core Component (Pending Requests & Point Management) ---
const AdminDashboardCore: React.FC<{ 
    db: Firestore, 
    fulfillRequest: (id: string, pendingTransactionId: string) => Promise<void>, 
    appId: string,
    saveProfile: (data: Partial<MLBBProfile>, targetUserId: string | null) => Promise<void>,
    appSettings: AppSettings, // NEW: Pass app settings
    saveAppSettings: (data: Partial<AppSettings>) => Promise<void> // NEW: Pass save settings handler
}> = ({ db, fulfillRequest, appId, saveProfile, appSettings, saveAppSettings }) => {
    
    // Admin dashboard elements are now styled for a LIGHT background (bg-white)
    
    const [pendingRequests, setPendingRequests] = useState<({ id: string } & MLBBProfile)[]>([]); 
    const [loading, setLoading] = useState(true);
    
    // State for Point Management
    const [targetUserIdForPoints, setTargetUserIdForPoints] = useState('');
    const [pointsToAdjust, setPointsToAdjust] = useState<number | ''>('');
    const [pointManagementLoading, setPointManagementLoading] = useState(false);
    const [pointManagementMessage, setPointManagementMessage] = useState<{ message: string; isError: boolean } | null>(null);

    // --- State for Reset Time Management (NEW) ---
    const [resetDate, setResetDate] = useState(''); // YYYY-MM-DD
    const [resetTime, setResetTime] = useState(''); // HH:MM
    const [resetLoading, setResetLoading] = useState(false);
    const [resetMessage, setResetMessage] = useState<{ message: string; isError: boolean } | null>(null);

    // Effect to populate current reset time if available
    useEffect(() => {
        if (appSettings.nextResetTimeISO) {
            try {
                // Ensure the date object is created correctly
                const d = new Date(appSettings.nextResetTimeISO);
                if (isNaN(d.getTime())) {
                    console.error("Invalid date ISO string from settings:", appSettings.nextResetTimeISO);
                    return;
                }
                
                // Format date part YYYY-MM-DD
                const datePart = d.toISOString().split('T')[0];
                // Format time part HH:MM (24-hour format)
                const timePart = d.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: false, timeZone: 'Asia/Yangon' });
                
                setResetDate(datePart);
                setResetTime(timePart);
                
            } catch (e) {
                console.error("Error processing existing reset time:", e);
            }
        }
    }, [appSettings.nextResetTimeISO]);


    // Handler to set the new global reset time
    const handleSetResetTime = useCallback(async () => {
        if (!resetDate || !resetTime) {
            setResetMessage({ message: "❌ နေ့စွဲနှင့် အချိန်ကို ပြည့်စုံစွာ ထည့်သွင်းပေးပါ။", isError: true });
            return;
        }
        
        // Combine date and time (assuming MMT as the local target time for input)
        // Note: Using the client's local time zone interpretation here for date parsing.
        const targetDate = new Date(`${resetDate}T${resetTime}:00`); 
        
        if (isNaN(targetDate.getTime())) {
            setResetMessage({ message: "❌ နေ့စွဲ သို့မဟုတ် အချိန် format မမှန်ကန်ပါ။", isError: true });
            return;
        }

        if (targetDate.getTime() < Date.now()) {
            setResetMessage({ message: "❌ အချိန်သည် လက်ရှိအချိန်ထက် ကျော်လွန်ရပါမည်။", isError: true });
            return;
        }

        setResetLoading(true);
        setResetMessage(null);

        try {
            const newResetTimeISO = targetDate.toISOString();
            await saveAppSettings({ nextResetTimeISO: newResetTimeISO });

            setResetMessage({ message: `✅ Points Reset လုပ်မည့်အချိန်ကို ${targetDate.toLocaleString()} အဖြစ် သတ်မှတ်ခဲ့သည်။`, isError: false });
        } catch (error) {
            console.error("Error setting reset time:", error);
            setResetMessage({ message: "❌ Reset အချိန် သတ်မှတ်ရာတွင် အမှားအယွင်းရှိခဲ့သည်။", isError: true });
        } finally {
            setResetLoading(false);
        }
    }, [resetDate, resetTime, saveAppSettings]);
    // --- End Reset Time Management (NEW) ---

    useEffect(() => {
        if (!db) return;

        // Path: /artifacts/{appId}/public/data/mlbb_requests
        const requestsCollectionRef = collection(db, `artifacts/${appId}/public/data/mlbb_requests`);
        
        const q = query(
            requestsCollectionRef,
            where("isFulfilled", "==", false)
        );

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const requests = snapshot.docs.map(doc => ({
                id: doc.id,
                ...(doc.data() as MLBBProfile)
            }))
            .filter(r => r.pendingTransactionId && r.lastDiamondReward > 0); 
            
            requests.sort((a, b) => new Date(a.lastUpdate).getTime() - new Date(b.lastUpdate).getTime());
            
            setPendingRequests(requests);
            setLoading(false);
        }, (error) => {
            console.error("Admin Dashboard Snapshot Error:", error);
            setLoading(false);
        });

        return () => unsubscribe();
    }, [db, appId]);
    
    // Function to handle point adjustment (Set New Total)
    const handlePointAdjustment = useCallback(async () => {
        if (!targetUserIdForPoints || pointsToAdjust === '' || isNaN(Number(pointsToAdjust)) || Number(pointsToAdjust) < 0) {
            setPointManagementMessage({ message: "❌ အသုံးပြုသူ ID သို့မဟုတ် Points ပမာဏ (အနည်းဆုံး ၀) မှားယွင်းနေသည်။", isError: true });
            return;
        }

        setPointManagementLoading(true);
        setPointManagementMessage(null);
        
     
