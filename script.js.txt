const firebaseConfig = { databaseURL: "https://tirfe-app-v2-300c2-default-rtdb.firebaseio.com/" };
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

let localDB = { tenants: {}, buyers: {} };
let currentTenant = null;
let currentUserRole = "owner";
let myChart = null;
let currentLoginMode = "merchant"; 
let isOnline = true;
let activeCategoryFilter = "all";
let currentBuyer = null;

window.mainCart = [];
window.buyerCartData = [];

window.addEventListener('online', handleOnlineStatus);
window.addEventListener('offline', handleOnlineStatus);

// --- SECURITY FIX: XSS Protection Function ---
function eHTML(str) {
    if (str === null || str === undefined) return '';
    return String(str).replace(/[&<>'"]/g, function (tag) {
        return { '&': '&', '<': '<', '>': '>', "'": '&#39;', '"': '&quot;' }[tag] || tag;
    });
}

function processImageUpload(file, callback) {
    if(!file) { callback(""); return; }
    let reader = new FileReader();
    reader.onload = function(e) {
        let img = new Image();
        img.onload = function() {
            let canvas = document.createElement('canvas');
            let MAX_WIDTH = 500; let MAX_HEIGHT = 500;
            let width = img.width; let height = img.height;
            if(width > height) { if(width > MAX_WIDTH) { height *= MAX_WIDTH / width; width = MAX_WIDTH; } }
            else { if(height > MAX_HEIGHT) { width *= MAX_HEIGHT / height; height = MAX_HEIGHT; } }
            canvas.width = width; canvas.height = height;
            let ctx = canvas.getContext('2d');
            ctx.drawImage(img, 0, 0, width, height);
            callback(canvas.toDataURL('image/jpeg', 0.6));
        };
        img.src = e.target.result;
    };
    reader.readAsDataURL(file);
}

function viewImageFullscreen(src) {
    if(!src) return;
    let modal = document.getElementById('imageLightbox');
    document.getElementById('lightboxImg').src = src;
    modal.classList.remove('hidden');
}

function getTodayFormatted() {
    let todayObj = new Date();
    let yyyy = todayObj.getFullYear();
    let mm = String(todayObj.getMonth() + 1).padStart(2, '0');
    let dd = String(todayObj.getDate()).padStart(2, '0');
    return `${yyyy}-${mm}-${dd}`;
}

function handleOnlineStatus() {
    isOnline = navigator.onLine;
    const tag = document.getElementById('syncIndicator');
    const criticalScreen = document.getElementById('criticalOfflineScreen');
    
    if(!isOnline) {
        if(tag) tag.classList.remove('hidden');
        if(criticalScreen) criticalScreen.classList.remove('hidden');
    } else {
        if(tag) tag.classList.add('hidden');
        if(criticalScreen) criticalScreen.classList.add('hidden');
        pushToFirebase();
    }
}

function setCategoryFilter(cat) {
    activeCategoryFilter = cat;
    renderBuyerCatalog();
}

function loadLocalStorageBackup() {
    let backup = localStorage.getItem('tirfe_local_db');
    if(backup) {
        localDB = JSON.parse(backup);
        if(!localDB.buyers) localDB.buyers = {};
    }
}

function saveToLocalStorage() {
    localStorage.setItem('tirfe_local_db', JSON.stringify(localDB));
}

function sendTelegramAlert(message) {
    if (!currentTenant || !currentTenant.telegramToken || !currentTenant.telegramChatId) return;
    const token = currentTenant.telegramToken;
    const chatId = currentTenant.telegramChatId;
    const url = `https://api.telegram.org/bot${token}/sendMessage?chat_id=${chatId}&text=${encodeURIComponent(message)}`;
    fetch(url).catch(err => console.log("Telegram Error: ", err));
}

loadLocalStorageBackup();
checkAutomaticLogin();
handleOnlineStatus();

db.ref('tirfe_system').on('value', (snapshot) => {
    if(snapshot.exists()) {
        localDB = snapshot.val();
        if(!localDB.tenants) localDB.tenants = {};
        if(!localDB.buyers) localDB.buyers = {};
        saveToLocalStorage();
        
        if(currentTenant) {
            let checkTenant = localDB.tenants[currentTenant.username];
            if(!checkTenant || checkTenant.status === "blocked") { logout(); return; }
            currentTenant = checkTenant;
            renderApp();
        }
        
        if(currentBuyer) {
            let checkBuyer = localDB.buyers[currentBuyer.username];
            if(checkBuyer) currentBuyer = checkBuyer;
        }
        renderBuyerCatalog();
        if(!document.getElementById('adminPage').classList.contains('hidden')) { renderAdminPanel(); }
    }
}, (error) => {
    console.log("Firebase Error, running offline mode.");
    isOnline = false;
    handleOnlineStatus();
});

function checkAutomaticLogin() {
    let savedSessionStr = localStorage.getItem('tirfe_active_session');
    if (savedSessionStr) {
        try {
            let session = JSON.parse(atob(savedSessionStr));
            currentUserRole = session.role;
            currentLoginMode = session.loginMode;
            
            if (session.role === 'admin') {
                setTimeout(() => { switchView('adminPage'); renderAdminPanel(); }, 300);
            } else if (session.role === 'buyer' && localDB.buyers && localDB.buyers[session.username]) {
                if(localDB.buyers[session.username].status === "blocked") {
                    localStorage.removeItem('tirfe_active_session');
                } else {
                    currentBuyer = localDB.buyers[session.username];
                    setTimeout(() => { switchView('buyerPage'); }, 300);
                }
            } else if (localDB.tenants && localDB.tenants[session.username]) {
                let t = localDB.tenants[session.username];
                currentTenant = t;
                setTimeout(() => { launchApp(t); }, 300);
            }
        } catch(e) { localStorage.removeItem('tirfe_active_session'); }
    }
}

function checkTimeLock() {
    if(!currentTenant || !currentTenant.data || currentUserRole === "staff") return;
    let h = new Date().getHours();
    let isLockTime = (h >= 22 || h < 6);
    let d = currentTenant.data;
    if (isLockTime) {
        if (!d.shiftClosed) {
            document.getElementById('shiftStatusAlert').classList.remove('hidden');
            document.getElementById('shiftStatusAlert').innerHTML = "⚠️ ማታ 4:00 (10:00 PM) ሆኗል! ሲስተሙ ተቆልፏል፣ እባክዎ የዕለቱን ሂሳብ ወዲያውኑ ይዝጉ!";
            disableAllActionsExceptClose();
        } else {
            document.getElementById('shiftStatusAlert').classList.remove('hidden');
            document.getElementById('shiftStatusAlert').innerHTML = "🔒 ሲስተሙ የዕለት ሪፖርት ተቀብሎ ተቆልፏል። ጧት 12:00 (6:00 AM) ላይ ይከፈታል።";
            disableAllActions();
        }
    } else {
        document.getElementById('shiftStatusAlert').classList.add('hidden');
        enableAllActions();
    }
}

function disableAllActionsExceptClose() {
    const btns = ['btn_add_item', 'btn_main_sell', 'btn_expense', 'btn_credit', 'btn_draw', 'btn_settlement', 'btn_next_day'];
    btns.forEach(id => { let b = document.getElementById(id); if(b) {b.disabled = true;} });
}

function disableAllActions() {
    const btns = ['btn_add_item', 'btn_main_sell', 'btn_expense', 'btn_credit', 'btn_draw', 'btn_settlement', 'btn_next_day', 'btn_close_shift'];
    btns.forEach(id => { let b = document.getElementById(id); if(b) {b.disabled = true;} });
}

function enableAllActions() {
    const btns = ['btn_add_item', 'btn_main_sell', 'btn_expense', 'btn_credit', 'btn_draw', 'btn_settlement', 'btn_next_day', 'btn_close_shift'];
    btns.forEach(id => { let b = document.getElementById(id); if(b) {b.disabled = false;} });
}

setInterval(() => { checkTimeLock(); }, 60000);

function switchView(targetId) {
    document.getElementById('welcomeGateway').classList.add('hidden');
    document.getElementById('buyerLoginBox').classList.add('hidden');
    document.getElementById('loginPage').classList.add('hidden');
    document.getElementById('buyerPage').classList.add('hidden');
    document.getElementById('adminPage').classList.add('hidden');
    document.getElementById('appPage').classList.add('hidden');
    document.getElementById(targetId).classList.remove('hidden');
    if (targetId === 'buyerPage') renderBuyerCatalog();
}

function showBuyerLoginSection() {
    document.getElementById('welcomeGateway').classList.add('hidden');
    document.getElementById('buyerLoginBox').classList.remove('hidden');
}

function handleBuyerAuth() {
    let u = document.getElementById('buyerUsername').value.trim().toLowerCase();
    let p = document.getElementById('buyerPhone').value.trim();
    let err = document.getElementById('buyerLoginError');
    
    if(!u || !p) { err.innerText = "❌ እባክዎ ዩዘርኔም እና ስልክ ቁጥር በትክክል ያስገቡ!"; return; }

    if(!localDB.buyers) localDB.buyers = {};
    if(localDB.buyers[u]) {
        if(localDB.buyers[u].status === "blocked") {
            err.innerText = "❌ አካውንትዎ በአስተዳዳሪ ታግዷል (Blocked)!";
            return;
        }
        if(localDB.buyers[u].phone !== p) {
            err.innerText = "❌ ይህ ዩዘርኔም አስቀድሞ በሌላ ሰው ተይዟል! እባክዎ ሌላ የተለየ ዩዘርኔም ይምረጡ።";
            return;
        }
    } else {
        localDB.buyers[u] = { username: u, phone: p, joinDate: new Date().getTime(), receipts: [], status: "active" };
        pushToFirebase();
    }

    currentBuyer = localDB.buyers[u];
    localStorage.setItem('tirfe_active_session', btoa(JSON.stringify({ role: 'buyer', loginMode: 'buyer', username: u })));
    err.innerText = "";
    switchView('buyerPage');
}

function logoutBuyer() {
    currentBuyer = null;
    localStorage.removeItem('tirfe_active_session');
    switchView('welcomeGateway');
}

function checkAdminPrivacyKey() {
    showFormModal("🔒 የማስተር ሴኪዩሪቲ ማረጋገጫ", [
        { id: "masterKey", label: "እባክዎ የአከራይ ማስተር ሴኪዩሪቲ ኮድ (Master Security Key) ያስገቡ፦", type: "password", placeholder: "ማስተር ኮድ" }
    ], (res) => {
        if (btoa(res.masterKey) === "dGlyZmUtc2VjdXJlLTIwMjY=") {
            showLoginSection('admin');
        } else {
            showCustomAlert("⛔ መግባት አልተፈቀደም", "ያስገቡት ማስተር ሴኪዩሪቲ ኮድ የተሳሳተ ስለሆነ ገጹን መክፈት አይችሉም!");
        }
    });
}

function showLoginSection(mode) {
    currentLoginMode = mode;
    document.getElementById('loginError').innerText = "";
    document.getElementById('loginUser').value = "";
    document.getElementById('loginPass').value = "";
    
    if (mode === 'admin') {
        document.getElementById('loginTitle').innerText = "👑 የባለቤት (Admin) መግቢያ በር";
        document.getElementById('loginDesc').innerText = "አዳዲስ ሱቆችን ለመመዝገብ እና ውል ለማስተዳደር ይግቡ";
        document.getElementById('loginUser').placeholder = "የአስተዳዳሪ ስም (Admin Username)";
        document.getElementById('loginPass').placeholder = "የአስተዳዳሪ ይለፍ ቃል (Password)";
    } else if (mode === 'merchant') {
        document.getElementById('loginTitle').innerText = "💼 የሱቅ ባለቤቶች (Tenant) መግቢያ";
        document.getElementById('loginDesc').innerText = "የሱቅዎን ሂሳብ፣ ትርፍ እና ክምችት በባለቤትነት ለመቆጣጠር ይግቡ";
        document.getElementById('loginUser').placeholder = "የሱቅ ባለቤት ስም (Tenant Username)";
        document.getElementById('loginPass').placeholder = "የባለቤት ይለፍ ቃል (Password)";
    } else if (mode === 'staff') {
        document.getElementById('loginTitle').innerText = "🛠️ የሱቅ ሰራተኞች መግቢያ በር";
        document.getElementById('loginDesc').innerText = "በሱቅ ባለቤቱ የተሰጠዎትን የሰራተኛ መግቢያ ስም እና ኮድ ይጠቀሙ";
        document.getElementById('loginUser').placeholder = "የሰራተኛ ስም (Staff Username)";
        document.getElementById('loginPass').placeholder = "የሰራተኛ ይለፍ ቃል (Staff Password)";
    }
    switchView('loginPage');
}

function goToGateway() { switchView('welcomeGateway'); }

function handleLogin() {
    let user = document.getElementById('loginUser').value.trim().toLowerCase();
    let pass = document.getElementById('loginPass').value.trim();
    let error = document.getElementById('loginError');

    if(currentLoginMode === 'admin') {
        if(btoa(user) === "YWRtaW4=" && btoa(pass) === "YWRtaW4xMjM=") {
            localStorage.setItem('tirfe_active_session', btoa(JSON.stringify({ role: 'admin', loginMode: 'admin', username: 'admin' })));
            switchView('adminPage');
            renderAdminPanel();
        } else {
            error.innerText = "❌ የተሳሳተ የአስተዳዳሪ (Admin) ስም ወይም ይለፍ ቃል ነው!";
        }
        return;
    }

    let tenant = localDB.tenants ? localDB.tenants[user] : null;
    if (currentLoginMode === 'merchant') {
        if (tenant) {
            if (isTenantExpired(tenant, error)) return;
            if (!tenant.isActivated) {
                if (tenant.activationCode === pass) {
                    let now = new Date().getTime();
                    let twentyFourHours = 24 * 60 * 60 * 1000;
                    if ((now - tenant.codeCreatedAt) > twentyFourHours) {
                        error.innerText = "❌ ይህ ጊዜያዊ ኮድ ከ24 ሰዓት በላይ ስለቆየ ጊዜው አልፏል! እባክዎ አከራዩን አዲስ ኮድ ይጠይቁ።";
                        return;
                    }
                    
                    showFormModal("🔒 የፕራይቬሲ ፓስዎርድ ማቀናበሪያ", [
                        { id: "securePass", label: "ለእርስዎ ብቻ የሚሆን ጠንካራ ምስጢራዊ ፓስዎርድ ይፍጠሩ (አከራዩ ማየት አይችልም)፦", type: "password", placeholder: "አዲስ ፓስዎርድ" }
                    ], (res) => {
                        let sp = res.securePass.trim();
                        if (!sp) { showCustomAlert("ስህተት", "የይለፍ ቃል ባዶ መሆን አይችልም!"); return; }
                        tenant.password = sp; 
                        tenant.isActivated = true;
                        localDB.tenants[user] = tenant;
                        pushToFirebase();
                        currentUserRole = "owner";
                        localStorage.setItem('tirfe_active_session', btoa(JSON.stringify({ role: 'owner', loginMode: 'merchant', username: user })));
                        launchApp(tenant);
                    });
                    return;
                } else {
                    error.innerText = "❌ የተሳሳተ ጊዜያዊ መግቢያ ኮድ ነው!";
                    return;
                }
            } else {
                if (String(tenant.password).trim() === pass) {
                    currentUserRole = "owner";
                    localStorage.setItem('tirfe_active_session', btoa(JSON.stringify({ role: 'owner', loginMode: 'merchant', username: user })));
                    launchApp(tenant);
                    return;
                }
            }
        }
        error.innerText = "❌ የተሳሳተ የሱቅ ባለቤት ስም ወይም የይለፍ ቃል! (ወይም ገና አልነቃም)";
        return;
    }

    if (currentLoginMode === 'staff') {
        if(localDB.tenants) {
            for (let tKey in localDB.tenants) {
                let t = localDB.tenants[tKey];
                if (t.staffUser && t.staffUser === user && String(t.staffPass).trim() === pass) {
                    if (isTenantExpired(t, error)) return;
                    currentUserRole = "staff";
                    localStorage.setItem('tirfe_active_session', btoa(JSON.stringify({ role: 'staff', loginMode: 'staff', username: t.username })));
                    launchApp(t);
                    return;
                }
            }
        }
        error.innerText = "❌ የሰራተኛ ስም ወይም የይለፍ ቃል ስህተት ነው!";
        return;
    }
}

