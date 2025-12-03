<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WBC 賽事預測平台</title>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Babel for JSX transformation -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

    <!-- Import Map for React, Firebase, and Icons -->
    <script type="importmap">
    {
        "imports": {
            "react": "https://esm.sh/react@18.2.0",
            "react-dom/client": "https://esm.sh/react-dom@18.2.0/client",
            "firebase/app": "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js",
            "firebase/auth": "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js",
            "firebase/firestore": "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js",
            "lucide-react": "https://esm.sh/lucide-react@0.378.0?external=react"
        }
    }
    </script>

    <style>
        /* Custom scrollbar for webkit */
        ::-webkit-scrollbar {
            width: 8px;
        }
        ::-webkit-scrollbar-track {
            background: #1f2937; 
        }
        ::-webkit-scrollbar-thumb {
            background: #4b5563; 
            border-radius: 4px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #6b7280; 
        }
    </style>
</head>
<body class="bg-gray-900 text-white min-h-screen">
    <div id="root"></div>

    <script type="text/babel" data-type="module">
        import React, { useState, useEffect, useCallback, useMemo, useRef } from 'react';
        import { createRoot } from 'react-dom/client';
        // 加入 getApps 以防止重複初始化錯誤
        import { initializeApp, getApps } from 'firebase/app';
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from 'firebase/auth';
        import { 
            getFirestore, doc, getDoc, setDoc, onSnapshot, collection, query, updateDoc,
            writeBatch
        } from 'firebase/firestore';
        
        import { 
            Copy, Trophy, Users, Gift, Star, ArrowRight, Smartphone, Lock, User, 
            MessageSquare, Mail, Key, X, Calendar, Clock, Settings, CheckCircle, 
            Award, Info, LogOut, Zap, HelpCircle, MapPin, MousePointer, Edit2, Save
        } from 'lucide-react';

        // --- Configuration Section ---
        
        // 為了讓此 HTML 檔案能獨立運作，請在此填入您的 Firebase 設定
        // 如果您沒有設定，網頁會以「展示模式」運行（無法真正讀寫資料庫）
        const manualConfig = {
            apiKey: "",
            authDomain: "",
            projectId: "",
            storageBucket: "",
            messagingSenderId: "",
            appId: ""
        };

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-wbc-app';
        // 優先使用環境變數，若無則使用手動設定
        const firebaseConfig = typeof __firebase_config !== 'undefined' 
            ? JSON.parse(__firebase_config) 
            : manualConfig;

        const withBackoff = async (fn, retries = 3, delay = 1000) => {
            try {
                return await fn();
            } catch (error) {
                if (retries > 0) {
                    console.warn(`Retry attempt (${3 - retries + 1}):`, error.message);
                    await new Promise(resolve => setTimeout(resolve, delay));
                    return withBackoff(fn, retries - 1, delay * 2);
                }
                throw error;
            }
        };

        // --- Firebase Context and State Management ---

        const useFirebase = () => {
            const [db, setDb] = useState(null);
            const [auth, setAuth] = useState(null);
            const [firebaseUser, setFirebaseUser] = useState(null); 
            const [isFirebaseReady, setIsFirebaseReady] = useState(false);

            useEffect(() => {
                // Check if config is valid (has apiKey)
                if (!firebaseConfig || !firebaseConfig.apiKey) {
                    console.warn("Firebase configuration is missing or empty. Running in Demo Mode (UI only).");
                    // Simulate ready for UI purposes
                    setIsFirebaseReady(true);
                    return;
                }

                try {
                    // 使用 getApps() 檢查是否已經初始化，避免重複初始化錯誤
                    const app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApps()[0];
                    const firestoreDb = getFirestore(app);
                    const firebaseAuth = getAuth(app);
                    
                    setDb(firestoreDb);
                    setAuth(firebaseAuth);

                    const unsubscribe = onAuthStateChanged(firebaseAuth, async (currentUser) => {
                        setFirebaseUser(currentUser);
                        setIsFirebaseReady(true);
                    });

                    const initializeAuth = async () => {
                        if (!firebaseAuth.currentUser) {
                            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                                await signInWithCustomToken(firebaseAuth, __initial_auth_token);
                            } else {
                                await signInAnonymously(firebaseAuth);
                            }
                        }
                    };
                    
                    initializeAuth();
                    return () => unsubscribe();
                } catch (err) {
                    console.error("Firebase init error:", err);
                    setIsFirebaseReady(true); // Allow UI to render even if firebase fails
                }
            }, []);

            const getUserId = useCallback(() => {
                return firebaseUser ? firebaseUser.uid : null;
            }, [firebaseUser]);

            const handleSignOut = useCallback(async () => {
                if (auth) {
                    try {
                        await signOut(auth);
                        console.log("User signed out successfully.");
                    } catch (error) {
                        console.error("Sign out failed:", error);
                    }
                }
            }, [auth]);

            return { db, auth, firebaseUser, isFirebaseReady, getUserId, handleSignOut };
        };

        // --- Form Input Field Helper ---
        const InputField = ({ label, name, type = 'text', icon: Icon, placeholder, formData, handleChange, required = true, disabled = false, isTextArea = false }) => (
            <div className="space-y-1">
                <label className="text-sm font-medium text-gray-300">{label}</label>
                <div className="relative">
                    {Icon && <Icon className={`absolute left-3 ${isTextArea ? 'top-3' : 'top-1/2 transform -translate-y-1/2'} w-5 h-5 text-gray-500`} />}
                    {isTextArea ? (
                        <textarea
                            name={name}
                            value={formData[name] || ''}
                            onChange={handleChange}
                            placeholder={placeholder || `請輸入${label}`}
                            disabled={disabled}
                            rows="2"
                            className={`w-full px-4 py-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-blue-500 focus:border-blue-500 ${Icon ? 'pl-10' : ''} ${disabled ? 'opacity-60 cursor-not-allowed' : ''} resize-none text-sm`}
                            required={required}
                        />
                    ) : (
                        <input
                            type={type}
                            name={name}
                            value={formData[name] || ''}
                            onChange={handleChange}
                            placeholder={placeholder || `請輸入${label}`}
                            disabled={disabled}
                            className={`w-full px-4 py-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-blue-500 focus:border-blue-500 ${Icon ? 'pl-10' : ''} ${disabled ? 'opacity-60 cursor-not-allowed' : ''} text-sm`}
                            required={required}
                        />
                    )}
                </div>
            </div>
        );

        // --- Helper: Copy Function ---
        const copyToClipboard = (text) => {
            if (!text) return;
            
            const textArea = document.createElement("textarea");
            textArea.value = text;
            textArea.style.position = "fixed";
            textArea.style.left = "-9999px";
            textArea.style.top = "0";
            document.body.appendChild(textArea);
            textArea.focus();
            textArea.select();
            
            try {
                const successful = document.execCommand('copy');
                if (successful) {
                    alert("邀請碼已複製！");
                } else {
                    console.error('Copy command failed.');
                    alert("複製失敗，請手動複製。");
                }
            } catch (err) {
                console.error('Unable to copy', err);
                alert("複製發生錯誤。");
            }
            document.body.removeChild(textArea);
        };

        // --- Component: User Profile Modal ---
        const UserProfileModal = ({ isOpen, onClose, userProfile, onUpdateProfile }) => {
            const [isEditing, setIsEditing] = useState(false);
            const [formData, setFormData] = useState({ ...userProfile });

            const predictionPrizes = [
                { wins: 10, prize: '7-11 100元', id: 'wins_10' },
                { wins: 20, prize: '7-11 500元', id: 'wins_20' },
                { wins: 30, prize: '7-11 1000元', id: 'wins_30' },
                { wins: 40, prize: '7-11 2000元', id: 'wins_40' },
            ];
            const referralPrizes = [
                { count: 10, prize: '7-11 50元', id: 'referral_10' },
                { count: 20, prize: '7-11 100元', id: 'referral_20' },
                { count: 30, prize: '7-11 300元', id: 'referral_30' },
            ];

            useEffect(() => {
                if (userProfile) {
                    setFormData({ ...userProfile });
                }
            }, [userProfile]);

            const handleChange = (e) => {
                const { name, value } = e.target;
                setFormData(prev => ({ ...prev, [name]: value }));
            };

            const handleSave = (e) => {
                e.preventDefault();
                onUpdateProfile(formData);
                setIsEditing(false);
            };

            const getStatusBadge = (isUnlocked, statusKey, statusMap) => {
                const status = statusMap?.[statusKey];
                if (status === 'SENT') {
                    return <span className="px-2 py-0.5 rounded text-xs font-bold bg-blue-600 text-white">已寄出</span>;
                }
                if (isUnlocked) {
                    return <span className="px-2 py-0.5 rounded text-xs font-bold bg-green-600 text-white">已達成</span>;
                }
                return <span className="px-2 py-0.5 rounded text-xs font-bold bg-gray-600 text-gray-400">未達成</span>;
            };

            if (!isOpen) return null;

            return (
                <div className="fixed inset-0 bg-black/80 z-[70] flex items-center justify-center p-4">
                    <div className="bg-gray-800 p-6 sm:p-8 rounded-xl shadow-2xl w-full max-w-4xl border border-gray-700 max-h-[90vh] overflow-y-auto">
                        <div className="flex justify-between items-center border-b border-gray-700 pb-3 mb-6">
                            <h2 className="text-xl sm:text-3xl font-extrabold text-white flex items-center">
                                <User className="w-5 h-5 sm:w-6 sm:h-6 mr-2 text-blue-400" />
                                會員中心
                            </h2>
                            <button onClick={onClose} className="text-gray-400 hover:text-white transition">
                                <X className="w-6 h-6" />
                            </button>
                        </div>

                        <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
                            <div className="space-y-4">
                                <div className="flex justify-between items-center mb-2">
                                    <h3 className="text-lg font-bold text-white">個人資料</h3>
                                    {!isEditing ? (
                                        <button 
                                            onClick={() => setIsEditing(true)}
                                            className="text-sm text-blue-400 hover:text-blue-300 flex items-center"
                                        >
                                            <Edit2 className="w-4 h-4 mr-1" /> 修改
                                        </button>
                                    ) : (
                                        <button 
                                            onClick={handleSave}
                                            className="text-sm text-green-400 hover:text-green-300 flex items-center"
                                        >
                                            <Save className="w-4 h-4 mr-1" /> 儲存
                                        </button>
                                    )}
                                </div>
                                
                                <form className="space-y-3">
                                    <InputField label="帳號" name="username" icon={User} formData={formData} handleChange={handleChange} disabled={true} />
                                    <InputField label="密碼" name="password" type="password" icon={Key} formData={formData} handleChange={handleChange} disabled={!isEditing} />
                                    <InputField label="暱稱" name="nickname" icon={User} formData={formData} handleChange={handleChange} disabled={!isEditing} />
                                    <InputField label="電話" name="phone" type="tel" icon={Smartphone} formData={formData} handleChange={handleChange} disabled={!isEditing} />
                                    <InputField label="信箱" name="email" type="email" icon={Mail} formData={formData} handleChange={handleChange} disabled={!isEditing} />
                                    <InputField label="地址" name="address" icon={MapPin} formData={formData} handleChange={handleChange} disabled={!isEditing} isTextArea={true} />
                                </form>
                            </div>

                            <div className="space-y-6">
                                <div>
                                    <h3 className="text-lg font-bold text-white mb-3 flex items-center"><Gift className="w-5 h-5 mr-2 text-yellow-400"/> 預測獎勵狀況</h3>
                                    <div className="bg-gray-700/30 p-4 rounded-lg border border-gray-600 space-y-3">
                                        <div className="flex justify-between items-center pb-2 border-b border-gray-600">
                                            <span className="text-gray-300 text-sm">目前成功預測</span>
                                            <span className="text-xl font-bold text-yellow-400">{userProfile.successfulPredictions} 局</span>
                                        </div>
                                        <div className="space-y-2">
                                            {predictionPrizes.map((p, idx) => {
                                                const isUnlocked = userProfile.successfulPredictions >= p.wins;
                                                return (
                                                    <div key={idx} className="flex justify-between items-center text-sm">
                                                        <span className="text-gray-400">{p.wins} 局 ({p.prize})</span>
                                                        {getStatusBadge(isUnlocked, p.id, userProfile.prizesStatus)}
                                                    </div>
                                                );
                                            })}
                                            <div className="flex justify-between items-center text-sm pt-1">
                                                <span className="text-pink-400">40 局加碼 (iPhone 17)</span>
                                                <span className={`px-2 py-0.5 rounded text-xs font-bold ${userProfile.successfulPredictions >= 40 ? 'bg-pink-600 text-white' : 'bg-gray-600 text-gray-400'}`}>
                                                    {userProfile.successfulPredictions >= 40 ? '已具備資格' : '未達成'}
                                                </span>
                                            </div>
                                        </div>
                                    </div>
                                </div>

                                <div>
                                    <h3 className="text-lg font-bold text-white mb-3 flex items-center"><Users className="w-5 h-5 mr-2 text-green-400"/> 好友邀請獎勵狀況</h3>
                                    <div className="bg-gray-700/30 p-4 rounded-lg border border-gray-600 space-y-3">
                                        <div className="flex justify-between items-center pb-2 border-b border-gray-600">
                                            <span className="text-gray-300 text-sm">已邀請人數</span>
                                            <span className="text-xl font-bold text-green-400">{userProfile.referredUsers} 人</span>
                                        </div>
                                         <div className="space-y-2">
                                            {referralPrizes.map((p, idx) => {
                                                const isUnlocked = userProfile.referredUsers >= p.count;
                                                return (
                                                    <div key={idx} className="flex justify-between items-center text-sm">
                                                        <span className="text-gray-400">{p.count} 人 ({p.prize})</span>
                                                        {getStatusBadge(isUnlocked, p.id, userProfile.prizesStatus)}
                                                    </div>
                                                );
                                            })}
                                        </div>
                                        <div className="mt-3 pt-2 border-t border-gray-600 text-sm text-gray-400 flex flex-col sm:flex-row justify-between items-start sm:items-center gap-2">
                                            <span>專屬邀請碼:</span>
                                            <div className="flex items-center gap-2 w-full sm:w-auto">
                                                <span className="font-mono text-white bg-gray-600 px-3 py-1 rounded select-all flex-1 text-center">{userProfile.referralCode}</span>
                                                <button 
                                                    onClick={() => copyToClipboard(userProfile.referralCode)}
                                                    className="p-1.5 bg-blue-600 rounded hover:bg-blue-500 text-white transition"
                                                    title="複製邀請碼"
                                                >
                                                    <Copy className="w-4 h-4" />
                                                </button>
                                            </div>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            );
        };

        // --- Component: Registration Modal ---
        const RegistrationModal = ({ isOpen, onClose, onFormSubmit }) => {
            const [formData, setFormData] = useState({
                username: '',
                password: '',
                nickname: '',
                phone: '',
                email: '',
                address: '', 
                referralCode: '',
            });

            const handleChange = (e) => {
                const { name, value } = e.target;
                setFormData(prev => ({ ...prev, [name]: value }));
            };

            const handleSubmit = (e) => {
                e.preventDefault();
                onFormSubmit(formData);
                onClose();
            };

            if (!isOpen) return null;

            return (
                <div className="fixed inset-0 bg-black/80 z-[60] flex items-center justify-center p-4">
                    <div className="bg-gray-800 p-8 rounded-xl shadow-2xl w-full max-w-lg border border-gray-700 max-h-[90vh] overflow-y-auto">
                        <div className="flex justify-between items-center border-b border-gray-700 pb-3 mb-4">
                            <h2 className="text-2xl sm:text-3xl font-extrabold text-center text-white">會員註冊</h2>
                            <button onClick={onClose} className="text-gray-400 hover:text-white transition">
                                <X className="w-6 h-6" />
                            </button>
                        </div>
                        <form onSubmit={handleSubmit} className="space-y-4">
                            <InputField label="帳號" name="username" icon={User} formData={formData} handleChange={handleChange} />
                            <InputField label="密碼" name="password" type="password" icon={Key} formData={formData} handleChange={handleChange} />
                            <InputField label="暱稱" name="nickname" icon={User} formData={formData} handleChange={handleChange} />
                            <InputField label="電話" name="phone" type="tel" icon={Smartphone} placeholder="例如: 0912345678" formData={formData} handleChange={handleChange} />
                            <InputField label="信箱" name="email" type="email" icon={Mail} formData={formData} handleChange={handleChange} />
                            <InputField label="地址" name="address" icon={MapPin} formData={formData} handleChange={handleChange} isTextArea={true} />
                            <InputField label="邀請碼 (選填)" name="referralCode" icon={CheckCircle} formData={formData} handleChange={handleChange} required={false} />
                            
                            <div className="mt-6 text-center">
                                <p className="text-yellow-400 font-semibold mb-3 text-sm bg-yellow-400/10 p-2 rounded">
                                    請加入客服LINE@，開通帳號即可預測。
                                </p>
                                <button
                                    type="submit"
                                    className="w-full py-3 text-xl font-bold text-white bg-blue-600 rounded-lg hover:bg-blue-700 transition duration-200 shadow-lg shadow-blue-500/50 transform hover:scale-[1.01]"
                                >
                                    送出
                                </button>
                            </div>
                        </form>
                    </div>
                </div>
            );
        };

        // --- Component: Login Modal ---
        const LoginModal = ({ isOpen, onClose, onLoginSuccess }) => {
            const [formData, setFormData] = useState({
                username: 'ABC', 
                password: '123',
            });

            const handleChange = (e) => {
                const { name, value } = e.target;
                setFormData(prev => ({ ...prev, [name]: value }));
            };

            const handleSubmit = (e) => {
                e.preventDefault();
                if (formData.username === 'ABC' && formData.password === '123') {
                    onLoginSuccess();
                    onClose();
                } else {
                    alert("登入失敗：請使用預設帳號 ABC / 123");
                }
            };

            if (!isOpen) return null;

            return (
                <div className="fixed inset-0 bg-black/80 z-[60] flex items-center justify-center p-4">
                    <div className="bg-gray-800 p-8 rounded-xl shadow-2xl w-full max-w-lg border border-gray-700">
                        <div className="flex justify-between items-center border-b border-gray-700 pb-3 mb-4">
                            <h2 className="text-2xl sm:text-3xl font-extrabold text-center text-white">會員登入</h2>
                            <button onClick={onClose} className="text-gray-400 hover:text-white transition">
                                <X className="w-6 h-6" />
                            </button>
                        </div>
                        <form onSubmit={handleSubmit} className="space-y-4">
                            <InputField label="帳號" name="username" icon={User} formData={formData} handleChange={handleChange} />
                            
                            <div className="space-y-1">
                                <InputField label="密碼" name="password" type="password" icon={Key} formData={formData} handleChange={handleChange} />
                                <div className="flex justify-end pt-1">
                                    <button 
                                        type="button"
                                        className="text-sm text-blue-400 hover:text-blue-300 hover:underline flex items-center transition-colors"
                                    >
                                        <HelpCircle className="w-3 h-3 mr-1" />
                                        忘記密碼?
                                    </button>
                                </div>
                            </div>
                            
                            <div className="mt-6 text-center">
                                <button
                                    type="submit"
                                    className="w-full py-3 text-xl font-bold text-white bg-blue-600 rounded-lg hover:bg-blue-700 transition duration-200 shadow-lg shadow-blue-500/50 transform hover:scale-[1.01]"
                                >
                                    送出
                                </button>
                            </div>
                        </form>
                    </div>
                </div>
            );
        };

        // --- Component: Prediction Popup ---
        const PredictionPopup = ({ match, prediction, isOpen, onClose, onPredict }) => {
            const teamAPercent = match.ratioA || 50; 
            const teamBPercent = 100 - teamAPercent;

            const isLocked = match.officialWinner || new Date(match.date).getTime() < Date.now();
            const userPredicted = !!prediction;
            const { teamA, teamB } = match;

            if (!isOpen) return null;

            return (
                <div className="fixed inset-0 bg-black/80 z-[60] flex items-center justify-center p-4">
                    <div className="bg-gray-900 p-6 rounded-xl shadow-2xl w-full max-w-lg border-2 border-blue-500">
                        <div className="flex justify-between items-center border-b border-gray-700 pb-3 mb-4">
                            <h2 className="text-xl sm:text-2xl font-extrabold text-white">賽事預測：{teamA} VS {teamB}</h2>
                            <button onClick={onClose} className="text-gray-400 hover:text-white transition">
                                <X className="w-6 h-6" />
                            </button>
                        </div>
                        <div className="text-sm text-gray-400 mb-4 space-y-1">
                            <p className="flex items-center"><Calendar className="w-4 h-4 mr-2" />
                                {new Date(match.date).toLocaleDateString('zh-TW')}
                            </p>
                        </div>

                        <div className="mb-6 bg-gray-800 p-4 rounded-lg">
                            <p className="text-lg font-bold text-yellow-400 mb-3 text-center">大眾預測比例</p>
                            <div className="flex space-x-2 items-center">
                                <div className="flex-1 text-center">
                                    <p className="text-sm text-white mb-1">{teamA}</p>
                                    <div className="h-6 bg-gray-700 rounded-full overflow-hidden relative">
                                        <div className="h-full bg-blue-500" style={{ width: `${teamAPercent}%` }}></div>
                                        <span className="absolute inset-0 flex items-center justify-center text-xs font-bold text-white shadow-black drop-shadow-md">{teamAPercent}%</span>
                                    </div>
                                </div>
                                <div className="flex-1 text-center">
                                    <p className="text-sm text-white mb-1">{teamB}</p>
                                    <div className="h-6 bg-gray-700 rounded-full overflow-hidden relative">
                                        <div className="h-full bg-red-500 ml-auto" style={{ width: `${teamBPercent}%` }}></div>
                                        <span className="absolute inset-0 flex items-center justify-center text-xs font-bold text-white shadow-black drop-shadow-md">{teamBPercent}%</span>
                                    </div>
                                </div>
                            </div>
                        </div>

                        <p className="text-lg font-bold text-white mb-3 text-center">
                            {userPredicted ? `您的預測：${prediction.predictedWinner}` : '請選擇您預測的獲勝隊伍'}
                        </p>
                        
                        {isLocked ? (
                            <div className="p-4 bg-red-800/50 text-red-300 rounded-lg text-center font-bold">
                                賽事已截止或已結算。
                            </div>
                        ) : (
                            <div className="flex flex-col sm:flex-row justify-between space-y-3 sm:space-y-0 sm:space-x-4">
                                {[teamA, teamB].map((team) => (
                                    <button
                                        key={team}
                                        onClick={() => { onPredict(match.id, team); onClose(); }}
                                        className={`flex-1 p-4 text-center rounded-lg font-bold text-lg transition duration-200 shadow-md 
                                            ${(prediction?.predictedWinner === team) 
                                                ? 'bg-yellow-500 text-gray-900 border-4 border-yellow-300' 
                                                : 'bg-blue-600 text-white hover:bg-blue-700'
                                            }
                                        `}
                                    >
                                        {team}
                                    </button>
                                ))}
                            </div>
                        )}
                    </div>
                </div>
            );
        };

        // --- Component: Match Table Row ---
        const MatchTableRow = ({ match, prediction, onClick, isLoggedIn }) => {
            const isLocked = match.officialWinner || new Date(match.date).getTime() < Date.now();
            const userPredicted = !!prediction;
            const isCorrect = userPredicted && match.officialWinner && prediction.predictedWinner === match.officialWinner;
            
            const renderPredictionStatus = () => {
                if (!isLoggedIn) return null; 
                
                if (isCorrect) {
                    return <span className="px-2 py-0.5 text-xs font-bold rounded-full bg-green-500 text-white flex items-center justify-center w-fit mx-auto"><Trophy className="w-3 h-3 mr-1" />正確</span>;
                }
                if (userPredicted && !isLocked) {
                    return <span className="px-2 py-0.5 text-xs font-bold rounded-full bg-blue-500 text-white flex items-center justify-center w-fit mx-auto"><Star className="w-3 h-3 mr-1" />{prediction.predictedWinner}</span>;
                }
                if (isLocked && userPredicted && !isCorrect) {
                    return <span className="px-2 py-0.5 text-xs font-bold rounded-full bg-red-500 text-white w-fit mx-auto block">失敗</span>;
                }
                if (!userPredicted && !isLocked) {
                    return <span className="text-gray-400 text-xs">未預測</span>;
                }
                return <span className="text-gray-500 text-sm">-</span>;
            };

            return (
                <tr className={`border-b border-gray-700 hover:bg-gray-700/50 transition ${isLoggedIn && !isLocked ? 'cursor-pointer' : ''}`} onClick={() => isLoggedIn && !isLocked ? onClick(match) : null}>
                    <td className="p-3 text-xs sm:text-sm text-gray-300">{new Date(match.date).toLocaleDateString('zh-TW', { month: 'numeric', day: 'numeric' })}</td>
                    <td className="p-3 text-xs sm:text-sm text-gray-400">{new Date(match.date).toLocaleTimeString('zh-TW', { hour: '2-digit', minute: '2-digit' })}</td>
                    <td className="p-3 text-sm sm:text-base font-semibold text-white text-right">{match.teamA}</td>
                    <td className="p-3 text-center text-xs text-gray-500">VS</td>
                    <td className="p-3 text-sm sm:text-base font-semibold text-white text-left">{match.teamB}</td>
                    {isLoggedIn && (
                        <td className="p-3 text-center">
                            {renderPredictionStatus()}
                        </td>
                    )}
                    <td className="p-3 text-xs sm:text-sm font-semibold text-yellow-400 text-center">
                        {match.officialWinner ? `勝: ${match.officialWinner}` : '-'}
                    </td>
                </tr>
            );
        };

        // --- Component: Match Table ---
        const MatchTable = ({ matches, predictions, onPredict, setSelectedMatch, isLoggedIn }) => {
            const [activeTab, setActiveTab] = useState('C組');
            const tabs = ['A組', 'B組', 'C組', 'D組', '八強', '四強', '決賽'];
            const filteredMatches = useMemo(() => matches.filter(match => match.group === activeTab), [matches, activeTab]);

            return (
                <div className="bg-gray-800 p-4 sm:p-6 rounded-xl shadow-2xl mb-10 border border-blue-500/50 relative overflow-hidden">
                    <h2 className="text-xl sm:text-3xl font-extrabold text-white mb-4 border-b border-gray-700 pb-2 flex items-center">
                        <Trophy className="w-6 h-6 mr-3 text-blue-400" />
                        預測賽事區
                        {!isLoggedIn && <span className="ml-4 text-xs font-normal text-gray-400 hidden sm:inline">(登入後即可參與預測)</span>}
                    </h2>

                    <div className="flex overflow-x-auto space-x-3 pb-3 mb-4">
                        {tabs.map(tab => (
                            <button
                                key={tab}
                                onClick={() => setActiveTab(tab)}
                                className={`px-3 py-1.5 font-semibold text-sm rounded-full whitespace-nowrap transition duration-200 ${
                                    activeTab === tab ? 'bg-blue-600 text-white' : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                                }`}
                            >
                                {tab}
                            </button>
                        ))}
                    </div>

                    <div className="overflow-x-auto rounded-lg border border-gray-700">
                        <table className="min-w-full">
                            <thead className="bg-gray-700">
                                <tr>
                                    <th className="p-3 text-left text-xs font-bold text-gray-400 w-1/6">日期</th>
                                    <th className="p-3 text-left text-xs font-bold text-gray-400 w-1/12">時間</th>
                                    <th className="p-3 text-right text-xs font-bold text-gray-400 w-1/5">隊伍</th>
                                    <th className="p-3 text-center text-xs font-bold text-gray-400 w-1/12">VS</th>
                                    <th className="p-3 text-left text-xs font-bold text-gray-400 w-1/5">隊伍</th>
                                    {isLoggedIn && <th className="p-3 text-center text-xs font-bold text-gray-400 w-1/6">您的預測</th>}
                                    <th className="p-3 text-center text-xs font-bold text-gray-400 w-1/6">賽果</th>
                                </tr>
                            </thead>
                            <tbody className="divide-y divide-gray-700">
                                {filteredMatches.length === 0 ? (
                                    <tr><td colSpan={isLoggedIn ? 7 : 6} className="p-10 text-center text-gray-500">暫無賽事</td></tr>
                                ) : (
                                    filteredMatches.map(match => (
                                        <MatchTableRow
                                            key={match.id}
                                            match={match}
                                            prediction={predictions[match.id]}
                                            onClick={setSelectedMatch}
                                            isLoggedIn={isLoggedIn}
                                        />
                                    ))
                                )}
                            </tbody>
                        </table>
                    </div>
                    {isLoggedIn && (
                        <p className="mt-4 p-3 bg-blue-900/50 text-blue-300 rounded-lg text-sm text-center">
                            點擊賽事列即可進行預測。（八強/四強/決賽將即時更新）
                        </p>
                    )}
                </div>
            );
        };

        // --- Component: Prize Progress ---
        const PrizeProgress = ({ totalWins, prizesStatus, isLoggedIn }) => {
            const prizes = [
                { wins: 10, prize: '7-11禮券 100元', id: 'wins_10' },
                { wins: 20, prize: '7-11禮券 500元', id: 'wins_20' },
                { wins: 30, prize: '7-11禮券 1000元', id: 'wins_30' },
                { wins: 40, prize: '7-11禮券 2000元', id: 'wins_40' },
            ];
            
            return (
                <div className="bg-gray-800 p-4 sm:p-6 rounded-xl shadow-2xl mb-10 border border-yellow-700/50">
                    <h2 className="text-xl sm:text-3xl font-extrabold text-white mb-4 flex items-center justify-center">
                        <Gift className="w-6 h-6 mr-3 text-yellow-400" />
                        預測成功好禮
                    </h2>
                    
                    {isLoggedIn && (
                        <div className="flex justify-between items-center text-center text-gray-400 mb-4 border-b border-gray-700 pb-3">
                            <span className="text-lg font-semibold w-full">當前成功局數: <span className="text-yellow-400 text-2xl ml-1">{totalWins}</span> 局</span>
                        </div>
                    )}

                    <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-4">
                        {prizes.map((p) => {
                            const isAchieved = isLoggedIn && totalWins >= p.wins;
                            const status = prizesStatus?.[p.id];
                            const isSent = status === 'SENT';
                            
                            let cardColor = 'bg-gray-700 border-gray-600';
                            
                            let badge = '預測獎勵';
                            if (isLoggedIn) {
                                if (isSent) badge = '已寄出';
                                else if (isAchieved) badge = '已達成';
                                else badge = '未達成';
                            }
                            
                            let iconWrapperStyle = 'flex items-center justify-center h-12 w-12 mx-auto rounded-full bg-yellow-500/20 mb-3 border-4 border-yellow-500/50';
                            if (isLoggedIn && (isAchieved || isSent)) {
                                iconWrapperStyle += ' grayscale opacity-50';
                            }

                            return (
                                <div key={p.id} className={`p-3 rounded-xl shadow-lg transition duration-300 hover:scale-105 ${cardColor} relative group`}>
                                    <div className={`text-center relative z-10`}>
                                        <p className="text-sm font-bold text-white mb-2">{p.wins} 局</p>
                                        <div className={iconWrapperStyle}>
                                            <Gift className="w-6 h-6 text-yellow-300" />
                                        </div>
                                        <p className="text-xs font-semibold text-gray-300">{p.prize}</p>
                                    </div>
                                    <div className="text-center mt-2">
                                        <span className={`inline-block px-2 py-0.5 text-xs font-bold rounded-full ${isLoggedIn && (isAchieved || isSent) ? (isSent ? 'bg-blue-600 text-white' : 'bg-gray-500 text-gray-300') : 'text-white'}`}>
                                            {badge}
                                        </span>
                                    </div>
                                </div>
                            );
                        })}
                        
                        {(() => {
                            const isAchieved = isLoggedIn && totalWins >= 40;
                            const badge = isLoggedIn ? (isAchieved ? '已達成' : '未達成') : '預測獎勵';
                            
                            let iconWrapperStyle = 'flex items-center justify-center h-12 w-12 mx-auto rounded-full bg-pink-500/20 mb-3 border-4 border-pink-500/50';
                            if (isLoggedIn && isAchieved) {
                                iconWrapperStyle += ' grayscale opacity-50';
                            }
                            
                            return (
                                <div className={`p-3 rounded-xl shadow-lg transition duration-300 hover:scale-105 bg-gray-700 border-gray-600`}>
                                    <div className={`text-center`}>
                                        <p className="text-sm font-bold text-white mb-2 text-center">加碼 40 局</p>
                                        <div className={iconWrapperStyle}>
                                            <Star className="w-6 h-6 text-pink-300" />
                                        </div>
                                        <p className={`text-xs font-semibold ${isLoggedIn && !isAchieved ? 'text-pink-400' : 'text-gray-400'} text-center`}>iPhone 17 * 1</p>
                                    </div>
                                    <div className="text-center mt-2">
                                        <span className={`inline-block px-2 py-0.5 text-xs font-bold rounded-full ${isLoggedIn && isAchieved ? 'bg-gray-500 text-gray-300' : 'text-white'}`}>
                                            {badge}
                                        </span>
                                    </div>
                                </div>
                            );
                        })()}
                    </div>
                    
                    <p className="mt-4 text-center text-sm text-gray-400">
                        <span className="text-red-400 font-bold">*</span> 40 局加碼禮：需決賽+冠軍賽預測成功才具備抽獎資格。
                    </p>
                </div>
            );
        };

        // --- Component: Referral Recruitment ---
        const ReferralRecruitment = ({ referralCode, referredCount, isLoggedIn, handleCopyReferral, prizesStatus }) => {
            const tiers = [
                { count: 10, prize: '7-11禮券 50元', description: '邀請 10 人', id: 'referral_10' },
                { count: 20, prize: '7-11禮券 100元', description: '邀請 20 人', id: 'referral_20' },
                { count: 30, prize: '7-11禮券 300元', description: '邀請 30 人', id: 'referral_30' },
            ];

            return (
                <div className="bg-gray-800 p-4 sm:p-6 rounded-xl shadow-2xl mb-10 border border-green-700/50">
                    <h2 className="text-xl sm:text-3xl font-extrabold text-white mb-2 flex items-center justify-center">
                        <Users className="w-6 h-6 mr-3 text-green-400" />
                        好友招募
                    </h2>
                    <div className="text-center text-lg font-semibold text-yellow-400 mb-6 border-b border-gray-700 pb-3">
                        <p>【 集結好友 X 作伙預測！ 】</p>
                        <p>完成預約後立即招募好友，成功招募越多人，獎勵越豐富！</p>
                    </div>
                    
                    {isLoggedIn && (
                        <div className="flex flex-col sm:flex-row items-center justify-between space-y-4 sm:space-y-0 sm:space-x-6 mb-6 p-4 border border-green-700 rounded-lg bg-gray-700/30">
                            <div className="text-center sm:text-left">
                                <p className="text-sm font-semibold text-gray-400">當前已邀請人數</p>
                                <p className="text-green-400 text-3xl font-extrabold">{referredCount}</p>
                            </div>
                            <div className="flex-1 min-w-0 w-full sm:w-auto">
                                <p className="text-sm font-semibold text-gray-400 mb-1">您的專屬邀請碼</p>
                                <div className="flex">
                                    <span className="flex-1 text-lg font-mono px-4 py-2 bg-gray-700 rounded-l-lg text-white border border-green-500/50 truncate">
                                        {referralCode}
                                    </span>
                                    <button
                                        onClick={handleCopyReferral}
                                        className="flex items-center px-4 py-2 bg-green-600 text-white font-semibold rounded-r-lg hover:bg-green-700 transition duration-200"
                                    >
                                        <Copy className="w-4 h-4 mr-2" />複製
                                    </button>
                                </div>
                            </div>
                        </div>
                    )}

                    <div className="grid grid-cols-2 lg:grid-cols-3 gap-4">
                        {tiers.map((t) => {
                            const isAchieved = isLoggedIn && referredCount >= t.count;
                            const status = prizesStatus?.[t.id];
                            const isSent = status === 'SENT';

                            let cardColor = 'bg-gray-700 border-gray-600';
                            let badge = '招募獎勵';
                            if (isLoggedIn) {
                                if (isSent) badge = '已寄出';
                                else if (isAchieved) badge = '已達成';
                                else badge = '未達成';
                            }
                            
                            let iconWrapperStyle = 'flex items-center justify-center h-12 w-12 mx-auto rounded-full bg-green-500/20 mb-3 border-4 border-green-500/50';
                            if (isLoggedIn && (isAchieved || isSent)) {
                                iconWrapperStyle += ' grayscale opacity-50';
                            }

                            return (
                                <div key={t.count} className={`p-3 rounded-xl shadow-lg transition duration-300 hover:scale-105 ${cardColor}`}>
                                    <div className="text-center">
                                        <p className="text-sm font-bold text-white mb-2">{t.description}</p>
                                        <div className={iconWrapperStyle}>
                                            <Users className="w-6 h-6 text-green-300" />
                                        </div>
                                        <p className="text-xs font-semibold text-gray-400">{t.prize}</p>
                                    </div>
                                    <div className="text-center mt-2">
                                        <span className={`inline-block px-2 py-0.5 text-xs font-bold rounded-full ${isLoggedIn && (isAchieved || isSent) ? (isSent ? 'bg-blue-600 text-white' : 'bg-gray-500 text-gray-300') : 'text-white'}`}>
                                            {badge}
                                        </span>
                                    </div>
                                </div>
                            );
                        })}
                    </div>
                </div>
            );
        };

        // --- Component: Banner Section (Hero) ---
        const BannerSection = ({ profile, isLoggedIn, handleSignOut }) => {
            return (
                <div className="relative w-full overflow-hidden bg-gray-800 shadow-2xl rounded-xl mb-10 border-4 border-blue-600/50 group">
                    <div className="absolute inset-0 bg-gradient-to-b from-gray-900 via-gray-900/40 to-transparent pointer-events-none z-10"></div>

                    <div className="relative z-20 w-full h-full p-6 flex flex-col items-start">
                        
                        <div className="flex flex-col md:flex-row items-start md:items-center justify-start w-full gap-4 mb-6">
                            
                            <div className="inline-block px-4 py-1.5 bg-orange-600 text-white font-bold rounded-full shadow-lg text-xs sm:text-sm tracking-wide whitespace-nowrap">
                                GSBET x WBC 經典賽
                            </div>
                            
                            <div className="flex flex-wrap items-center gap-2 sm:gap-3 bg-gray-900/40 p-2 rounded-xl backdrop-blur-sm border border-white/5 shadow-xl">
                                <h1 className="text-2xl sm:text-3xl lg:text-4xl font-black text-green-400 tracking-wider italic transform -skew-x-6">
                                    WBC 預測風潮
                                </h1>

                                <div className="flex items-center gap-2">
                                    <div className="bg-white text-gray-900 p-1 rounded-full shadow-lg transform scale-100">
                                        <Zap className="w-4 h-4 fill-current" />
                                    </div>

                                    <span className="text-lg sm:text-xl font-bold text-white tracking-wide">
                                        好禮拿不完
                                    </span>
                                </div>
                            </div>
                        </div>

                        <div className="w-full flex justify-center items-end mt-20 mb-10">
                        </div>
                    </div>
                    
                    <div 
                        className="absolute inset-0 bg-cover bg-center opacity-40 z-0" 
                        style={{ backgroundImage: `url('https://placehold.co/1200x600/1e293b/fbbf24?text=Lightning+Bg')` }}
                    ></div>
                </div>
            );
        };

        // --- Component: Pre-Registration Section ---
        const PreRegistrationSection = () => {
            const [isRegisterModalOpen, setIsRegisterModalOpen] = useState(false);

            const handleRegistrationSubmit = (formData) => {
                console.log("Registration:", formData);
                alert(`預約註冊成功！請記得加入 LINE@ 開通帳號。\n帳號: ${formData.username}`);
            };

            return (
                <>
                    <div className="bg-gray-800 rounded-xl shadow-2xl mb-10 border border-green-500/50 flex flex-col lg:flex-row overflow-hidden">
                        
                        <div className="lg:w-1/2 bg-gray-900/90 p-8 flex flex-col items-center justify-center text-center border-r border-gray-700 relative">
                            <div className="absolute inset-0 opacity-10 bg-[radial-gradient(ellipse_at_center,_var(--tw-gradient-stops))] from-yellow-500 via-gray-900 to-gray-900"></div>
                            <h2 className="text-2xl font-bold text-white mb-6 relative z-10">事前預約好禮</h2>
                            
                            <div className="w-40 h-40 bg-gray-800 rounded-full flex items-center justify-center border-4 border-yellow-500 shadow-[0_0_20px_rgba(234,179,8,0.5)] mb-6 relative z-10">
                                <Gift className="w-20 h-20 text-yellow-400" />
                            </div>
                            
                            <div className="relative z-10">
                                <p className="text-xl font-bold text-white mb-1">預約成功可先領取獎勵</p>
                                <p className="text-3xl font-extrabold text-yellow-400 mb-2">7-11 禮券 50 元</p>
                                <p className="text-sm text-gray-400 mt-2 bg-black/40 px-3 py-1 rounded-full inline-block">
                                    * 但須預測成功 15 局才會發放
                                </p>
                            </div>
                        </div>

                        <div className="lg:w-1/2 p-8 flex flex-col justify-center space-y-6 bg-gray-800">
                            <div className="bg-gray-700/50 p-6 rounded-xl border border-green-500/30">
                                <h3 className="text-lg font-bold text-white mb-2 flex items-center">
                                    <MessageSquare className="w-5 h-5 mr-2 text-green-400" />
                                    1. 官方 LINE@ 加入好友 (必要)
                                </h3>
                                <p className="text-sm text-gray-400 mb-4 ml-7">
                                    點擊下方按鈕連結到 LINE@，後續再提供連結資訊。
                                </p>
                                <button className="w-full py-3 bg-[#06C755] hover:bg-[#05b54d] text-white font-bold rounded-lg transition shadow-lg flex items-center justify-center">
                                    <MessageSquare className="w-5 h-5 mr-2" /> 加入 LINE 好友
                                </button>
                            </div>

                            <div className="bg-gray-700/50 p-6 rounded-xl border border-blue-500/30">
                                <h3 className="text-lg font-bold text-white mb-2 flex items-center">
                                    <User className="w-5 h-5 mr-2 text-blue-400" />
                                    2. 填寫預約資訊
                                </h3>
                                <p className="text-sm text-gray-400 mb-4 ml-7">
                                    填寫基本資料完成預約，即可獲得預測資格。
                                </p>
                                <button 
                                    onClick={() => setIsRegisterModalOpen(true)}
                                    className="w-full py-3 bg-blue-600 hover:bg-blue-700 text-white font-bold rounded-lg transition shadow-lg flex items-center justify-center"
                                >
                                    <MousePointer className="w-5 h-5 mr-2" /> 預約註冊
                                </button>
                            </div>
                        </div>
                    </div>
                    
                    <RegistrationModal 
                        isOpen={isRegisterModalOpen} 
                        onClose={() => setIsRegisterModalOpen(false)} 
                        onFormSubmit={handleRegistrationSubmit}
                    />
                </>
            );
        };

        // --- Main App Component ---
        const App = () => {
            const { db, auth, isFirebaseReady, getUserId, handleSignOut } = useFirebase();
            
            const [isLoggedIn, setIsLoggedIn] = useState(false);
            const [userProfile, setUserProfile] = useState(null); 
            
            const [matches, setMatches] = useState([]);
            const [predictions, setPredictions] = useState({});
            const [selectedMatch, setSelectedMatch] = useState(null);
            const [isLoginModalOpen, setIsLoginModalOpen] = useState(false);
            const [isProfileModalOpen, setIsProfileModalOpen] = useState(false);

            const userId = getUserId(); 

            useEffect(() => {
                // If not connected to Firebase (Demo Mode), load dummy data
                if (!db) {
                     const now = Date.now();
                     const dummy = [
                        { id: 'm1', teamA: '台灣', teamB: '澳洲', date: new Date(now + 86400000).toISOString(), group: 'C組', officialWinner: '', ratioA: 65 },
                        { id: 'm2', teamA: '日本', teamB: '韓國', date: new Date(now + 172800000).toISOString(), group: 'C組', officialWinner: '日本', ratioA: 72 },
                        { id: 'm3', teamA: '捷克', teamB: '中國', date: new Date(now + 259200000).toISOString(), group: 'C組', officialWinner: '', ratioA: 48 },
                        { id: 'm4', teamA: '荷蘭', teamB: '古巴', date: new Date(now + 86400000).toISOString(), group: 'A組', officialWinner: '', ratioA: 55 },
                        { id: 'm5', teamA: '義大利', teamB: '巴拿馬', date: new Date(now + 172800000).toISOString(), group: 'A組', officialWinner: '', ratioA: 60 },
                        { id: 'm6', teamA: '美國', teamB: '英國', date: new Date(now + 86400000).toISOString(), group: 'B組', officialWinner: '', ratioA: 80 },
                        { id: 'm7', teamA: '墨西哥', teamB: '哥倫比亞', date: new Date(now + 172800000).toISOString(), group: 'B組', officialWinner: '', ratioA: 52 },
                        { id: 'm8', teamA: '委內瑞拉', teamB: '多明尼加', date: new Date(now + 86400000).toISOString(), group: 'D組', officialWinner: '', ratioA: 45 },
                        { id: 'm9', teamA: '波多黎各', teamB: '以色列', date: new Date(now + 172800000).toISOString(), group: 'D組', officialWinner: '', ratioA: 68 },
                        { id: 'qf1', teamA: '日本', teamB: '義大利', date: new Date(now + 500000000).toISOString(), group: '八強', officialWinner: '', ratioA: 70 },
                        { id: 'sf1', teamA: '美國', teamB: '古巴', date: new Date(now + 600000000).toISOString(), group: '四強', officialWinner: '', ratioA: 62 },
                        { id: 'f1', teamA: '日本', teamB: '美國', date: new Date(now + 700000000).toISOString(), group: '決賽', officialWinner: '', ratioA: 50 },
                     ];
                     setMatches(dummy);
                     return;
                }

                if (!userId) return;
                const matchesRef = collection(db, `artifacts/${appId}/public/data/matches`);
                const unsub = onSnapshot(matchesRef, (snap) => {
                    let data = snap.docs.map(d => ({ id: d.id, ...d.data() }));
                     if (data.length === 0) {
                         const now = Date.now();
                         const dummy = [
                            { id: 'm1', teamA: '台灣', teamB: '澳洲', date: new Date(now + 86400000).toISOString(), group: 'C組', officialWinner: '', ratioA: 65 },
                            { id: 'm2', teamA: '日本', teamB: '韓國', date: new Date(now + 172800000).toISOString(), group: 'C組', officialWinner: '日本', ratioA: 72 },
                            { id: 'm3', teamA: '捷克', teamB: '中國', date: new Date(now + 259200000).toISOString(), group: 'C組', officialWinner: '', ratioA: 48 },
                            { id: 'm4', teamA: '荷蘭', teamB: '古巴', date: new Date(now + 86400000).toISOString(), group: 'A組', officialWinner: '', ratioA: 55 },
                            { id: 'm5', teamA: '義大利', teamB: '巴拿馬', date: new Date(now + 172800000).toISOString(), group: 'A組', officialWinner: '', ratioA: 60 },
                            { id: 'm6', teamA: '美國', teamB: '英國', date: new Date(now + 86400000).toISOString(), group: 'B組', officialWinner: '', ratioA: 80 },
                            { id: 'm7', teamA: '墨西哥', teamB: '哥倫比亞', date: new Date(now + 172800000).toISOString(), group: 'B組', officialWinner: '', ratioA: 52 },
                            { id: 'm8', teamA: '委內瑞拉', teamB: '多明尼加', date: new Date(now + 86400000).toISOString(), group: 'D組', officialWinner: '', ratioA: 45 },
                            { id: 'm9', teamA: '波多黎各', teamB: '以色列', date: new Date(now + 172800000).toISOString(), group: 'D組', officialWinner: '', ratioA: 68 },
                            { id: 'qf1', teamA: '日本', teamB: '義大利', date: new Date(now + 500000000).toISOString(), group: '八強', officialWinner: '', ratioA: 70 },
                            { id: 'sf1', teamA: '美國', teamB: '古巴', date: new Date(now + 600000000).toISOString(), group: '四強', officialWinner: '', ratioA: 62 },
                            { id: 'f1', teamA: '日本', teamB: '美國', date: new Date(now + 700000000).toISOString(), group: '決賽', officialWinner: '', ratioA: 50 },
                         ];
                         setMatches(dummy);
                     } else {
                         data.sort((a,b) => new Date(a.date) - new Date(b.date));
                         setMatches(data);
                     }
                }, (error) => {
                    console.error("Error fetching matches:", error);
                });
                return () => unsub();
            }, [db, userId]);

            useEffect(() => {
                if (!db || !userId) return; 

                const predsRef = collection(db, `artifacts/${appId}/users/${userId}/predictions`);
                const unsubPreds = onSnapshot(predsRef, (snap) => {
                    const map = {};
                    snap.docs.forEach(d => map[d.id] = d.data());
                    setPredictions(map);
                }, (error) => {
                    console.error("Error fetching predictions:", error);
                });

                return () => { unsubPreds(); };
            }, [db, userId]);

            const handleLoginSuccess = () => {
                setIsLoggedIn(true);
                setUserProfile({
                    username: 'ABC',
                    password: '123',
                    nickname: '測試員',
                    phone: '0988776543',
                    email: 'Aabc@gmail.com',
                    address: '臺北市中正區重慶南路一段122號',
                    successfulPredictions: 25, 
                    referredUsers: 28, 
                    referralCode: 'WBC888',
                    prizesStatus: {
                        'wins_10': 'SENT', 
                        'referral_10': 'SENT' 
                    }, 
                });
            };

            const handleLogout = () => {
                setIsLoggedIn(false);
                setUserProfile(null);
            };

            const handleUpdateProfile = (newProfile) => {
                setUserProfile(newProfile);
                alert("會員資料更新成功！");
            };

            const handlePredict = async (matchId, team) => {
                if (!isLoggedIn) return; 
                
                // If demo mode (no db), just update local state for visual feedback
                if (!db || !userId) {
                    setPredictions(prev => ({
                        ...prev,
                        [matchId]: { matchId, predictedWinner: team }
                    }));
                    return;
                }

                const ref = doc(db, `artifacts/${appId}/users/${userId}/predictions/${matchId}`);
                await setDoc(ref, { matchId, predictedWinner: team }, { merge: true });
            };

            const handleCopyReferral = () => {
                if(userProfile?.referralCode) {
                    copyToClipboard(userProfile.referralCode);
                }
            };

            if (!isFirebaseReady) return <div className="bg-gray-900 min-h-screen text-white flex items-center justify-center">載入中...</div>;

            return (
                <div className="bg-gray-900 min-h-screen text-white relative font-sans">
                    
                    {!isLoggedIn ? (
                        <button
                            onClick={() => setIsLoginModalOpen(true)}
                            className="fixed top-4 right-4 z-50 px-4 py-2 sm:px-6 sm:py-2 bg-blue-600 hover:bg-blue-500 text-white font-bold rounded-full shadow-lg border-2 border-white/20 transition transform hover:scale-105 text-sm sm:text-base"
                        >
                            會員登入
                        </button>
                    ) : (
                        <div className="fixed top-4 right-4 z-50 flex items-center gap-2 sm:gap-3">
                            <button 
                                onClick={() => setIsProfileModalOpen(true)}
                                className="bg-gray-800/90 px-3 py-1.5 sm:px-4 sm:py-2 rounded-full border border-green-500/50 text-xs sm:text-sm hover:bg-gray-700 transition cursor-pointer"
                            >
                                <span className="text-gray-400">Hi, </span>
                                <span className="text-green-400 font-bold">{userProfile?.nickname || 'User'}</span>
                            </button>
                            <button
                                onClick={handleLogout}
                                className="p-1.5 sm:p-2 bg-red-600 hover:bg-red-500 text-white rounded-full shadow-lg transition"
                                title="登出"
                            >
                                <LogOut className="w-4 h-4 sm:w-5 sm:h-5" />
                            </button>
                        </div>
                    )}

                    <LoginModal 
                        isOpen={isLoginModalOpen} 
                        onClose={() => setIsLoginModalOpen(false)} 
                        onLoginSuccess={handleLoginSuccess}
                    />
                    
                    <UserProfileModal 
                        isOpen={isProfileModalOpen}
                        onClose={() => setIsProfileModalOpen(false)}
                        userProfile={userProfile}
                        onUpdateProfile={handleUpdateProfile}
                    />

                    <div className="max-w-screen-lg mx-auto p-4 sm:p-8 pt-20 sm:pt-24">
                        
                        <BannerSection profile={userProfile} isLoggedIn={isLoggedIn} handleSignOut={handleLogout} />

                        <PreRegistrationSection />

                        <MatchTable 
                            matches={matches} 
                            predictions={predictions} 
                            setSelectedMatch={(m) => {
                                if (isLoggedIn) setSelectedMatch(m);
                                else alert("請先登入會員，即可參與預測！");
                            }}
                            isLoggedIn={isLoggedIn}
                        />

                        <PrizeProgress 
                            totalWins={userProfile?.successfulPredictions || 0} 
                            prizesStatus={userProfile?.prizesStatus || {}}
                            isLoggedIn={isLoggedIn}
                        />

                        <ReferralRecruitment 
                            referralCode={userProfile?.referralCode}
                            referredCount={userProfile?.referredUsers || 0}
                            prizesStatus={userProfile?.prizesStatus || {}}
                            isLoggedIn={isLoggedIn}
                            handleCopyReferral={handleCopyReferral}
                        />

                        {selectedMatch && isLoggedIn && (
                            <PredictionPopup 
                                match={selectedMatch}
                                prediction={predictions[selectedMatch.id]}
                                isOpen={!!selectedMatch}
                                onClose={() => setSelectedMatch(null)}
                                onPredict={handlePredict}
                            />
                        )}
                    </div>
                </div>
            );
        };

        const root = createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