function isTenantExpired(tenant, errorElement) {
    if(tenant.expiryDate) {
        let today = new Date();
        today.setHours(0,0,0,0);
        let expiry = new Date(tenant.expiryDate); expiry.setHours(0,0,0,0);
        if(today > expiry) {
            tenant.status = "blocked";
            localDB.tenants[tenant.username] = tenant;
            pushToFirebase();
            errorElement.innerText = "🔒 የኪራይ ውልዎ ጊዜ አልቋል! እባክዎ ባለቤቱን ያነጋግሩ።";
            return true;
        }
    }
    if(tenant.status === "blocked") { errorElement.innerText = "🔒 አካውንትዎ ታግዷል!";
    return true; }
    return false;
}

function checkMonthlyAccessReset() {
    if (!currentTenant || !currentTenant.data) return;
    let now = new Date();
    let currentTimestamp = now.getTime();
    
    if (!currentTenant.data.lastMonthlyResetDate) {
        currentTenant.data.lastMonthlyResetDate = currentTenant.codeCreatedAt || currentTimestamp;
        saveAndRefresh();
        return;
    }
    
    let diffTime = Math.abs(currentTimestamp - currentTenant.data.lastMonthlyResetDate);
    let diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
    if (diffDays >= 30) {
        let d = currentTenant.data;
        let expensesList = d.expenses || [];
        let totalMonthlyExp = 0;
        expensesList.forEach(e => totalMonthlyExp += parseFloat(e.amount) || 0);
        let totalMonthlySales = 0;
        let totalMonthlyProfit = 0;
        let inv = d.inventory || [];
        inv.forEach(item => {
            totalMonthlySales += (item.price * item.sold);
            totalMonthlyProfit += (item.price - item.cost) * item.sold;
        });
        let finalMonthlyNetProfit = totalMonthlyProfit - totalMonthlyExp;
        
        if(!d.history) d.history = [];
        
        let lastResetObj = new Date(d.lastMonthlyResetDate);
        let formattedPeriod = `${lastResetObj.toLocaleDateString('en-GB')} - ${now.toLocaleDateString('en-GB')}`;
        
        d.history.push({
            date: "የወር ማጠቃለያ", employee: formattedPeriod, sales: totalMonthlySales, profit: finalMonthlyNetProfit,
            expenses: totalMonthlyExp, draws: 0, reportedCash: d.reportedCash || 0, expectedCash: d.expectedCash || 0,
            variance: d.variance || 0, isMonthlyArchive: true
        });
        d.expenses = [];
        d.lastMonthlyResetDate = currentTimestamp; 
        
        localDB.tenants[currentTenant.username] = currentTenant;
        saveToLocalStorage(); pushToFirebase();
        showCustomAlert("📅 አዲስ ወር ጀምሯል", `ያለፈው 30 ቀናት የሱቅ ወጪና የሂሳብ መረጃዎች ተጠቅልለው ማህደር (Archive) ውስጥ ገብተዋል። ለአዲሱ ወር ወጪው ከ 0 ተጀምሯል።`);
    }
}

function launchApp(tenant) {
    currentTenant = tenant;
    document.getElementById('loginPage').classList.add('hidden');
    document.getElementById('appPage').classList.remove('hidden');
    
    document.getElementById('shopTitle').innerText = eHTML(tenant.shopName) + (currentUserRole === "staff" ? " (የሰራተኛ ገጽ)" : " (የባለቤት ገጽ)");
    document.getElementById('roleSubTitle').innerText = currentUserRole === "staff" ? "🛠️ የተገደበ የሰራተኛ መሸጫ እና መመዝገቢያ ሞድ" : "👑 ሙሉ የሱቅና የኪራይ መቆጣጠሪያ ፓነል";
    document.getElementById('profShopName').innerText = eHTML(tenant.shopName);
    document.getElementById('profExpiry').innerText = tenant.expiryDate ? `${eHTML(tenant.expiryDate)} (${eHTML(tenant.contractType)})` : "ያልተገደበ";
    document.getElementById('receiptDateFilter').value = getTodayFormatted();
    let activeTheme = tenant.theme || 'theme-deepblue';
    document.body.className = activeTheme;
    document.getElementById('themeSelector').value = activeTheme;
    document.getElementById('inventorySearchInput').value = "";
    
    if (currentUserRole === "staff") {
        document.getElementById('ownerDashboard').classList.add('hidden');
        document.getElementById('chartContainer').classList.add('hidden');
        document.getElementById('btn_add_item').classList.add('hidden');
        document.getElementById('btn_expense').classList.add('hidden');
        document.getElementById('btn_next_day').classList.add('hidden');
        document.getElementById('btn_clear_all').classList.add('hidden');
        document.getElementById('owner_add_box').classList.add('hidden');
        document.getElementById('ownerStaffConfig').classList.add('hidden');
        document.getElementById('expiryRowOwner').classList.add('hidden');
        document.getElementById('btn_settlement').classList.add('hidden');
        document.getElementById('btn_profile_editor').classList.add('hidden');
        document.getElementById('historySection').classList.add('hidden');
    } else {
        document.getElementById('ownerDashboard').classList.remove('hidden');
        document.getElementById('chartContainer').classList.remove('hidden');
        document.getElementById('btn_add_item').classList.remove('hidden');
        document.getElementById('btn_expense').classList.remove('hidden');
        document.getElementById('btn_next_day').classList.remove('hidden');
        document.getElementById('btn_clear_all').classList.remove('hidden');
        document.getElementById('owner_add_box').classList.remove('hidden');
        document.getElementById('ownerStaffConfig').classList.remove('hidden');
        document.getElementById('expiryRowOwner').classList.remove('hidden');
        document.getElementById('btn_settlement').classList.remove('hidden');
        document.getElementById('btn_profile_editor').classList.remove('hidden');
        document.getElementById('historySection').classList.remove('hidden');
        
        document.getElementById('staffUser').value = tenant.staffUser || "";
        document.getElementById('staffPass').value = tenant.staffPass || "";
        
        checkMonthlyAccessReset();
    }

    setTimeout(() => {
        if(currentUserRole === "owner") initChart();
        checkMorningSession();
    }, 200);
}

window.openDeliveryOrderModal = function(shopKey, itemIdx, itemName, price) {
    if(!currentBuyer) { showCustomAlert("ማሳሰቢያ", "እባክዎ መጀመሪያ እንደ ገዥ ይግቡ/ይመዝገቡ!"); return; }
    showFormModal("🚚 " + eHTML(itemName) + " - ዴሊቨሪ ማዘዣ", [
        { id: "phone", label: "ስልክ ቁጥርዎ", type: "text", defaultValue: currentBuyer.phone },
        { id: "address", label: "ያሉበት ትክክለኛ አድራሻ / ሰፈር", type: "text", placeholder: "ምሳሌ: ቦሌ ሚካኤል፣ ህንፃ 3..." },
        { id: "mapLink", label: "የጎግል ማፕ ሊንክ (አማራጭ)", type: "text", placeholder: "https://maps.google.com/..." },
        { id: "qty", label: "የሚፈልጉት ብዛት", type: "number", defaultValue: "1" }
    ], (res) => {
        let qty = parseFloat(res.qty) || 0;
        if(qty <= 0 || !res.address) { showCustomAlert("ስህተት", "እባክዎ አድራሻዎን እና የሚፈልጉትን ብዛት በትክክል ይሙሉ!"); return; }

        let t = localDB.tenants[shopKey];
        if(!t.data.deliveryOrders) t.data.deliveryOrders = [];

        let orderId = Math.floor(100000 + Math.random() * 900000);
        t.data.deliveryOrders.push({
            orderId: orderId, buyerUser: currentBuyer.username, buyerPhone: res.phone,
            address: res.address, mapLink: res.mapLink, itemIdx: itemIdx, itemName: itemName,
            qty: qty, price: price, total: qty * price, status: "pending", date: getTodayFormatted()
        });
        localDB.tenants[shopKey] = t; pushToFirebase();
        showCustomAlert("ተሳክቷል", "ትዕዛዝዎ ለሻጩ ተልኳል። ሻጩ ሲቀበለው በገጽዎ ላይ 'በመንገድ ላይ ነው' የሚል ምልክት ያያሉ።");
        renderBuyerCatalog();
    });
};

window.buyFromShop = function(shopKey, itemIdx, itemName, price, availableRem) {
    if(!currentBuyer) { showCustomAlert("ማሳሰቢያ", "እባክዎ መጀመሪያ እንደ ገዥ ይግቡ/ይመዝገቡ!"); return; }
    showFormModal("🛒 " + eHTML(itemName) + " - ወደ ቅርጫት (Cart) ማስገቢያ", [
        { id: "qty", label: "የሚፈልጉት ብዛት", type: "number", defaultValue: "1" }
    ], (res) => {
        let qty = parseFloat(res.qty) || 0;
        if(qty <= 0) { showCustomAlert("ስህተት", "የተሳሳተ ብዛት!"); return; }
        if(qty > availableRem) { showCustomAlert("ብዛት የለም", "የጠየቁት ብዛት በአሁኑ ሰዓት ከስቶር የለም (አልቋል)!"); return; }

        let existIdx = window.buyerCartData.findIndex(c => c.shopKey === shopKey && c.itemIdx === itemIdx);
        if(existIdx > -1) {
            let totalWanted = window.buyerCartData[existIdx].qty + qty;
            if(totalWanted > availableRem) { showCustomAlert("ስህተት", "ከክምችት በላይ ነው!"); return; }
            window.buyerCartData[existIdx].qty += qty;
            window.buyerCartData[existIdx].total = window.buyerCartData[existIdx].qty * price;
        } else {
            window.buyerCartData.push({
                shopKey: shopKey, itemIdx: itemIdx, itemName: itemName, qty: qty, price: price, total: qty * price
            });
        }
        renderBuyerCart();
        showCustomAlert("🛒 በቅርጫትዎ ውስጥ ገብቷል", "ትዕዛዙ Cart ውስጥ ገብቷል። ሲጨርሱ ከላይ 'እርግጠኛ ነኝ ትዕዛዙን ላክ' የሚለውን ይጫኑ።");
    });
};

window.renderBuyerCart = function() {
    let section = document.getElementById('buyerCartSection');
    let listBody = document.getElementById('buyerCartList');
    let totalSumEl = document.getElementById('buyerCartTotalSum');

    if(!window.buyerCartData || window.buyerCartData.length === 0) {
        section.style.display = 'none';
        listBody.innerHTML = '';
        totalSumEl.innerText = "0";
        return;
    }

    section.style.display = 'block';
    listBody.innerHTML = '';
    let grandTotal = 0;

    window.buyerCartData.forEach((c, i) => {
        grandTotal += c.total;
        let shopName = localDB.tenants[c.shopKey] ? localDB.tenants[c.shopKey].shopName : "ሱቅ";
        listBody.innerHTML += `
        <tr style="border-bottom:1px solid rgba(255,255,255,0.05);">
            <td style="color:var(--text-color);"><b>${eHTML(c.itemName)}</b><br><small style="color:var(--accent-color)">[${eHTML(shopName)}]</small></td>
            <td style="color:var(--text-color);">${c.qty}</td>
            <td style="color:var(--success-color);"><b>${c.total}</b></td>
            <td><button class="btn-expense btn-sm" onclick="removeFromBuyerCart(${i})">❌ አጥፋ</button></td>
        </tr>`;
    });
    totalSumEl.innerText = grandTotal;
};

window.removeFromBuyerCart = function(i) {
    if(window.buyerCartData) {
        window.buyerCartData.splice(i, 1);
        renderBuyerCart();
    }
};

window.checkoutBuyerCart = function() {
    if(!window.buyerCartData || window.buyerCartData.length === 0) {
        showCustomAlert("ስህተት", "ምንም ዕቃ አልመረጡም!"); return;
    }
    showCustomConfirm("ትዕዛዝ ማረጋገጫ", "ሁሉንም የቅርጫት ትዕዛዞች ወደየሱቆቹ መላክ ይፈልጋሉ?", () => {
        let shops = {};
        window.buyerCartData.forEach(c => {
            if(!shops[c.shopKey]) shops[c.shopKey] = [];
            shops[c.shopKey].push(c);
        });

        for(let sKey in shops) {
            let t = localDB.tenants[sKey];
            if(!t.data.remoteCarts) t.data.remoteCarts = {};
            if(!t.data.remoteCarts[currentBuyer.username]) t.data.remoteCarts[currentBuyer.username] = [];
            
            shops[sKey].forEach(item => {
                t.data.remoteCarts[currentBuyer.username].push({
                    itemIdx: item.itemIdx, itemName: item.itemName, qty: item.qty, price: item.price, total: item.total
                });
            });
            localDB.tenants[sKey] = t;
        }
        
        window.buyerCartData = []; 
        renderBuyerCart();
        pushToFirebase();
        showCustomAlert("✅ ተሳክቷል", "ትዕዛዞችዎ በተሳካ ሁኔታ ተልከዋል! ሻጮች ሲያረጋግጡ የ'ተቆረጡ ደረሰኞች' ቦታ ላይ ይደርስዎታል።");
    });
};

function renderBuyerCatalog() {
    if(currentBuyer) {
        let badge = document.getElementById('buyerProfileBadge');
        if(badge) badge.innerText = `👤 የተጠቃሚ ስም: ${eHTML(currentBuyer.username)} | 📱 ስልክ: ${eHTML(currentBuyer.phone)}`;
        renderBuyerCart();
    }

    let container = document.getElementById('buyerShopsContainer');
    if(!container) return;
    container.innerHTML = '';
    let hasData = false;
    let query = document.getElementById('buyerSearchInput') ? document.getElementById('buyerSearchInput').value.trim().toLowerCase() : "";
    let categories = new Set();
    if (localDB.tenants) {
        Object.values(localDB.tenants).forEach(t => {
            if (t.status === "active") {
                let bType = t.businessType || "አጠቃላይ ንግድ";
                categories.add(bType);
            }
        });
    }
    
    let catContainer = document.getElementById('buyerCategoryContainer');
    if (catContainer) {
        let catHTML = `<button class="category-btn ${activeCategoryFilter === 'all' ? 'active' : ''}" onclick="setCategoryFilter('all')">🌐 ሁሉም</button>`;
        categories.forEach(cat => {
            catHTML += `<button class="category-btn ${activeCategoryFilter === cat ? 'active' : ''}" onclick="setCategoryFilter('${eHTML(cat)}')">🛍️ ${eHTML(cat)}</button>`;
        });
        catContainer.innerHTML = catHTML;
    }

    let myOrdersHTML = "";
    let myReceiptsHTML = "";
    let liveBuyer = (currentBuyer && localDB.buyers) ? localDB.buyers[currentBuyer.username] : currentBuyer;
    if(liveBuyer && liveBuyer.receipts) {
        let reversed = [...liveBuyer.receipts].reverse();
        let filterDate = document.getElementById('buyerReceiptDateFilter') ? document.getElementById('buyerReceiptDateFilter').value : "";
        reversed.forEach(rec => {
            if (filterDate && rec.date !== filterDate) return;
            myReceiptsHTML += `<tr>
                <td><b>#${rec.recId}</b></td>
                <td>${rec.date}</td>
                <td>${eHTML(rec.itemName)} (${rec.count})</td>
                <td style="color:var(--success-color);"><b>${rec.totalVal} ETB</b></td>
                <td><button class="btn-sm btn-add" onclick="viewBuyerReceipt('${rec.recId}')">📥 አውርድ</button></td>
            </tr>`;
        });
    }

    if (localDB.tenants) {
        Object.keys(localDB.tenants).forEach(tKey => {
            let t = localDB.tenants[tKey];
            if (t.status === "active") {
                let tBType = t.businessType || "አጠቃላይ ንግድ";
                if (activeCategoryFilter !== "all" && tBType !== activeCategoryFilter) return;
                
                let matchingItems = [];
                if (t.data && t.data.inventory) {
                    matchingItems = t.data.inventory.map((item, index) => ({...item, originalIdx: index})).filter(item => {
                        if (query === "") return true;
                        return item.name.toLowerCase().includes(query) || (item.model && item.model.toLowerCase().includes(query));
                    });
                }

                if (query !== "" && matchingItems.length === 0) return;
                hasData = true;

                let shopLogo = t.shopLogo || "https://cdn-icons-png.flaticon.com/512/869/869636.png";
                let tgLink = t.telegram && t.telegram !== "-" ? (t.telegram.startsWith('@') ? t.telegram.substring(1) : t.telegram) : "";
                let shopCardHTML = `
                <div class="shop-card">
                    <div class="shop-card-header">
                        <img src="${shopLogo}" class="shop-avatar" onerror="this.src='https://cdn-icons-png.flaticon.com/512/869/869636.png'">
                        <div class="shop-meta">
                            <h3>${eHTML(t.shopName)}</h3>
                            <p>📍 አድራሻ፡ ${eHTML(t.address || 'ያልተገለጸ')} <br><span style="color:var(--accent-color); font-size:0.75rem;">[${eHTML(tBType)}]</span></p>
                        </div>
                    </div>
                    <div style="margin-top:5px; font-size:0.85rem; color:#94a3b8; font-weight:bold;">📦 ዕቃዎች ዝርዝር፦</div>
                    <div class="shop-items-list">`;
                
                if (matchingItems.length === 0) {
                    shopCardHTML += `<p style="font-size:0.8rem; color:#64748b; padding:5px 0;">በአሁኑ ሰዓት የተመዘገበ ዕቃ የለም።</p>`;
                } else {
                    matchingItems.forEach(item => {
                        let itemImg = item.imgUrl || "https://cdn-icons-png.flaticon.com/512/3342/3342137.png";
                        let modelDisplay = item.model && item.model !== "-" ? `<br><small style="color:var(--accent-color)">ሞዴል: ${eHTML(item.model)}</small>` : '';
                        let unitLabel = item.unitType === 'kg' ? 'ኪሎ' : (item.isAdvanced ? 'ሜትር' : 'ፍሬ');
                        let rem = item.qty - item.sold;

                        shopCardHTML += `
                        <div class="catalog-item-card">
                            <img src="${itemImg}" class="catalog-item-img" onclick="viewImageFullscreen('${itemImg}')" onerror="this.src='https://cdn-icons-png.flaticon.com/512/3342/3342137.png'">
                            <div class="catalog-item-info">
                                <span style="font-weight:bold; font-size:0.9rem;">${eHTML(item.name)}</span>${modelDisplay}
                                <div style="color:var(--warning-color); font-weight:bold; margin-top:2px;">${item.price} ETB <small>(${unitLabel})</small></div>
                                <div style="display:flex; gap:5px; margin-top:5px; flex-wrap:wrap;">
                                    <button class="btn-add btn-sm" onclick="openDeliveryOrderModal('${tKey}', ${item.originalIdx}, '${eHTML(item.name).replace(/'/g, "\\'")}', ${item.price})">🚚 ዴሊቨሪ</button>
                                    <button class="btn-success btn-sm" style="background:var(--warning-color); color:#000;" onclick="buyFromShop('${tKey}', ${item.originalIdx}, '${eHTML(item.name).replace(/'/g, "\\'")}', ${item.price}, ${rem})">🛒 ሱቅ ነኝ ግዛ</button>
                                </div>
                            </div>
                        </div>`;
                    });
                }

                shopCardHTML += `
                    </div>
                    <div class="shop-links">
                        <a href="tel:${eHTML(t.phone)}" class="btn-link-action" style="background:#22c55e; color:#fff;">📞 ስልክ፡ ${eHTML(t.phone)}</a>
                        ${tgLink ? `<a href="https://t.me/${eHTML(tgLink)}" target="_blank" class="btn-link-action" style="background:#0088cc; color:#fff;">✈️ ቴሌግራም</a>` : `<span class="btn-link-action" style="background:#334155; color:#64748b;">✈️ ቴሌግራም የለም</span>`}
                        ${t.googleMapsLink ? `<a href="${eHTML(t.googleMapsLink)}" target="_blank" class="btn-link-action" style="background:var(--accent-color); color:#000; grid-column: span 2; margin-top:4px;">📍 ጎግል ማፕ (Google Maps)</a>` : `<span class="btn-link-action" style="background:#334155; color:#64748b; grid-column: span 2; margin-top:4px;">📍 ሎኬሽን አልተጫነም</span>`}
                    </div>
                </div>`;
                container.innerHTML += shopCardHTML;

                if(liveBuyer && t.data && t.data.deliveryOrders) {
                    t.data.deliveryOrders.forEach(ord => {
                        if(ord.buyerUser === liveBuyer.username) {
                            let st = ord.status;
                            let badge = st === "pending" ? "በመጠባበቅ ላይ" : (st === "accepted" ? "በመንገድ ላይ" : (st === "completed" ? "ተረክበዋል" : "ተመልሷል"));
                            let cl = st === "pending" ? "text-warning" : (st === "accepted" ? "text-success" : "text-danger");
                            myOrdersHTML += `<tr>
                                <td>${eHTML(t.shopName)}</td>
                                <td>${eHTML(ord.itemName)} (x${ord.qty})</td>
                                <td>${ord.total} ETB</td>
                                <td>${ord.date}</td>
                                <td class="${cl}"><b>${badge}</b></td>
                            </tr>`;
                        }
                    });
                }
            }
        });
    }

    if(!hasData) {
        container.innerHTML = '<p style="text-align:center; color:#94a3b8; grid-column: 1/-1; padding:20px;">በተፈለገው ስም የተገኘ ምንም ሱቅ ወይም ዕቃ የለም።</p>';
    }

    let ordersBody = document.getElementById('buyerOrdersBody');
    if(ordersBody) {
        if(myOrdersHTML === "") ordersBody.innerHTML = `<tr><td colspan="5" style="text-align:center; color:#94a3b8;">ምንም የዴሊቨሪ ትዕዛዝ አልጠየቁም።</td></tr>`;
        else ordersBody.innerHTML = myOrdersHTML;
    }

    let receiptsBody = document.getElementById('buyerReceiptsBody');
    if(receiptsBody) {
        if(myReceiptsHTML === "") receiptsBody.innerHTML = `<tr><td colspan="5" style="text-align:center; color:#94a3b8;">የተቆረጠ ደረሰኝ የለም።</td></tr>`;
        else receiptsBody.innerHTML = myReceiptsHTML;
    }
}

window.viewBuyerReceipt = function(recId) {
    if (!currentBuyer || !localDB.buyers[currentBuyer.username]) return;
    let latestBuyerData = localDB.buyers[currentBuyer.username];
    if (!latestBuyerData.receipts) return;

    let rec = latestBuyerData.receipts.find(r => r.recId === parseInt(recId) || r.recId == recId);
    if(!rec) {
        showCustomAlert("ስህተት", "ይህ ደረሰኝ አልተገኘም!");
        return;
    }
    
    let bName = latestBuyerData.username;
    let bPhone = latestBuyerData.phone;
    if(rec.advancedItems) { 
        generateAdvancedReceipt(rec.advancedItems, rec.totalVal, rec.seller, rec.recId, false, rec.shopName, rec.bType, bName, bPhone);
    } else { 
        generateAdvancedReceipt([{name: rec.itemName, count: rec.count, unitPrice: rec.totalVal/rec.count, total: rec.totalVal}], rec.totalVal, rec.seller, rec.recId, false, rec.shopName, rec.bType, bName, bPhone);
    }
};

function generateRandomCode() { return Math.floor(100000 + Math.random() * 900000).toString(); }

function registerTenant() {
    let shop = document.getElementById('newShopName').value.trim();
    let fullName = document.getElementById('newFullName').value.trim();
    let user = document.getElementById('newUsername').value.trim().toLowerCase(); 
    let phone = document.getElementById('newPhone').value.trim();
    let telegram = document.getElementById('newTelegram').value.trim();
    let mapsLink = document.getElementById('newMapsLink').value.trim();
    let address = document.getElementById('newAddress').value.trim();
    let businessType = document.getElementById('newBusinessType').value.trim() || 'አጠቃላይ ንግድ';
    let registrationFee = parseFloat(document.getElementById('newRegistrationFee').value) || 0;
    let contractType = document.getElementById('newContractType').value;
    let expiryDate = document.getElementById('newExpiryDate').value;
    
    if(!shop || !user || !expiryDate || !fullName || !phone) { 
        showCustomAlert("ስህተት", "እባክዎ መሠረታዊ መፈላጊ መረጃዎችን ያሟሉ!"); return; 
    }

    if (localDB.tenants && localDB.tenants[user]) {
        showCustomAlert("⚠️ ምዝገባው አልተሳካም", `"${eHTML(user)}" የሚለው የተጠቃሚ ስም (Username) አስቀድሞ በሌላ ተከራይ ተወስዷል! እባክዎ የተለየ ስም ይጠቀሙ።`);
        return;
    }

    let fileInput = document.getElementById('newShopLogoFile');
    let file = fileInput.files[0];
    let proceedRegistration = function(shopLogoBase64) {
        let genCode = generateRandomCode();
        let timestampNow = new Date().getTime();
        localDB.tenants[user] = { 
            shopName: shop, fullName: fullName, phone: phone, telegram: telegram || "-", address: address || "-",
            businessType: businessType, googleMapsLink: mapsLink || "", shopLogo: shopLogoBase64 || "",
            username: user, password: genCode, activationCode: genCode, codeCreatedAt: timestampNow,
            isActivated: false, contractType: contractType, expiryDate: expiryDate, registrationFee: registrationFee,
            status: "active", theme: "theme-deepblue", staffUser: "", staffPass: "",
            data: { sessionActive: false, shiftClosed: false, inventory: [], expenses: [], debts: [], drawerLog: [], history: [], receipts: [], deliveryOrders: [], remoteCarts: {}, lastMonthlyResetDate: timestampNow } 
        };
        pushToFirebase(); renderAdminPanel();
        
        document.getElementById('newShopName').value = ''; document.getElementById('newFullName').value = '';
        document.getElementById('newUsername').value = ''; document.getElementById('newPhone').value = ''; 
        document.getElementById('newTelegram').value = ''; document.getElementById('newMapsLink').value = '';
        document.getElementById('newAddress').value = ''; document.getElementById('newBusinessType').value = '';
        document.getElementById('newExpiryDate').value = ''; document.getElementById('newRegistrationFee').value = ''; 
        document.getElementById('newShopLogoFile').value = '';
        showCustomAlert("ተሳክቷል", "አዲሱ ተከራይ በተሳካ ሁኔታ ተመዝግቧል!");
    };

    if(file) processImageUpload(file, proceedRegistration);
    else proceedRegistration("");
}

function openAdminTenantEditor(user) {
    let t = localDB.tenants[user];
    showFormModal(`✍️ የተከራይ መረጃ ማሻሻያ (${eHTML(t.shopName)})`, [
        { id: "shopName", label: "የሱቅ ስም", type: "text", defaultValue: t.shopName },
        { id: "fullName", label: "የተከራይ ሙሉ ስም", type: "text", defaultValue: t.fullName },
        { id: "phone", label: "ስልክ ቁጥር", type: "text", defaultValue: t.phone },
        { id: "telegram", label: "ቴሌግራም", type: "text", defaultValue: t.telegram },
        { id: "mapsLink", label: "ጎግል ማፕ ሊንክ", type: "text", defaultValue: t.googleMapsLink || "" },
        { id: "address", label: "አድራሻ (ሀገር/ከተማ)", type: "text", defaultValue: t.address },
        { id: "businessType", label: "የንግድ ዘርፍ", type: "text", defaultValue: t.businessType || "አጠቃላይ ንግድ" },
        { id: "registrationFee", label: "የመመዝገቢያ/ኪራይ ክፍያ (ETB)", type: "number", defaultValue: t.registrationFee || 0 },
        { id: "expiryDate", label: "የውል ማቂያ ቀን", type: "date", defaultValue: t.expiryDate }
    ], (res) => {
        t.shopName = res.shopName.trim();
        t.fullName = res.fullName.trim();
        t.phone = res.phone.trim(); t.telegram = res.telegram.trim();
        t.googleMapsLink = res.mapsLink.trim(); t.address = res.address.trim();
        t.businessType = res.businessType.trim();
        t.registrationFee = parseFloat(res.registrationFee) || 0;
        t.expiryDate = res.expiryDate;
        localDB.tenants[user] = t; pushToFirebase(); renderAdminPanel();
        showCustomAlert("ተሳክቷል", "የተከራዩ መረጃ በተሳካ ሁኔታ ተሻሽሏል!");
    });
}

function openTenantProfileEditor() {
    if(currentUserRole === "staff") { showCustomAlert("ክልክል", "ይህን መረጃ እና የይለፍ ቃል ማስተካከል የሚችለው የሱቁ ባለቤት ብቻ ነው!"); return; }

    showFormModal("⚙️ የሱቅ መረጃ እና ምስጢራዊ ኮድ ማስተካከያ", [
        { id: "shopName", label: "የሱቅ ስም", type: "text", defaultValue: currentTenant.shopName },
        { id: "phone", label: "የሱቅ ስልክ ቁጥር", type: "text", defaultValue: currentTenant.phone },
        { id: "mapsLink", label: "የጎግል ማፕ ሊንክ (Google Maps URL)", type: "text", defaultValue: currentTenant.googleMapsLink || "" },
        { id: "newLogo", label: "የሱቅ ፎቶ/ሎጎ ለመቀየር (አማራጭ)", type: "file" },
        { id: "newPassword", label: "አዲስ ምስጢራዊ ኮድ / ፓስዎርድ ለመቀየር (ባዶ ከሆነ አይቀየርም)", type: "password", placeholder: "አዲስ ነባር ኮድ" }
    ], (res, fileInput) => {
        let updateTenantData = function(base64Logo) {
            currentTenant.shopName = res.shopName.trim();
            currentTenant.phone = res.phone.trim();
            currentTenant.googleMapsLink = res.mapsLink.trim();
            if(base64Logo) currentTenant.shopLogo = base64Logo;
            if (res.newPassword && res.newPassword.trim() !== "") {
                currentTenant.password = res.newPassword.trim();
            }
            saveAndRefresh();
            showCustomAlert("ተሳክቷል", "የሱቅዎ መረጃ በተሳካ ሁኔታ ተስተካክሏል!");
        };

        if(fileInput && fileInput.files[0]) {
            processImageUpload(fileInput.files[0], updateTenantData);
        } else {
            updateTenantData("");
        }
    });
}

function renderAdminPanel() {
    let tbody = document.getElementById('tenantTableBody');
    tbody.innerHTML = '';
    let query = document.getElementById('adminSearchInput') ? document.getElementById('adminSearchInput').value.trim().toLowerCase() : "";

    let totalTenants = 0; let activeTenants = 0;
    let totalFeesCollected = 0;

    Object.keys(localDB.tenants).forEach(key => {
        let t = localDB.tenants[key]; totalTenants++;
        if (t.status === "active") activeTenants++;
        totalFeesCollected += (parseFloat(t.registrationFee) || 0);

        if (query !== "" && !t.username.toLowerCase().includes(query)) return;

        let statusBadge = t.status === "active" ? `<span class="badge-success">Active</span>` : `<span class="badge-danger">Blocked</span>`;
        let profileInfo = `👤 <b>${eHTML(t.fullName || '-')}</b><br>📞 ${eHTML(t.phone || '-')}<br>📍 ${eHTML(t.address || '-')}<br>✈️ ${eHTML(t.telegram || '-')}`;
        
        let codeDisplay = "";
        if (!t.isActivated) {
            codeDisplay = `⏱️ ጊዜያዊ ኮድ: <b class="text-warning" style="font-size:1.1rem; background:rgba(0,0,0,0.4); padding:2px 6px; border-radius:4px;">${eHTML(t.activationCode)}</b>`;
        } else {
            codeDisplay = `<span class="text-success">🔒 ተከራዩ የራሱን ምስጢር ቆልፏል</span>`;
        }
        
        let loginInfo = `👤 አባል ስም: <code>${eHTML(t.username)}</code><br>${codeDisplay}<br>🛠️ ሰራተኛ: <code>${eHTML(t.staffUser || '-')}</code>`;
        let contractDisplay = `<span>${eHTML(t.contractType || 'በወር')}</span><br><b class="text-warning">${t.registrationFee || 0} ETB</b>`;
        let bType = t.businessType || 'አጠቃላይ ንግድ';
        tbody.innerHTML += `<tr>
            <td><b>${eHTML(t.shopName)}</b><br><span style="color:var(--accent-color); font-size:0.8rem;">[${eHTML(bType)}]</span></td>
            <td>${profileInfo}</td>
            <td>${loginInfo}</td>
            <td>${contractDisplay}</td>
            <td style="color:var(--danger-color)"><b>${eHTML(t.expiryDate || '-')}</b></td>
            <td>${statusBadge}</td>
            <td>
                <button class="btn-add btn-sm" onclick="openAdminTenantEditor('${t.username}')">✍️ አሻሽል</button>
                <button class="btn-config btn-sm" onclick="toggleTenantStatus('${t.username}')">ሁኔታ ቀይር</button>
                <button class="btn-expense btn-sm" onclick="deleteTenant('${t.username}')">Delete</button>
                <button class="btn-add btn-sm" style="margin-top:4px;" onclick="regenerateTenantCode('${t.username}')">🔄 አዲስ ኮድ</button>
            </td>
        </tr>`;
    });

    document.getElementById('adminTotalTenants').innerText = totalTenants;
    document.getElementById('adminActiveTenants').innerText = activeTenants;
    document.getElementById('adminTotalFees').innerText = totalFeesCollected.toFixed(1) + " ETB";
    document.getElementById('adminTotalBuyers').innerText = Object.keys(localDB.buyers || {}).length;
    
    renderAdminBuyers();
}

function renderAdminBuyers() {
    let tbody = document.getElementById('adminBuyersTableBody');
    if(!tbody) return;
    tbody.innerHTML = '';
    if(!localDB.buyers) return;
    
    Object.values(localDB.buyers).forEach(b => {
        let status = b.status === "blocked" ? '<span class="badge-danger">Blocked / ታግዷል</span>' : '<span class="badge-success">Active / ይሰራል</span>';
        let actionText = b.status === "blocked" ? "Unblock አድርግ" : "Block አድርግ";
        let actionClass = b.status === "blocked" ? "btn-add" : "btn-expense";
        tbody.innerHTML += `<tr>
            <td>👤 ${eHTML(b.username)}</td>
            <td>📞 ${eHTML(b.phone)}</td>
            <td>${status}</td>
            <td><button class="${actionClass} btn-sm" onclick="toggleBuyerStatus('${b.username}')">🚫 ${actionText}</button></td>
        </tr>`;
    });
}

window.toggleBuyerStatus = function(username) {
    if(localDB.buyers && localDB.buyers[username]) {
        let b = localDB.buyers[username];
        b.status = b.status === "blocked" ? "active" : "blocked";
        pushToFirebase();
        renderAdminBuyers();
        showCustomAlert("ተስተካክሏል", "የገዥው መረጃ ሁኔታ ተቀይሯል።");
    }
};

function regenerateTenantCode(user) {
    let t = localDB.tenants[user];
    let newCode = generateRandomCode();
    t.activationCode = newCode; t.password = newCode; t.codeCreatedAt = new Date().getTime(); t.isActivated = false; 
    localDB.tenants[user] = t;
    pushToFirebase(); renderAdminPanel();
    showCustomAlert("ኮድ ተለውጧል", `ለተከራዩ አዲስ ኮድ ተፈጥሯል፦ ${newCode}`);
}

function toggleTenantStatus(user) {
    let t = localDB.tenants[user];
    t.status = t.status === "active" ? "blocked" : "active"; pushToFirebase(); renderAdminPanel();
}

function deleteTenant(user) { 
    showCustomConfirm("ተከራይ ማጥፊያ", "ይህንን ተከራይ ሙሉ በሙሉ ለማጥፋት እርግጠኛ ኖት?", () => {
        delete localDB.tenants[user]; 
        if(isOnline) db.ref(`tirfe_system/tenants/${user}`).remove();
        pushToFirebase(); renderAdminPanel(); 
    });
}

function logout() { 
    currentTenant = null; localStorage.removeItem('tirfe_active_session'); switchView('welcomeGateway');
}

function saveAndRefresh() { 
    localDB.tenants[currentTenant.username] = currentTenant; saveToLocalStorage(); pushToFirebase();
    renderApp(); checkTimeLock();
}

function addItemDirectly() {
    if(currentUserRole === "staff") return;
    let name = document.getElementById('itemName').value.trim();
    let model = document.getElementById('itemModel').value.trim();
    let cost = parseFloat(document.getElementById('itemCost').value) || 0;
    let price = parseFloat(document.getElementById('itemPrice').value) || 0;
    let qty = parseInt(document.getElementById('itemQty').value) || 0;
    let fileInput = document.getElementById('itemImgFile');
    let file = fileInput.files[0];
    if(!name || cost <= 0 || price <= 0 || qty <= 0) { showCustomAlert("ስህተት", "እባክዎ ትክክለኛ የዕቃ መረጃ ያስገቡ!"); return; }
    
    let proceedAdd = function(imgBase64) {
        let inv = currentTenant.data.inventory || [];
        let existingItem = inv.find(item => item.name.toLowerCase() === name.toLowerCase() && (!item.model || item.model.toLowerCase() === model.toLowerCase()));
        if (existingItem) {
            existingItem.qty += qty;
            existingItem.cost = cost;
            existingItem.price = price;
            if(imgBase64) existingItem.imgUrl = imgBase64;
            showCustomAlert("🔄 ዕቃው ተሞልቷል", `"${eHTML(name)}" አስቀድሞ ስለነበረ አዲሱ ብዛት ተደምሮበታል። አጠቃላይ የነበረው ብዛት፦ ${existingItem.qty}`);
        } else {
            inv.push({ name, model: model || "-", cost, price, qty, sold: 0, profit: 0, imgUrl: imgBase64 || "", unitType: "pcs" });
        }
        currentTenant.data.inventory = inv; saveAndRefresh();
        document.getElementById('itemName').value = '';
        document.getElementById('itemModel').value = '';
        document.getElementById('itemCost').value = ''; document.getElementById('itemPrice').value = ''; 
        document.getElementById('itemQty').value = ''; document.getElementById('itemImgFile').value = '';
    };
    if(file) processImageUpload(file, proceedAdd);
    else proceedAdd("");
}

function openExpenseModal() {
    showFormModal("አዲስ ወጪ መዝግብ", [
        { id: "reason", label: "የወጪ ምክንያት", type: "text", placeholder: "ምሳሌ፡ ለመብራት ክፍያ" },
        { id: "amount", label: "የገንዘብ መጠን (ETB)", type: "number", placeholder: "0.00" }
    ], (res) => {
        let amount = parseFloat(res.amount) || 0; let reason = res.reason.trim();
        if(!reason || amount <= 0) return;
        let d = currentTenant.data || {}; if(!d.expenses) d.expenses = [];
        
        d.expenses.push({ reason, amount, date: getTodayFormatted(), time: new Date().toLocaleTimeString('en-GB') });
        currentTenant.data = d; saveAndRefresh();
    });
}

function openDebtModal() {
    let inv = currentTenant.data.inventory || [];
    if (inv.length === 0) { showCustomAlert("⚠️ ዕቃ አልተገኘም", "ዕዳ ለመመዝገብ አስቀድሞ በዕቃዎች ዝርዝር ውስጥ ቢያንስ አንድ ዕቃ መኖር አለበት!"); return; }

    let itemOptions = inv.map((item, index) => { return { value: index, label: `${eHTML(item.name)} (${item.price} ETB)` }; });
    showFormModal("አዲስ የዕዳ መዝገብ", [
        { id: "customer", label: "የባለዕዳ ሙሉ ስም", type: "text", placeholder: "የሰውየው ስም..." },
        { id: "phone", label: "ስልክ ቁጥር", type: "text", placeholder: "09..." },
        { id: "itemIdx", label: "የወሰደው የዕቃ አይነት", type: "select", options: itemOptions },
        { id: "qty", label: "የዕቃው ብዛት", type: "number", placeholder: "1", defaultValue: "1" },
        { id: "date", label: "ቀን", type: "date", defaultValue: getTodayFormatted() }
    ], (res) => {
        let customer = res.customer.trim();
        let phone = res.phone.trim();
        let itemIdx = parseInt(res.itemIdx);
        let qty = parseInt(res.qty) || 0;
        let selectedDate = res.date ? res.date : getTodayFormatted();

        if (!customer || qty <= 0 || isNaN(itemIdx)) { showCustomAlert("ስህተት", "እባክዎ የተሟላና ትክክለኛ መረጃ ያስገቡ!"); return; }

        let selectedItem = inv[itemIdx];
        let calculatedAmount = selectedItem.price * qty;
        let d = currentTenant.data || {}; if (!d.debts) d.debts = [];
        d.debts.push({ customer: customer, phone: phone || "-", itemName: selectedItem.name, qty: qty, amount: calculatedAmount, paid: 0, date: selectedDate });
        selectedItem.sold += qty; 
        currentTenant.data = d; saveAndRefresh();
        sendTelegramAlert(`💳 አዲስ እዳ ተመዘገበ (${currentUserRole === 'staff' ? 'በሰራተኛ' : 'በአሰሪ'})፦\nባለእዳ፦ ${customer}\nእቃ፦ ${selectedItem.name} (${qty})\nየታሰበ ሂሳብ፦ ${calculatedAmount} ETB\nቀን፦ ${selectedDate}`);
        showCustomAlert("ተሳክቷል", `${eHTML(customer)} በዕዳ የወሰደው ሂሳብ በራሱ ተባዝቶ ገብቷል፦ ${calculatedAmount} ETB`);
    });
}

function collectDebt(idx) {
    let debt = currentTenant.data.debts[idx];
    let remaining = debt.amount - debt.paid;
    
    showFormModal(`${eHTML(debt.customer)} እዳ ክፍያ መቀበያ`, [
        { id: "amount", label: `የተከፈለው ገንዘብ (ቀሪ ዕዳ፡ ${remaining} ETB)`, type: "number", placeholder: "0.00", defaultValue: remaining }
    ], (res) => {
        let amt = parseFloat(res.amount) || 0;
        if(amt <= 0 || amt > remaining) { showCustomAlert("ስህተት", "የክፍያ መጠን ልክ አይደለም!"); return; }
    
        debt.paid += amt;
        currentTenant.data.collectedCreditToday = (parseFloat(currentTenant.data.collectedCreditToday) || 0) + amt;
        saveAndRefresh();
        sendTelegramAlert(`💵 የዕዳ ክፍያ ተሰበሰበ (${currentUserRole === 'staff' ? 'በሰራተኛ' : 'በአሰሪ'})፦\nከ ${debt.customer} ላይ ${amt} ETB ተቀብለዋል።`);
        showCustomAlert("ክፍያ ተፈጽሟል", `${eHTML(debt.customer)} እዳ ከፍሏል!`);
    });
}

function openDrawerModal() {
    showFormModal("ከሳጥን ብር ማንሻ / የተነሳ መመለሻ", [
        { id: "actionType", label: "የድርጊት ዓይነት ይምረጡ፦", type: "select", options: [
            { value: "withdraw", label: "💸 ከሳጥን ብር ማንሻ (Withdrawal)" },
            { value: "return", label: "📥 የተነሳ ብር መመለሻ (Repayment/Return)" }
        ]},
        { id: "reason", label: "ምክንያት / ማስታወሻ", type: "text", placeholder: "ምሳሌ፡ ለመልስ መለወጫ / የወሰድኩትን መለስኩ" },
        { id: "amount", label: "የገንዘብ መጠን (ETB)", type: "number", placeholder: "0.00" }
    ], (res) => {
        let amount = parseFloat(res.amount) || 0; 
        let reason = res.reason.trim();
        let action = res.actionType;
        if(!reason || amount <= 0) return;
        
        let d = currentTenant.data || {}; if(!d.drawerLog) d.drawerLog = [];
        
        let finalAmount = action === "withdraw" ? amount : -amount;
        let displayType = action === "withdraw" ? "ገንዘብ ተነሳ" : "ገንዘብ ተመለሰ";
        
        d.drawerLog.push({ reason: `${action === "withdraw" ? "⚠️ [የተነሳ] " : "✅ [የተመለሰ] "} ${reason}`, amount: finalAmount, time: new Date().toLocaleTimeString('en-GB') });
        currentTenant.data = d; saveAndRefresh();
        sendTelegramAlert(`💸 ከሳጥን ${displayType} (${currentUserRole === 'staff' ? 'በሰራተኛ' : 'በአሰሪ'})፦\nምክንያት፡ ${reason}\nመጠን፡ ${amount} ETB`);
    });
}

function openSettlementModal() {
    if(currentUserRole === "staff") return;
    showFormModal("📊 የሂሳብ ማወራረጃ ማዕከል", [
        { id: "periodType", label: "የማወራረጃ ዓይነት ይምረጡ፦", type: "select", options: [
            { value: "monthly", label: "📅 የወር ሂሳብ (Monthly)" },
            { value: "yearly", label: "📆 የአመት ሂሳብ (Yearly)" }
        ]},
        { id: "periodDate", label: "ወር / አመት ይምረጡ (ለወር: YYYY-MM, ለአመት: YYYY)፦", type: "text", placeholder: "ምሳሌ: 2026-06 ወይም 2026", defaultValue: getTodayFormatted().substring(0,7) },
        { id: "bankBalance", label: "በባንክ / ቴሌብር ላይ ያለ ጠቅላላ ገንዘብ (ETB)፦", type: "number", placeholder: "0.00" }
    ], (res) => {
        let type = res.periodType;
        let periodStr = res.periodDate.trim();
        let bankAmt = parseFloat(res.bankBalance) || 0;
        let d = currentTenant.data || {};

        let hist = d.history || [];
        let tSales = 0, tProfit = 0, tExp = 0, tDraws = 0, tReported = 0;

        let matchedEntries = hist.filter(h => {
            if(type === "monthly") return h.date.startsWith(periodStr) && !h.isMonthlyArchive;
            if(type === "yearly") return h.date.startsWith(periodStr) && !h.isMonthlyArchive;
            return false;
        });
        matchedEntries.forEach(h => {
            tSales += parseFloat(h.sales) || 0;
            tProfit += parseFloat(h.profit) || 0;
            tExp += parseFloat(h.expenses) || 0;
            tDraws += parseFloat(h.draws) || 0;
            tReported += parseFloat(h.reportedCash) || 0;
        });

        let currentStockValue = 0;
        (d.inventory || []).forEach(item => {
            let remaining = Math.max(0, item.qty - item.sold);
            currentStockValue += (item.cost * remaining);
        });
        let totalDebtRemaining = 0;
        (d.debts || []).forEach(debt => { totalDebtRemaining += (debt.amount - debt.paid); });
        let expectedBank = tSales - tExp - tDraws - totalDebtRemaining;
        if(expectedBank < 0) expectedBank = 0;
        let variance = bankAmt - expectedBank;

        let AmharicSummary = `======= 📊 ማወራረጃ (${periodStr}) =======\n• የተጣራ አጠቃላይ ሽያጭ፡ ${tSales.toFixed(2)} ETB\n• አጠቃላይ ወጪዎች፡ ${tExp.toFixed(2)} ETB\n• የተጣራ ትርፍ፡ ${tProfit.toFixed(2)} ETB\n• ከካዝና የተነሳ፡ ${tDraws.toFixed(2)} ETB\n• የተሰበሰበ ካሽ ሪፖርት፡ ${tReported.toFixed(2)} ETB\n----------------------------------------\n• በሱቅ ያለ ዕቃ ካፒታል፡ ${currentStockValue.toFixed(2)} ETB\n• ያልተሰበሰበ ቀሪ ዕዳ፡ ${totalDebtRemaining.toFixed(2)} ETB\n----------------------------------------\n• ሲስተሙ የሚጠብቀው ገንዘብ (Expected)፦ ${expectedBank.toFixed(2)} ETB\n• እርስዎ ያስገቡት የባንክ መጠን፦ ${bankAmt.toFixed(2)} ETB\n• ልዩነት (Variance)፦ ${variance.toFixed(2)} ETB\n`;
        showCustomAlert("📊 ማወራረጃ ማጠቃለያ", AmharicSummary);
        sendTelegramAlert(`📊 ሂሳብ ማወራረጃ ሪፖርት (${periodStr})፦\n${AmharicSummary}`);
    });
}

function configureBank() {
    if(currentUserRole === "staff") {
        showCustomAlert("🏦 የባንክ ሂሳብ መረጃ", `የአሰሪው የባንክ ሂሳብ ቁጥር (CBE/Telebirr)፦ ${currentTenant.bankAccount || "ያልተገናኘ"}`);
        return;
    }

    showFormModal("🏦 የባንክ እና የቴሌግራም አገናኝ መቼት", [
        { id: "telegramToken", label: "የቴሌግራም ቦት ቶከን (Telegram Bot Token)", type: "text", placeholder: "Token...", defaultValue: currentTenant.telegramToken || "" },
        { id: "telegramChatId", label: "የቴሌግራም ቻት ID (Telegram Chat ID)", type: "text", placeholder: "Chat ID...", defaultValue: currentTenant.telegramChatId || "" },
        { id: "bankAccountNumber", label: "የባንክ ሂሳብ ቁጥር (CBE/Telebirr)", type: "text", placeholder: "የባንክ ቁጥር...", defaultValue: currentTenant.bankAccount || "" }
    ], (res) => {
        currentTenant.telegramToken = res.telegramToken.trim();
        currentTenant.telegramChatId = res.telegramChatId.trim();
        currentTenant.bankAccount = res.bankAccountNumber.trim();
        saveAndRefresh();
        showCustomAlert("ተሳክቷል", "የማያያዣ መቼቶች በተሳካ ሁኔታ ተቀምጠዋል!");
    });
}

function saveStaffAccount() {
    if(currentUserRole === "staff") return;
    let u = document.getElementById('staffUser').value.trim().toLowerCase();
    let p = document.getElementById('staffPass').value.trim();
    if(!u || !p) { showCustomAlert("ስህተት", "እባክዎ የተሟላ የሰራተኛ መግቢያ ይሙሉ!"); return; }
    
    if (u === currentTenant.username) { showCustomAlert("⚠️ ስህተት", "የሰራተኛው Username ከሱቅ ባለቤቱ Username ጋር አንድ አይነት መሆን አይችልም!"); return; }
    if (localDB.tenants) {
        for (let key in localDB.tenants) {
            let otherTenant = localDB.tenants[key];
            if (otherTenant.username === u) { showCustomAlert("⚠️ ስህተት", "ይህ ዩዘርኔም አስቀድሞ በሌላ የሱቅ ባለቤት ተይዟል!"); return; }
            if (otherTenant.username !== currentTenant.username && otherTenant.staffUser && otherTenant.staffUser === u) {
                showCustomAlert("⚠️ ስህተት", "ይህ ዩዘርኔም አስቀድሞ በሌላ ሱቅ ሰራተኛ ተይዟል!");
                return;
            }
        }
    }

    currentTenant.staffUser = u;
    currentTenant.staffPass = p; saveAndRefresh();
    showCustomAlert("ተገኝቷል", "የሰራተኛ አካውንት በተሳካ ሁኔታ ተቀይሯል!");
}

function triggerShiftReport() {
    let d = currentTenant.data || {};
    let session = d.sessionData || {};
    
    let sysSales = parseFloat(d.collectedCreditToday || 0);
    let todayProfit = 0;
    let inv = d.inventory || [];
    
    inv.forEach(item => {
        sysSales += (item.price * item.sold);
        todayProfit += (item.price - item.cost) * item.sold;
    });
    showFormModal("🔒 የዕለት ሂሳብ ሪፖርት መዝጊያ ማቅረቢያ", [
        { id: "reportedCash", label: "በእጅዎ የሚገኘውን ትክክለኛ የጥሬ ገንዘብ (Cash) መጠን ያስገቡ፦", type: "number", placeholder: "0.00" }
    ], (res) => {
        let reported = parseFloat(res.reportedCash) || 0;
        let tExp = 0; let tDraw = 0;
        let formattedDateToday = getTodayFormatted();

        (d.expenses || []).forEach(e => { if (e.date === formattedDateToday) tExp += parseFloat(e.amount) || 0; });
        (d.drawerLog || []).forEach(dr => tDraw += parseFloat(dr.amount) || 0);

        let creditSalesToday = 0;
        (d.debts || []).forEach(debt => { if(debt.date === formattedDateToday) creditSalesToday += debt.amount; });

        let expectedCash = ((parseFloat(session.initialFloat) || 0) + sysSales) - creditSalesToday - tExp - tDraw;
        let variance = reported - expectedCash;
        let statusText = variance === 0 ? "ትክክል (Balanced)" : `ልዩነት አለ (${variance} ETB)`;
        d.shiftClosed = true; d.reportedCash = reported; d.variance = variance; d.expectedCash = expectedCash;

        document.getElementById('shiftStatusAlert').classList.add('hidden');
        let msg = `የዕለቱ ሂሳብ በተሳካ ሁኔታ ተዘጋጅቷል!\nሁኔታ፡ ${statusText}\nበሲስተሙ የሚጠበቅ ካሽ፡ ${expectedCash} ETB\nየቀረበው ካሽ፡ ${reported} ETB`;
        showCustomAlert("ሪፖርት ቀርቧል", msg);
        if(!d.history) d.history = [];
        d.history.push({
            date: formattedDateToday, employee: session.employee || "ሰራተኛ", sales: sysSales, 
            profit: todayProfit - tExp, expenses: tExp, draws: tDraw, reportedCash: reported,
            expectedCash: expectedCash, variance: variance, isMonthlyArchive: false
        });
        currentTenant.data = d; saveAndRefresh();
        sendTelegramAlert(`🔒 የዕለት ሂሳብ ተዘግቷል (${currentUserRole === 'staff' ? 'በሰራተኛ' : 'በአሰሪ'}):\n${msg}`);
    });
}

function startNewDaySession() {
    if(currentUserRole === "staff") return;
    let d = currentTenant.data || {};
    
    if(d.sessionActive && !d.shiftClosed) {
        showCustomAlert("ክልክል!", "መጀመሪያ የትላንቱን (ወይም የዛሬውን) የዕለት ሂሳብ 'የዕለት ሂሳብ ዝጋ' በሚለው ዘግተው ሪፖርት ማቅረብ አለብዎት!");
        return;
    }

    showCustomConfirm("አዲስ ቀን መጀመር", "የዛሬውን ቀን ሂሳብ ሙሉ በሙሉ አጽድተው ለአዲስ ቀን ማዘጋጀት ይፈልጋሉ? (የወር ትርፍዎ አይጠፋም)", () => {
        let inv = d.inventory || [];
        inv.forEach(item => {
            item.qty = Math.max(0, item.qty - item.sold); 
            item.sold = 0;
        });

        d.sessionActive = false; d.shiftClosed = false; d.drawerLog = []; d.collectedCreditToday = 0;
        currentTenant.data = d; 
        saveAndRefresh(); checkMorningSession();
        sendTelegramAlert(`🔄 አዲስ የሥራ ቀን በአሰሪ ተጀምሯል! የትላንትና ሂሳብ ተሰርዞ ወደ አዲስ ቀን ተሸጋግረዋል።`);
    });
}

function clearAllTenantData() {
    if(currentUserRole === "staff") return;
    showCustomConfirm("ሁሉንም ዳታ ማጽዳት", "ሁሉንም ዳታ ለማጥፋት እርግጠኛ ኖት?", () => {
        currentTenant.data = { sessionActive: false, shiftClosed: false, inventory: [], expenses: [], debts: [], drawerLog: [], history: [], receipts: [], deliveryOrders: [], remoteCarts: {}, lastMonthlyResetDate: new Date().getTime() };
        saveAndRefresh(); checkMorningSession();
    });
}

function renderHistoryTable() {
    let d = currentTenant.data || {};
    let historyBody = document.getElementById('historyBody');
    historyBody.innerHTML = '<tr><th>ቀን/ዓይነት</th><th>ሰራተኛ/ወቅት</th><th>ሽያጭ</th><th>ትርፍ</th><th>ሪፖርት ካሽ</th><th>ልዩነት</th></tr>';
    
    let historyList = d.history || [];
    let filterValue = document.getElementById('historyDateFilter').value;
    let filtered = historyList.filter(h => {
        if(!filterValue) return true;
        return h.date === filterValue;
    });
    if(filtered.length === 0) {
        historyBody.innerHTML += '<tr><td colspan="6" style="text-align:center; color:#94a3b8;">በተፈለገው ቀን ምንም ታሪክ የለም</td></tr>';
    } else {
        filtered.forEach(h => {
            let vColor = h.variance === 0 ? 'var(--success-color)' : 'var(--danger-color)';
            let rowStyle = h.isMonthlyArchive ? `style="background: rgba(192, 132, 252, 0.15); border-left: 4px solid var(--purple-color);"` : '';
            historyBody.innerHTML += `<tr ${rowStyle}>
                <td><b>${eHTML(h.date)}</b></td>
                <td>${eHTML(h.employee)}</td>
                <td style="color:var(--success-color)">${h.sales}</td>
                <td style="color:var(--accent-color)"><b>${h.profit}</b></td>
                <td>${h.reportedCash || 0}</td>
                <td style="${rowStyle ? '' : 'color:'+vColor}"><b>${h.variance || 0}</b></td>
            </tr>`;
        });
    }
}

window.acceptDelivery = function(idx) {
    let ord = currentTenant.data.deliveryOrders[idx];
    let item = currentTenant.data.inventory[ord.itemIdx];
    let neededMeters = item.isAdvanced && item.unitType !== 'kg' ? ord.qty * item.unitPerPack : ord.qty;
    if(item.qty - item.sold < neededMeters) { showCustomAlert("ስህተት", "ይህንን ትዕዛዝ ለማስተናገድ በቂ ክምችት የሎትም!"); return; }
    
    ord.status = "accepted"; saveAndRefresh();
    showCustomAlert("ተቀብለዋል", "ትዕዛዙ ተቀባይነት አግኝቷል! እቃው በመንገድ ላይ ነው ተብሎ ምልክት ተደርጎበታል። (ክፍያ ሲረከቡ ደረሰኝ ይቁረጡ)");
};
window.completeDelivery = function(idx) {
    let ord = currentTenant.data.deliveryOrders[idx];
    let item = currentTenant.data.inventory[ord.itemIdx];
    let neededMeters = item.isAdvanced && item.unitType !== 'kg' ? ord.qty * item.unitPerPack : ord.qty;
    
    item.sold += neededMeters;
    ord.status = "completed";
    generateDigitalReceipt(ord.itemName, ord.qty, ord.total, ord.orderId, null, true, ord.buyerUser, ord.buyerPhone);
    saveAndRefresh();
};
window.returnDelivery = function(idx) {
     let ord = currentTenant.data.deliveryOrders[idx];
     ord.status = "returned"; saveAndRefresh();
     showCustomAlert("ተመልሷል", "እቃው ተመልሷል!");
};

window.handleRemoteCartCheckout = function(buyerUser) {
    let t = currentTenant.data;
    let remoteCart = t.remoteCarts[buyerUser];
    if(!remoteCart || remoteCart.length === 0) return;
    showCustomConfirm("ክፍያ መቀበያ (Remote Checkout)", `የ ${eHTML(buyerUser)} ትዕዛዝ ክፍያ ተቀብለዋል? ደረሰኝ ይቆረጥ?`, () => {
        let grandTotal = 0;
        let receiptItems = [];
        remoteCart.forEach(c => {
            let item = t.inventory[c.itemIdx];
            let neededMeters = item.isAdvanced && item.unitType !== 'kg' ? c.qty * item.unitPerPack : c.qty;
            item.sold += neededMeters;
            grandTotal += c.total;
            receiptItems.push({ name: c.itemName, count: c.qty, unitPrice: c.price, total: c.total });
        });

        delete t.remoteCarts[buyerUser];
     
        let currentSeller = currentUserRole === 'staff' ? 'ሰራተኛ (Employee)' : 'ባለቤት (Employer)';
        let bPhone = localDB.buyers[buyerUser] ? localDB.buyers[buyerUser].phone : "";
        generateAdvancedReceipt(receiptItems, grandTotal, currentSeller, null, true, null, null, buyerUser, bPhone);
        saveAndRefresh();
        sendTelegramAlert(`🛍️ የኦንላይን ሽያጭ (Remote Cart Checkout)፦\nየገዢ ስም: ${buyerUser}\nጠቅላላ ሂሳብ፡ ${grandTotal} ETB`);
    });
};

function renderApp() {
    let d = currentTenant.data || {};
    let session = d.sessionData || {};
    if(d.sessionActive) { 
        document.getElementById('sessionDisplay').innerText = `📅 ${eHTML(session.date)} | 👤 አስገቢ፡ ${eHTML(session.employee)} | 💰 መነሻ ካዝና፡ ${session.initialFloat} ETB`; 
    }
    
    let headerRow = document.getElementById('inventoryTableHeader');
    if (currentUserRole === "staff") {
        headerRow.innerHTML = `<th>የዕቃ ስም</th><th>ሞዴል</th><th>መሸጫ ዋጋ</th><th>የተሸጠው</th><th>ቀሪ ክምችት</th><th>ድርጊት (Cart)</th>`;
    } else {
        headerRow.innerHTML = `<th>የዕቃ ስም</th><th>ሞዴል</th><th>መግዣ</th><th>መሸጫ (ች/ጅምላ)</th><th>የነበረው</th><th>የተሸጠው</th><th>ቀሪ</th><th>ትርፍ</th><th>እርምጃ</th>`;
    }

    let tbody = document.getElementById('inventoryBody'); tbody.innerHTML = '';
    let collectedCredit = parseFloat(d.collectedCreditToday) || 0;
    let tSales = collectedCredit; let todayProfit = 0; let tExp = 0; let tDraw = 0;
    let currentTotalCapital = 0;
    
    let expensesList = d.expenses || []; expensesList.forEach(e => tExp += parseFloat(e.amount) || 0);
    let drawsList = d.drawerLog || []; drawsList.forEach(dr => tDraw += parseFloat(dr.amount) || 0);

    let query = document.getElementById('inventorySearchInput') ? document.getElementById('inventorySearchInput').value.trim().toLowerCase() : "";

    let inv = d.inventory || [];
    inv.forEach((item, idx) => {
        let remaining = Math.max(0, item.qty - item.sold); 
        let profit = (item.price - item.cost) * item.sold; 
        tSales += (item.price * item.sold); 
        todayProfit += profit; 
        currentTotalCapital += (item.cost * remaining);

        if (query !== "" && !item.name.toLowerCase().includes(query)) return;

        let rowClass = remaining <= 3 ? 'low-stock-row' : '';
        let stockBadge = remaining <= 3 ? '<span class="low-stock-badge">⚠️</span>' : '';
        let itemModelText = eHTML(item.model) || "-";
       
        let wholesaleText = item.wholesalePrice ? ` / ${item.wholesalePrice}` : '';
        let priceDisplay = `${item.price}${wholesaleText}`;
        
        let sellAction = `
            <div style="display:flex; gap:5px; align-items:center;">
                <input type="number" id="quickQty_${idx}" style="width:60px; padding:4px; margin:0;" placeholder="ብዛት" value="1">
                <select id="quickType_${idx}" style="width:70px; padding:4px; margin:0; ${item.wholesalePrice > 0 ? '' : 'display:none;'}">
                    <option value="retail">ችርቻሮ</option>
                    <option value="wholesale">ጅምላ</option>
                </select>
                <button class="btn-sell btn-sm" onclick="addToMainCart(${idx})">➕ ሽጥ</button>
                ${currentUserRole === "owner" ? `<button class="btn-expense btn-sm" onclick="deleteInventoryItem(${idx})" style="margin-left:5px;">🗑️</button>` : ''}
            </div>
        `;

        let displayQty = item.qty; let displaySold = item.sold; let displayRem = remaining;

        if(item.isAdvanced || item.unitType === 'kg') {
            let uLabel = item.unitType === 'kg' ? ' ኪሎ' : ' ሜትር';
            displayQty = `${item.qty}${uLabel}`; displaySold = `${item.sold}${uLabel}`; displayRem = `${remaining}${uLabel}`;
        }

        if (currentUserRole === "staff") {
            tbody.innerHTML += `<tr class="${rowClass}">
                <td><strong>${eHTML(item.name)}</strong> ${stockBadge}</td>
                <td>${itemModelText}</td>
                <td>${item.price} ETB</td>
                <td><b>${displaySold}</b></td>
                <td style="${remaining <= 3 ? 'color:#f87171;font-weight:bold;' : ''}">${displayRem}</td>
                <td>${sellAction}</td>
            </tr>`;
        } else {
            tbody.innerHTML += `<tr class="${rowClass}">
                <td><strong>${eHTML(item.name)}</strong> ${stockBadge}</td>
                <td>${itemModelText}</td>
                <td>${item.cost}</td>
                <td>${priceDisplay}</td>
                <td>${displayQty}</td>
                <td><b>${displaySold}</b></td>
                <td>${displayRem}</td>
                <td>${profit}</td>
                <td>${sellAction}</td>
            </tr>`;
        }
    });

    let formattedDateToday = getTodayFormatted();
    let todayExpensesTotal = 0; let creditSalesToday = 0;
    
    expensesList.forEach(e => { if (e.date === formattedDateToday) todayExpensesTotal += parseFloat(e.amount) || 0; });
    (d.debts || []).forEach(debt => { if (debt.date === formattedDateToday) creditSalesToday += debt.amount; });
    
    let finalCashInHand = ((parseFloat(session.initialFloat) || 0) + tSales) - creditSalesToday - todayExpensesTotal - tDraw;
    if (d.shiftClosed) {
        todayProfit = 0;
        finalCashInHand = 0;
    }

    document.getElementById('totalInCash').innerText = finalCashInHand.toFixed(1) + " ETB";
    
    let totalBuyersCount = localDB.buyers ? Object.keys(localDB.buyers).length : 0;
    let sellerTotalBuyersEl = document.getElementById('sellerTotalBuyers');
    if(sellerTotalBuyersEl) sellerTotalBuyersEl.innerText = totalBuyersCount;
    if (currentUserRole === "owner") {
        let monthlyProfit = todayProfit - todayExpensesTotal;
        let historyList = d.history || []; 
        historyList.forEach(h => {
            if(!h.isMonthlyArchive) monthlyProfit += parseFloat(h.profit) || 0;
        });
        document.getElementById('totalCapital').innerText = currentTotalCapital.toFixed(1) + " ETB";
        document.getElementById('todayNetProfit').innerText = (todayProfit - todayExpensesTotal).toFixed(1) + " ETB";
        document.getElementById('monthlyNetProfit').innerText = monthlyProfit.toFixed(1) + " ETB";
        document.getElementById('monthlyExpenses').innerText = tExp.toFixed(1) + " ETB";
        document.getElementById('totalDraws').innerText = tDraw.toFixed(1) + " ETB";
        if (myChart) {
            myChart.data.datasets[0].data = [currentTotalCapital, tSales, todayProfit - todayExpensesTotal];
            myChart.update();
        }
        renderHistoryTable();
    }

    let remoteBody = document.getElementById('sellerRemoteCartsBody');
    if(remoteBody) {
        remoteBody.innerHTML = "";
        let remoteCarts = d.remoteCarts || {};
        let hasRemotes = false;
        Object.keys(remoteCarts).forEach(bUser => {
            let items = remoteCarts[bUser];
            if(items && items.length > 0) {
                hasRemotes = true;
                let totalSum = 0;
         
                let detailsHTML = "";
                items.forEach(i => {
                    totalSum += i.total;
                    let invItem = d.inventory[i.itemIdx];
           
                    let modelTxt = (invItem && invItem.model && invItem.model !== "-") ? `(ሞዴል: ${eHTML(invItem.model)})` : "";
                    detailsHTML += `<div style="font-size:0.8rem; margin-bottom:2px; color: var(--accent-color);">▪ ${eHTML(i.itemName)} ${modelTxt} - ብዛት: ${i.qty}</div>`;
                });
                
                remoteBody.innerHTML += `<tr>
                    <td>👤 ${eHTML(bUser)}</td>
                    <td>${detailsHTML}</td>
                    <td><b style="color:var(--success-color)">${totalSum} ETB</b></td>
                    <td><button class="btn-sell btn-sm" onclick="handleRemoteCartCheckout('${eHTML(bUser).replace(/'/g, "\\'")}')">✅ ክፍያ ተቀበል (Checkout)</button></td>
                </tr>`;
            }
        });
        if(!hasRemotes) remoteBody.innerHTML = `<tr><td colspan="4" style="text-align:center; color:#94a3b8;">በአሁኑ ሰዓት የገዥዎች Cart ትዕዛዝ የለም።</td></tr>`;
    }

    let delBody = document.getElementById('sellerDeliveryBody');
    if(delBody) {
        delBody.innerHTML = "";
        let orders = d.deliveryOrders || [];
        let hasDel = false;
        orders.forEach((ord, idx) => {
            if(ord.status === "completed" || ord.status === "returned") return;
            hasDel = true;
            let statusBadge = ord.status === "pending" ? `<span class="badge-warning">በመጠባበቅ ላይ</span>` : `<span class="badge-success">በመንገድ ላይ</span>`;
            
            let actions = "";
            if(ord.status === "pending") {
                actions = `<button class="btn-sell btn-sm" onclick="acceptDelivery(${idx})">ተቀበል (Accept)</button>`;
            } else if(ord.status === "accepted") {
                actions = `
                    <button class="btn-sell btn-sm" onclick="completeDelivery(${idx})">ተረክቦ ደረሰኝ ቆርጥ</button>
                    <button class="btn-expense btn-sm" style="margin-top:4px;" onclick="returnDelivery(${idx})">እቃው ተመለሰ</button>
                `;
            }
            
            let invItem = d.inventory[ord.itemIdx];
            let modelTxt = (invItem && invItem.model && invItem.model !== "-") ? `(ሞዴል: ${eHTML(invItem.model)})` : "";

            delBody.innerHTML += `<tr>
                <td>👤 ${eHTML(ord.buyerUser)}<br>📞 ${eHTML(ord.buyerPhone)}</td>
                <td>📍 ${eHTML(ord.address)} <br> ${ord.mapLink ? `<a href="${eHTML(ord.mapLink)}" target="_blank" style="color:var(--accent-color);">Map Link</a>` : ''}</td>
                <td>📦 <b style="color:var(--accent-color);">${eHTML(ord.itemName)}</b> <br> ${modelTxt} <br> ብዛት: ${ord.qty}</td>
                <td>${ord.total} ETB</td>
                <td>${statusBadge}</td>
                <td>${actions}</td>
            </tr>`;
        });
        if(!hasDel) delBody.innerHTML = `<tr><td colspan="6" style="text-align:center; color:#94a3b8;">በአሁኑ ሰዓት ምንም አዲስ የዴሊቨሪ ትዕዛዝ የለም።</td></tr>`;
    }

    let creditBody = document.getElementById('creditBody');
    creditBody.innerHTML = '<tr><th>ባለዕዳ / ስልክ</th><th>የወሰደው ዕቃ (ብዛት)</th><th>ጠቅላላ ዕዳ</th><th>ቀሪ</th><th>ድርጊት</th></tr>';
    let debts = d.debts || [];
    if(debts.length === 0) {
        creditBody.innerHTML += '<tr><td colspan="5" style="text-align:center; color:#94a3b8;">ምንም የዕዳ መዝገብ የለም</td></tr>';
    } else {
        debts.forEach((debt, idx) => {
            let remaining = debt.amount - debt.paid;
            if (remaining > 0) {
                let itemDisplay = debt.itemName ? `${eHTML(debt.itemName)} (${debt.qty || 1} ፍሬ)` : "-";
                creditBody.innerHTML += `<tr>
                    <td><b>${eHTML(debt.customer)}</b><br><small style="color:#94a3b8">${eHTML(debt.phone)}</small><br><small style="color:var(--warning-color)">📅 ${eHTML(debt.date || '')}</small></td>
                    <td>${itemDisplay}</td>
                    <td>${debt.amount} ETB</td>
                    <td style="color:var(--danger-color)"><b>${remaining} ETB</b></td>
                    <td><button class="btn-sell btn-sm" onclick="collectDebt(${idx})">ክፍያ</button></td>
                </tr>`;
            }
        });
    }

    let drawBody = document.getElementById('drawBody');
    drawBody.innerHTML = '<tr><th>ምክንያት</th><th>የተወሰደው</th><th>ሰዓት</th></tr>';
    if(drawsList.length === 0) {
        drawBody.innerHTML += '<tr><td colspan="3" style="text-align:center; color:#94a3b8;">ምንም የተነሳ ገንዘብ የለም</td></tr>';
    } else {
        drawsList.forEach(dr => {
            let isReturn = dr.amount < 0;
            let displayAmt = isReturn ? Math.abs(dr.amount) + " ETB (መለሰ)" : dr.amount + " ETB";
            let displayColor = isReturn ? "var(--success-color)" : "var(--purple-color)";
            let tbodyColor = `style="color:${displayColor}; font-weight:bold;"`;
            drawBody.innerHTML += `<tr><td>${eHTML(dr.reason)}</td><td ${tbodyColor}>${displayAmt}</td><td>${dr.time}</td></tr>`;
        });
    }

    let receiptHistoryBody = document.getElementById('receiptHistoryTableBody');
    receiptHistoryBody.innerHTML = '';
    let pastReceipts = d.receipts || [];
    let receiptFilterDate = document.getElementById('receiptDateFilter').value;

    if (!receiptFilterDate) {
        receiptHistoryBody.innerHTML = '<tr><td colspan="7" style="text-align:center; color:#94a3b8; font-weight: bold;">📅 እባክዎ ደረሰኞችን ለማየት መጀመሪያ ቀን ይምረጡ!</td></tr>';
    } else {
        let filteredReceipts = pastReceipts.filter(rec => rec.date === receiptFilterDate);
        if (filteredReceipts.length === 0) {
            receiptHistoryBody.innerHTML = `<tr><td colspan="7" style="text-align:center; color:#94a3b8;">የመረጡት ቀን (${receiptFilterDate}) የተቆረጠ ምንም ደረሰኝ የለም።</td></tr>`;
        } else {
            let reversedReceipts = [...pastReceipts].reverse();
            reversedReceipts.forEach((rec, originalIdx) => {
                let actualIdx = pastReceipts.length - 1 - originalIdx;
                if (rec.date === receiptFilterDate) {
                    receiptHistoryBody.innerHTML += 
                    `<tr>
                        <td><b>#${rec.recId}</b></td>
                        <td>${rec.date}</td>
                        <td>${eHTML(rec.itemName)}</td>
                        <td>${rec.count}</td>
                        <td class="text-success"><b>${rec.totalVal} ETB</b></td>
                        <td><span class="text-warning">${eHTML(rec.seller)}</span></td>
                        <td><button class="btn-config btn-sm" onclick="viewPastReceipt(${actualIdx})">👁️ ድጋሚ እይ / Print</button></td>
                    </tr>`;
                }
            });
        }
    }
    renderMainCart();
    checkTimeLock();
}

function initChart() {
    let canvasElement = document.getElementById('businessChart');
    if (!canvasElement || currentUserRole === "staff") return;
    let ctx = canvasElement.getContext('2d');
    if (myChart) myChart.destroy();
    myChart = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: ['ካፒታል', 'የዛሬ ሽያጭ', 'የዛሬ ትርፍ'],
            datasets: [{ label: 'የገንዘብ መጠን (ETB)', data: [0, 0, 0], backgroundColor: ['#38bdf8', '#4ade80', '#fbbf24'], borderRadius: 6 }]
        },
        options: {
            responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } },
            scales: { y: { grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#94a3b8' } }, x: { ticks: { color: '#94a3b8' } } }
        }
    });
}

function checkMorningSession() {
    let d = currentTenant.data || {};
    if (!d.sessionActive) {
        showFormModal("የቀኑ መጀመሪያ መመዝገቢያ (የካዝና ማስሞያ)", [
            { id: "employee", label: "የገቢ አድራጊው/ሰራተኛው ስም ያስገቡ፦", type: "text", placeholder: "ስም", defaultValue: currentUserRole === "staff" ? "ሰራተኛ" : "አሰሪ" },
            { id: "initialFloat", label: "ጠዋት በካዝና/ሳጥን ውስጥ የተገኘ መነሻ ገንዘብ (Float)፦", type: "number", placeholder: "0.00", defaultValue: "0" }
        ], (res) => {
            d.sessionData = { date: getTodayFormatted(), loginTime: new Date().toLocaleTimeString('en-GB'), employee: res.employee || "ሰራተኛ", initialFloat: parseFloat(res.initialFloat) || 0 };
            d.sessionActive = true; d.shiftClosed = false; d.expenses = d.expenses || []; 
            d.drawerLog = []; d.debts = d.debts || []; d.receipts = d.receipts || [];
            d.deliveryOrders = d.deliveryOrders || []; d.collectedCreditToday = 0;
            currentTenant.data = d; 
            document.getElementById('receiptDateFilter').value = getTodayFormatted();
            saveAndRefresh();
        });
    } else {
        renderApp();
    }
}

// --- SECURITY FIX: Firebase Race Condition Protection ---
function pushToFirebase() { 
    if(!isOnline) return;
    if (currentUserRole === 'admin') {
        db.ref('tirfe_system').update(localDB);
    } else if (currentTenant) {
        db.ref(`tirfe_system/tenants/${currentTenant.username}`).update(currentTenant);
    } else if (currentBuyer) {
        db.ref(`tirfe_system/buyers/${currentBuyer.username}`).update(currentBuyer);
    }
}

function openModalContainer() { document.getElementById('modalOverlay').classList.remove('hidden'); }
function closeActiveModal() { document.getElementById('modalOverlay').classList.add('hidden');
document.querySelectorAll('.modal-card').forEach(m => m.classList.add('hidden')); }

function showCustomAlert(title, message, callback) {
    document.getElementById('alertTitle').innerText = eHTML(title);
    document.getElementById('alertMessage').innerText = eHTML(message);
    openModalContainer(); document.getElementById('alertModal').classList.remove('hidden');
    document.querySelector('#alertModal .btn-add').onclick = function() { closeActiveModal(); if(callback) callback(); };
}

function showFormModal(title, fields, onSubmit) {
    document.getElementById('formModalTitle').innerText = title;
    let body = document.getElementById('formModalBody'); body.innerHTML = '';
    
    fields.forEach(f => {
        let label = document.createElement('label'); label.innerText = f.label;
        label.style.fontSize = '0.85rem'; label.style.color = 'var(--accent-color)'; label.style.display = 'block'; label.style.marginTop = '10px';
        
        let input;
        if(f.type === 'select') {
            input = document.createElement('select');
            f.options.forEach(opt => {
                let o = document.createElement('option');
                if (typeof opt === 'object' && opt !== null) { o.value = opt.value; o.innerText = opt.label; }
                input.appendChild(o);
            });
        } else if (f.type === 'file') {
            input = document.createElement('input');
            input.type = 'file';
            input.accept = 'image/*';
        } else {
            input = document.createElement('input');
            input.type = f.type || 'text'; input.placeholder = f.placeholder || '';
            if(f.defaultValue !== undefined) input.value = f.defaultValue;
        }
        input.id = 'formField_' + f.id;
        body.appendChild(label); body.appendChild(input);
    });
    
    let footer = document.getElementById('formModalFooter'); footer.innerHTML = '';
    let cancelBtn = document.createElement('button'); cancelBtn.className = 'btn-config'; cancelBtn.innerText = 'ሰርዝ';
    cancelBtn.onclick = closeActiveModal;
    let submitBtn = document.createElement('button'); submitBtn.className = 'btn-sell'; submitBtn.innerText = 'አስገባ';
    submitBtn.onclick = function() {
        let data = {};
        let fileInputObj = null;
        fields.forEach(f => { 
            if (f.type === 'file') { fileInputObj = document.getElementById('formField_' + f.id); }
            else { data[f.id] = document.getElementById('formField_' + f.id).value; }
        });
        closeActiveModal(); onSubmit(data, fileInputObj);
    };
    footer.appendChild(cancelBtn); footer.appendChild(submitBtn);
    openModalContainer(); document.getElementById('formModal').classList.remove('hidden');
}

function showCustomConfirm(title, message, onConfirm) {
    document.getElementById('confirmTitle').innerText = eHTML(title);
    document.getElementById('confirmMessage').innerText = eHTML(message);
    openModalContainer(); document.getElementById('confirmModal').classList.remove('hidden');
    document.getElementById('confirmYesBtn').onclick = function() { closeActiveModal(); if(onConfirm) onConfirm(); };
}

function changeTheme(themeClass) { document.body.className = themeClass; if(currentTenant) { currentTenant.theme = themeClass; saveAndRefresh(); } }

function deleteInventoryItem(idx) { 
    if(currentUserRole === "staff") return;
    showCustomConfirm("እቃ መሰረዣ", "ይህንን እቃ ማጥፋት ይፈልጋሉ?", () => { currentTenant.data.inventory.splice(idx, 1); saveAndRefresh(); });
}

function downloadReceiptPDF(filename) {
    const element = document.getElementById('printableReceiptArea');
    const opt = { 
        margin: [5,5,5,5], filename: filename + '.pdf', image: { type: 'jpeg', quality: 1.0 }, 
        html2canvas: { scale: 3, useCORS: true, letterRendering: true, backgroundColor: "#ffffff" }, 
        jsPDF: { unit: 'mm', format: [80, 150], orientation: 'portrait' } 
    };
    html2pdf().set(opt).from(element).save();
}

function shareToSocial(platform, text) {
    let url = "";
    if(platform === 'tg') { url = `tg://msg?text=${encodeURIComponent(text)}`; }
    else if(platform === 'wa') { url = `https://api.whatsapp.com/send?text=${encodeURIComponent(text)}`; }
    window.open(url, '_blank');
}

function openItemRegistrationChoice() {
    showFormModal("የዕቃ ምዝገባ አማራጭ", [
        { id: "regType", label: "እባክዎ የሚመዘግቡትን የዕቃ ልኬት አይነት ይምረጡ፦", type: "select", options: [
            {value: "standard", label: "📦 መደበኛ (በፍሬ / ልኬት የሌለው)"},
            {value: "advanced", label: "📏/⚖️ በጥቅል/ሜትር ወይም በኪሎግራም"}
        ]}
    ], (res) => {
        if(res.regType === "standard") {
            document.getElementById('itemName').focus(); showCustomAlert("መረጃ", "መደበኛ ዕቃዎችን ከታች ባለው 'የዕቃ ስም' በሚለው ፎርም ቀጥታ መመዝገብ ይችላሉ።");
        } else if(res.regType === "advanced") { openAdvancedRegistration(); }
    });
}

function openAdvancedRegistration() {
    showFormModal(`📏/⚖️ በጥቅል/ሜትር ወይም በኪሎግራም የሚለካ ዕቃ መዝግብ`, [
        { id: "unitType", label: "የልኬት አይነት ይምረጡ", type: "select", options: [{value: "meter", label: "📏 በሜትር (Meter)"}, {value: "kg", label: "⚖️ በኪሎግራም (KG)"}] },
        { id: "name", label: "የዕቃ ስም (ምሳሌ፡ የኤሌክትሪክ ገመድ/ስኳር)", type: "text", placeholder: `ስም` },
        { id: "model", label: "ሞዴል / አይነት", type: "text", placeholder: "-" },
        { id: "packCount", label: "ስንት ጥቅል (Package/Roll/Sack) ገባ?", type: "number", placeholder: "0" },
        { id: "unitPerPack", label: "በአንድ ጥቅል ውስጥ ያለው ጠቅላላ ሜትር/ኪሎ", type: "number", placeholder: "0" },
        { id: "totalCost", label: `የጠቅላላ ዕቃው የገባበት ዋጋ (ካፒታል)`, type: "number", placeholder: "0" },
        { id: "retailPrice", label: "የ 1 ሜትር/ኪሎ መሸጫ ዋጋ (ችርቻሮ)", type: "number", placeholder: "0" },
        { id: "wholesalePrice", label: "በጅምላ (በጥቅል/ጆንያ) ሲሸጥ የአንድ ጥቅል መሸጫ ዋጋ", type: "number", placeholder: "0" },
        { id: "advImgFile", label: "የዕቃው ፎቶ ከጋላሪ ይምረጡ (አማራጭ)፡", type: "file" }
    ], (res, fileInputObj) => {
        let name = res.name.trim();
        let packCount = parseFloat(res.packCount) || 0; let unitPerPack = parseFloat(res.unitPerPack) || 0;
        let totalQtyInMeters = packCount * unitPerPack;
        let totalCost = parseFloat(res.totalCost) || 0; let retailPrice = parseFloat(res.retailPrice) || 0;
        if(!name || packCount <= 0 || unitPerPack <= 0 || totalCost <= 0 || retailPrice <= 0) { showCustomAlert("ስህተት", "እባክዎ የተሟላ እና ትክክለኛ መረጃ ያስገቡ!"); return; }

        let proceedAdd = function(imgBase64) {
            let inv = currentTenant.data.inventory || [];
            let existingItem = inv.find(item => item.name.toLowerCase() === name.toLowerCase() && (!item.model || item.model.toLowerCase() === (res.model || "-").toLowerCase()));
            let unitCostPerMeter = totalCost / totalQtyInMeters;

            if (existingItem) {
                existingItem.qty += totalQtyInMeters;
                existingItem.cost = unitCostPerMeter; existingItem.price = retailPrice; existingItem.wholesalePrice = parseFloat(res.wholesalePrice) || 0; existingItem.unitPerPack = unitPerPack;
                if(imgBase64) existingItem.imgUrl = imgBase64;
                showCustomAlert("🔄 ዕቃው ተሞልቷል", `"${eHTML(name)}" አስቀድሞ ስለነበረ አዲሱ ብዛት ተደምሮበታል። አጠቃላይ የነበረው፦ ${existingItem.qty}`);
            } else {
                inv.push({ name: name, model: res.model || "-", cost: unitCostPerMeter, price: retailPrice, qty: totalQtyInMeters, sold: 0, profit: 0, imgUrl: imgBase64 || "", wholesalePrice: parseFloat(res.wholesalePrice) || 0, isAdvanced: true, unitType: res.unitType, unitPerPack: unitPerPack });
                showCustomAlert("ተሳክቷል", `ዕቃው በተሳካ ሁኔታ ተመዝግቧል! አጠቃላይ ብዛት: ${totalQtyInMeters} ${res.unitType === 'kg' ? 'ኪሎ' : 'ሜትር'}`);
            }
            currentTenant.data.inventory = inv; saveAndRefresh();
        };

        if(fileInputObj && fileInputObj.files[0]) {
            processImageUpload(fileInputObj.files[0], proceedAdd);
        } else {
            proceedAdd("");
        }
    });
}

function openSellChoiceModal() {
    document.getElementById('inventorySearchInput').focus();
    showCustomAlert("መረጃ", "እባክዎ ከታች ካለው የዕቃዎች ዝርዝር (ቴብል) ላይ '➕ ሽጥ' የሚለውን በመጫን ወደ ቅርጫት (Cart) ያስገቡ እና ክፍያ ይፈፅሙ።");
}

window.addToMainCart = function(idx) {
    if(currentTenant.data.shiftClosed) { showCustomAlert("ስህተት", "የዕለቱ ፈረቃ ተዘግቷል! ማሸጥ አይቻልም።"); return; }
    
    let qtyInput = document.getElementById(`quickQty_${idx}`);
    let qty = parseFloat(qtyInput.value) || 0;
    let typeSelect = document.getElementById(`quickType_${idx}`); let isWholesale = typeSelect && typeSelect.value === 'wholesale';
    let item = currentTenant.data.inventory[idx]; let rem = item.qty - item.sold;
    
    if(qty <= 0) { showCustomAlert("ስህተት", "የተሳሳተ ብዛት ነው!"); return; }

    let unitPriceToUse = (isWholesale && item.wholesalePrice > 0) ? item.wholesalePrice : item.price;
    let neededMeters = qty; if(isWholesale && item.isAdvanced) { neededMeters = qty * item.unitPerPack; }

    if(neededMeters > rem) { showCustomAlert("ስህተት", "ከክምችት በላይ ነው!"); return; }

    let existIdx = mainCart.findIndex(c => c.index === idx && c.isWholesale === isWholesale);
    if(existIdx > -1) {
        let totalNeeded = mainCart[existIdx].deductedMeters + neededMeters;
        if(totalNeeded > rem) { showCustomAlert("ስህተት", "ከክምችት በላይ ነው!"); return; }
        mainCart[existIdx].qty += qty;
        mainCart[existIdx].deductedMeters += neededMeters; mainCart[existIdx].total = mainCart[existIdx].qty * unitPriceToUse;
    } else {
        let nName = item.name + (isWholesale ? (item.isAdvanced ? " (በጥቅል)" : " (በጅምላ)") : "");
        mainCart.push({ index: idx, name: nName, qty: qty, deductedMeters: neededMeters, price: unitPriceToUse, total: qty * unitPriceToUse, isWholesale: isWholesale });
    }
    
    qtyInput.value = '1'; renderMainCart();
};

window.renderMainCart = function() {
    let container = document.getElementById('cartItemsList'); let totalEl = document.getElementById('cartTotalSum');
    let emptyMsg = document.getElementById('emptyCartMsg');
    
    if(!mainCart || mainCart.length === 0) {
        container.innerHTML = "";
        emptyMsg.style.display = "block"; totalEl.innerText = "0"; return;
    }
    
    emptyMsg.style.display = "none";
    let html = '<table style="width:100%; border-collapse:collapse; margin-bottom:10px;">';
    let grandTotal = 0;
    mainCart.forEach((c, i) => {
        grandTotal += c.total;
        html += `<tr style="border-bottom:1px solid rgba(255,255,255,0.1);">
            <td style="padding:8px 0; color:var(--text-color);">${eHTML(c.name)}</td>
            <td style="color:var(--text-color);">${c.qty}</td>
            <td style="color:var(--text-color);">${c.price} ETB</td>
        
            <td style="color:var(--success-color);"><b>${c.total} ETB</b></td>
            <td style="text-align:right;"><button class="btn-expense btn-sm" onclick="removeMainCartItem(${i})">❌</button></td>
        </tr>`;
    });
    html += '</table>'; container.innerHTML = html; totalEl.innerText = grandTotal;
};

window.removeMainCartItem = function(i) { mainCart.splice(i, 1); renderMainCart(); };
window.checkoutMainCart = function() {
    if(!mainCart || mainCart.length === 0) { showCustomAlert("ስህተት", "እባክዎ መጀመሪያ ከቴብሉ እቃ ወደ ቅርጫቱ ያስገቡ!"); return; }
    
    let grandTotal = 0;
    let currentSeller = currentUserRole === 'staff' ? 'ሰራተኛ (Employee)' : 'ባለቤት (Employer)';
    let receiptItems = [];
    mainCart.forEach(c => {
        let item = currentTenant.data.inventory[c.index];
        item.sold += c.deductedMeters; 
        grandTotal += c.total;
        receiptItems.push({ name: c.name, count: c.qty, unitPrice: c.price, total: c.total });
    });
    sendTelegramAlert(`🛍️ የሽያጭ ማስታወቂያ (${currentSeller})፦\nየሱቅ ስም: ${currentTenant.shopName}\nየተሸጡ ዕቃዎች፡ ${receiptItems.length} አይነት\nጠቅላላ ሂሳብ፡ ${grandTotal} ETB`);

    mainCart = []; saveAndRefresh(); renderMainCart(); generateAdvancedReceipt(receiptItems, grandTotal, currentSeller);
};

function generateAdvancedReceipt(itemsArray, grandTotal, currentSeller, recId = null, saveToHistory = true, givenShopName = null, givenBType = null, buyerName = null, buyerPhone = null) {
    if (!recId) recId = Math.floor(10000 + Math.random() * 90000);
    let dateStr = getTodayFormatted();
    let shopName = givenShopName || currentTenant.shopName;
    let bType = givenBType || currentTenant.businessType || 'አጠቃላይ ንግድ';
    let ownerName = currentTenant.fullName || "ያልተመዘገበ";
    let ownerPhone = currentTenant.phone || "ያልተመዘገበ";
    let shopLogo = currentTenant.shopLogo || "https://cdn-icons-png.flaticon.com/512/869/869636.png";
    let displayBuyerName = buyerName;
    let displayBuyerPhone = buyerPhone;
    
    if (buyerName && localDB.buyers && localDB.buyers[buyerName] && !buyerPhone) {
        displayBuyerPhone = localDB.buyers[buyerName].phone;
    } else if (currentBuyer && !buyerName) {
        displayBuyerName = currentBuyer.username;
        displayBuyerPhone = currentBuyer.phone;
    }

    let rawTextForShare = `======= ${shopName.toUpperCase()} =======\nየንግድ ዘርፍ: ${bType}\nደረሰኝ ቁጥር: #${recId}\nየሸጠው ሰው: ${currentSeller}\nቀን: ${dateStr}\n---------------------------\n`;
    let tableRows = "";
    itemsArray.forEach(itm => {
        rawTextForShare += `ዕቃ: ${itm.name} | ብዛት: ${itm.count} | ዋጋ: ${itm.total} ETB\n`;
        tableRows += `<tr><td style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;"><b>${eHTML(itm.name)}</b></td><td style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;">${itm.count}</td><td style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;">${itm.unitPrice.toFixed(1)}</td><td style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;"><b>${itm.total} ETB</b></td></tr>`;
    });
    rawTextForShare += `---------------------------\nጠቅላላ ሂሳብ: ${grandTotal} ETB\n`;
    if (displayBuyerName) {
         rawTextForShare += `ገዥ: ${displayBuyerName} | ስልክ: ${displayBuyerPhone || ''}\n`;
    }
    rawTextForShare += `እናመሰግናለን!`;
    if (saveToHistory) {
        if(!currentTenant.data.receipts) currentTenant.data.receipts = [];
        let mainName = itemsArray.length === 1 ? itemsArray[0].name : "የተለያዩ ዕቃዎች (" + itemsArray.length + ")";
        let mainCount = itemsArray.length === 1 ? itemsArray[0].count : "-";
        let recObj = { recId: recId, date: dateStr, itemName: mainName, count: mainCount, totalVal: grandTotal, seller: currentSeller, advancedItems: itemsArray, shopName: shopName, bType: bType, buyerName: displayBuyerName, buyerPhone: displayBuyerPhone };
        currentTenant.data.receipts.push(recObj);
        
        if(displayBuyerName && localDB.buyers && localDB.buyers[displayBuyerName]) {
            if(!localDB.buyers[displayBuyerName].receipts) localDB.buyers[displayBuyerName].receipts = [];
            localDB.buyers[displayBuyerName].receipts.push(recObj);
        }
        saveAndRefresh();
    }

    let buyerSection = "";
    if (displayBuyerName) {
        buyerSection = `<div style="margin-top: 15px; border-top: 2px dashed #333; padding-top: 10px; text-align: left; font-size: 0.9rem;">
                            <b>ገዥ:</b> ${eHTML(displayBuyerName)} <br>
                            <b>ስልክ ቁጥር:</b> ${eHTML(displayBuyerPhone || '')}
                        </div>`;
    }

    let receiptHTML = `
    <div class="receipt-container" id="printableReceiptArea" style="background:#fff; color:#000; padding:15px; width:100%; max-width:350px; margin:0 auto;">
        <div class="receipt-header" style="display:flex; flex-direction:column; align-items:center;">
            <img src="${shopLogo}" style="width:60px; height:60px; border-radius:50%; margin-bottom:10px; object-fit:cover; border: 1px solid #ddd;" onerror="this.src='https://cdn-icons-png.flaticon.com/512/869/869636.png'">
            <h4 style="margin:0; font-size:1.3rem; color:#111; text-transform:uppercase;">${eHTML(shopName)}</h4>
            <p style="color:#565656; font-weight:bold; margin: 4px 0;">[ ${eHTML(bType)} ]</p>
            <p style="margin: 2px 0; font-size: 0.85rem;"><b>የባለቤት ስም:</b> ${eHTML(ownerName)}</p>
            <p style="margin: 2px 0; font-size: 0.85rem;"><b>ስልክ:</b> ${eHTML(ownerPhone)}</p>
        
            <div style="border-bottom: 2px dashed #333; width: 100%; margin: 10px 0;"></div>
            <p style="margin: 2px 0; font-size: 0.85rem; font-weight:bold;">ዲጂታል የሽያጭ ደረሰኝ</p>
            <p style="margin: 2px 0; font-size: 0.85rem;"><b>ቁጥር (No):</b> #${recId} | ቀን: ${dateStr}</p>
            <p style="margin: 2px 0; font-size: 0.85rem;"><b>የሻጭ ማንነት:</b> ${eHTML(currentSeller)}</p>
        </div>
        <table class="receipt-table" style="color:#000; width:100%; margin-top: 10px; border-collapse: collapse;">
            <thead><tr><th style="color:#000!important; text-align:left; border-bottom: 1px dashed #ddd; padding: 5px;">የዕቃ ስም</th><th style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;">ብዛት</th><th style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;">ነጠላ</th><th style="color:#000!important; border-bottom: 1px dashed #ddd; padding: 5px;">ጠቅላ局</th></tr></thead>
            <tbody>${tableRows}</tbody>
        </table>
        <div class="receipt-summary" style="margin-top: 15px; border-top: 2px dashed #333; padding-top: 8px; text-align: right; font-size: 0.95rem; font-weight: bold; color: #111;">
            የተከፈለ ጠቅላላ ሂሳብ: ${grandTotal.toFixed(2)} ETB
        </div>
      
        ${buyerSection}
        <div class="receipt-footer" style="text-align: center; margin-top: 20px; font-size: 0.8rem; color: #777; font-style: italic;">~ ስለመጡ እናመሰግናለን! እንደገና ይጎብኙን ~</div>
    </div>
    <div class="receipt-actions-grid">
        <button class="btn-sell" onclick="window.print()">𖖨️ ደረሰኝ አትም (Print)</button>
        <button class="btn-add" onclick="downloadReceiptPDF('Receipt_${recId}')">📥 ፒዲኤፍ (PDF)</button>
        <button class="btn-config" style="background:#0088cc; color:white; grid-column: span 2;" onclick="shareToSocial('tg', \`${rawTextForShare}\`)">✈️ በቴሌግራም አጋራ</button>
        <button class="btn-expense" style="grid-column: span 2;" onclick="closeActiveModal()">❌ ዝጋ</button>
    </div>
    `;
    document.getElementById('formModalTitle').innerText = "🧾 የሽያጭ ደረሰኝ";
    document.getElementById('formModalBody').innerHTML = receiptHTML;
    document.getElementById('formModalFooter').innerHTML = '';
    openModalContainer(); document.getElementById('formModal').classList.remove('hidden');
}

function viewPastReceipt(idx) {
    let rec = currentTenant.data.receipts[idx];
    if(rec.advancedItems) { generateAdvancedReceipt(rec.advancedItems, rec.totalVal, rec.seller, rec.recId, false, rec.shopName, rec.bType, rec.buyerName, rec.buyerPhone); } 
    else { generateAdvancedReceipt([{name: rec.itemName, count: rec.count, unitPrice: rec.totalVal/rec.count, total: rec.totalVal}], rec.totalVal, rec.seller, rec.recId, false, rec.shopName, rec.bType, rec.buyerName, rec.buyerPhone); }
}

function generateDigitalReceipt(itemName, count, totalVal, recId = null, sellerRole = null, saveToHistory = true, buyerUserForReceipt = null, buyerPhoneForReceipt = null) {
    let items = [{name: itemName, count: count, unitPrice: totalVal/count, total: totalVal}];
    let currentSeller = sellerRole || (currentUserRole === 'staff' ? 'ሰራተኛ (Employee)' : 'ባለቤት (Employer)');
    generateAdvancedReceipt(items, totalVal, currentSeller, recId, saveToHistory, null, null, buyerUserForReceipt, buyerPhoneForReceipt);
}

