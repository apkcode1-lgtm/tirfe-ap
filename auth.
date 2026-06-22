// Email Verification State
var emailVerificationCode = "";
var pendingRegistrationData = null;
var pendingRegType = null; // 'buyer', 'tenant', 'forgot_pass'
var onVerifySuccess = null;

function openUnifiedLogin() {
    switchView('unifiedLoginBox');
    document.getElementById('loginUnifiedError').innerText = "";
    document.getElementById('loginUnifiedUser').value = "";
    document.getElementById('loginUnifiedEmail').value = "";
    document.getElementById('loginUnifiedPass').value = "";
}

function openUnifiedRegister() {
    switchView('unifiedRegisterBox');
    document.getElementById('unifiedRegRole').value = 'buyer';
    toggleUnifiedRegForm();
}

function toggleUnifiedRegForm() {
    let role = document.getElementById('unifiedRegRole').value;
    if(role === 'buyer') {
        document.getElementById('unifiedBuyerForm').classList.remove('hidden');
        document.getElementById('unifiedTenantForm').classList.add('hidden');
    } else {
        document.getElementById('unifiedBuyerForm').classList.add('hidden');
        document.getElementById('unifiedTenantForm').classList.remove('hidden');
    }
}

function autoFillPubCapitalFee() {
    let capital = document.getElementById('pub_newCapitalTier').value;
    let feeInput = document.getElementById('pub_newRegistrationFee');
    let tariffs = localDB.tariffs || { low: 500, medium: 1000, high: 2000 };
    
    if (capital === 'low') feeInput.value = tariffs.low;
    else if (capital === 'medium') feeInput.value = tariffs.medium;
    else if (capital === 'high') feeInput.value = tariffs.high;
    else feeInput.value = '';
}

function handleUnifiedLogin() {
    let user = document.getElementById('loginUnifiedUser').value.trim().toLowerCase();
    let email = document.getElementById('loginUnifiedEmail').value.trim();
    let pass = document.getElementById('loginUnifiedPass').value.trim();
    let err = document.getElementById('loginUnifiedError');

    if(!user || !email || !pass) { 
        err.innerText = "❌ እባክዎ ዩዘርኔም፣ ኢሜል እና የይለፍ ቃል በትክክል ያስገቡ!";
        return; 
    }

    if((user === "admin" || email === "apkcode1@gmail.com") && pass === "admin123") {
        localStorage.setItem('tirfe_active_session', JSON.stringify({ role: 'admin', loginMode: 'admin', username: 'admin' }));
        switchView('adminPage');
        renderAdminPanel();
        return;
    }

    if(localDB.tenants && localDB.tenants[user]) {
        let t = localDB.tenants[user];
        if(t.gmail === email && String(t.password).trim() === pass) {
            if(isTenantExpired(t, err)) return;
            currentUserRole = "owner";
            localStorage.setItem('tirfe_active_session', JSON.stringify({ role: 'owner', loginMode: 'merchant', username: user }));
            launchApp(t);
            return;
        }
    }

    if(localDB.buyers && localDB.buyers[user]) {
        let b = localDB.buyers[user];
        if(b.email === email && String(b.password).trim() === pass) {
            if(b.status === "blocked") { err.innerText = "❌ አካውንትዎ ታግዷል (Blocked)!"; return; }
            currentBuyer = b;
            localStorage.setItem('tirfe_active_session', JSON.stringify({ role: 'buyer', loginMode: 'buyer', username: user }));
            switchView('buyerPage');
            return;
        }
    }

    if(localDB.tenants) {
        for(let tKey in localDB.tenants) {
            let t = localDB.tenants[tKey];
            if(t.staffAccounts) {
                let found = t.staffAccounts.find(s => s.user === user && s.gmail === email && String(s.pass).trim() === pass);
                if(found) {
                    if (isTenantExpired(t, err)) return;
                    currentUserRole = "staff";
                    localStorage.setItem('tirfe_active_session', JSON.stringify({ role: 'staff', loginMode: 'staff', username: t.username }));
                    launchApp(t);
                    return;
                }
            }
        }
    }

    err.innerText = "❌ መረጃው ስህተት ነው! አካውንት አልተገኘም።";
}

function triggerUnifiedRegistration() {
    let role = document.getElementById('unifiedRegRole').value;
    if(role === 'buyer') {
        let name = document.getElementById('pubBuyerName').value.trim();
        let email = document.getElementById('pubBuyerEmail').value.trim();
        let phone = document.getElementById('pubBuyerPhone').value.trim();
        let user = document.getElementById('pubBuyerUser').value.trim().toLowerCase();

        if(!name || !email || !phone || !user) { showCustomAlert("ስህተት", "እባክዎ መረጃዎን ሙሉ በሙሉ ይሙሉ!"); return; }

        let takenMsg = isSystemDataTaken(user, phone, "", "");
        if(takenMsg) { showCustomAlert("ስህተት", takenMsg); return; }

        pendingRegType = 'buyer';
        pendingRegistrationData = { name, email, phone, user };
        triggerOTPFlow(email);
        
        onVerifySuccess = () => {
            showFormModal("🔒 የይለፍ ቃል ይፍጠሩ", [
                { id: "newPass", label: "ለአካውንትዎ አዲስ የይለፍ ቃል ይፍጠሩ፦", type: "password", placeholder: "ሚስጥራዊ ፓስዎርድ" }
            ], (res) => {
                if(!res.newPass) { showCustomAlert("ስህተት", "ፓስዎርድ አልፈጠሩም!"); return; }
                if(!localDB.buyers) localDB.buyers = {};
                
                localDB.buyers[pendingRegistrationData.user] = { 
                    username: pendingRegistrationData.user, phone: pendingRegistrationData.phone, 
                    name: pendingRegistrationData.name, email: pendingRegistrationData.email,
                    password: res.newPass, joinDate: new Date().getTime(), receipts: [], status: "active" 
                };
                pushToFirebase();
                showCustomAlert("✅ ተሳክቷል", "በተሳካ ሁኔታ ተመዝግበዋል! ወደ ዋናው ገጽ ይመለሳሉ።");
                switchView('welcomeGateway');
            });
        };
    } 
    else if(role === 'tenant') {
        let shop = document.getElementById('pub_newShopName').value.trim();
        let fullName = document.getElementById('pub_newFullName').value.trim();
        let user = document.getElementById('pub_newUsername').value.trim().toLowerCase();
        let phone = document.getElementById('pub_newPhone').value.trim();
        let newEmail = document.getElementById('pub_newEmail').value.trim();
        let telegram = document.getElementById('pub_newTelegram').value.trim();
        let mapsLink = document.getElementById('pub_newMapsLink').value.trim();
        let address = document.getElementById('pub_newAddress').value.trim();
        let businessType = document.getElementById('pub_newBusinessType').value.trim() || 'አጠቃላይ ንግድ';
        let registrationFee = parseFloat(document.getElementById('pub_newRegistrationFee').value) || 0;
        let contractType = document.getElementById('pub_newContractType').value;
        let expiryDate = document.getElementById('pub_newExpiryDate').value;
        
        if(!shop || !user || !expiryDate || !fullName || !phone || !newEmail) { 
            showCustomAlert("ስህተት", "እባክዎ መሠረታዊ መፈላጊ መረጃዎችን (ኢሜልን ጨምሮ) ያሟሉ!"); return; 
        }

        let checkUser = isSystemDataTaken(user, phone, "", "");
        if (checkUser) { showCustomAlert("⚠️ ምዝገባው አልተሳካም", checkUser); return; }

        let fileInput = document.getElementById('pub_newShopLogoFile');
        let file = fileInput ? fileInput.files[0] : null;

        pendingRegType = 'tenant';
        triggerOTPFlow(newEmail);
        onVerifySuccess = () => {
            showFormModal("🔒 የይለፍ ቃል ይፍጠሩ", [
                { id: "newPass", label: "ለሱቅዎ አዲስ ጠንካራ የይለፍ ቃል ይፍጠሩ፦", type: "password", placeholder: "ሚስጥራዊ ፓስዎርድ" }
            ], (res) => {
                if(!res.newPass) { showCustomAlert("ስህተት", "ፓስዎርድ አልፈጠሩም!"); return; }
                
                let proceedReg = function(shopLogoBase64) {
                    let timestampNow = new Date().getTime();
                    localDB.tenants[user] = { 
                        shopName: shop, fullName: fullName, phone: phone, telegram: telegram || "-", address: address || "-",
                        businessType: businessType, googleMapsLink: mapsLink || "", shopLogo: shopLogoBase64 || "", gmail: newEmail,
                        username: user, password: res.newPass, activationCode: res.newPass, codeCreatedAt: timestampNow,
                        isActivated: true, contractType: contractType, expiryDate: expiryDate, registrationFee: registrationFee,
                        status: "active", theme: "theme-deepblue", staffAccounts: [],
                        data: { sessionActive: false, shiftClosed: false, inventory: [], expenses: [], debts: [], drawerLog: [], history: [], receipts: [], deliveryOrders: [], remoteCarts: {}, lastMonthlyResetDate: timestampNow } 
                    };
                    pushToFirebase();
                    showCustomAlert("✅ ተሳክቷል", "ሱቅዎ በተሳካ ሁኔታ ተመዝግቧል! ወደ ዋናው ገጽ ይመለሳሉ።");
                    switchView('welcomeGateway');
                };
                
                if(file) processImageUpload(file, proceedReg); else proceedReg("");
            });
        };
    }
}

function triggerForgotPassword() {
    showFormModal("የይለፍ ቃል ማደሻ (Forgot Password)", [
        { id: "f_user", label: "የተጠቃሚ ስምዎን (Username) ያስገቡ፦", type: "text" },
        { id: "f_email", label: "የተመዘገቡበትን ኢሜል (Gmail) ያስገቡ፦", type: "email" }
    ], (res) => {
        let u = res.f_user.trim().toLowerCase();
        let e = res.f_email.trim();
        if(!u || !e) { showCustomAlert("ስህተት", "መረጃ አልሞሉም!"); return; }

        let foundAccount = null;
        let accType = '';
        
        if(localDB.tenants && localDB.tenants[u] && localDB.tenants[u].gmail === e) {
            foundAccount = localDB.tenants[u]; accType = 'tenant';
        } else if(localDB.buyers && localDB.buyers[u] && localDB.buyers[u].email === e) {
            foundAccount = localDB.buyers[u]; accType = 'buyer';
        }

        if(!foundAccount) { showCustomAlert("ስህተት", "በዚህ ዩዘርኔም እና ኢሜል የተመዘገበ አካውንት የለም!"); return; }

        pendingRegType = 'forgot_pass';
        triggerOTPFlow(e);
        onVerifySuccess = () => {
            showFormModal("🔑 አዲስ የይለፍ ቃል ማስተካከያ", [
                { id: "newPass", label: "አዲሱን የይለፍ ቃልዎን ያስገቡ፦", type: "password" }
            ], (resPass) => {
                let np = resPass.newPass.trim();
                if(!np) { showCustomAlert("ስህተት", "ባዶ መሆን አይችልም!"); return; }
                
                if(accType === 'tenant') {
                    localDB.tenants[u].password = np;
                } else if(accType === 'buyer') {
                    localDB.buyers[u].password = np;
                }
                pushToFirebase();
                showCustomAlert("✅ ተሳክቷል", "የይለፍ ቃልዎ በተሳካ ሁኔታ ተቀይሯል! አሁን በአዲሱ መግባት ይችላሉ።");
            });
        };
    });
}

function triggerOTPFlow(emailAddress) {
    emailVerificationCode = Math.floor(10000 + Math.random() * 90000).toString();
    document.getElementById('verifyEmailDisplay').innerText = emailAddress;
    
    openModalContainer();
    document.getElementById('emailVerifyModal').classList.remove('hidden');
    for(let i=1; i<=5; i++) document.getElementById('code'+i).value = '';
    document.getElementById('code1').focus();
    setTimeout(() => {
        alert(`[ማሳሰቢያ]: ለሙከራ ጊዜያዊ የኢሜል ኮድዎ: ${emailVerificationCode} ነው!`);
    }, 500);
}

window.resendOTP = function() {
    let currentEmail = document.getElementById('verifyEmailDisplay').innerText;
    emailVerificationCode = Math.floor(10000 + Math.random() * 90000).toString();
    showCustomAlert("✅ ተልኳል", "አዲስ ማረጋገጫ ኮድ ተልኳል።");
    setTimeout(() => {
        alert(`[ማሳሰቢያ]: አዲሱ የኢሜል ኮድዎ: ${emailVerificationCode} ነው!`);
    }, 500);
};

function moveToNext(current, nextFieldID) {
    if (current.value.length >= 1) {
        if (nextFieldID) { document.getElementById(nextFieldID).focus(); } 
        else { current.blur(); }
    }
}

function moveToPrev(e, current, prevFieldID) {
    if (e.key === "Backspace" && current.value === "") {
        if (prevFieldID) {
            document.getElementById(prevFieldID).focus();
            document.getElementById(prevFieldID).value = '';
        }
    }
}

function verifyEmailCodeSubmit() {
    let enteredCode = "";
    for(let i=1; i<=5; i++) { enteredCode += document.getElementById('code'+i).value; }

    if (enteredCode === emailVerificationCode) {
        closeActiveModal();
        if(onVerifySuccess) onVerifySuccess();
    } else {
        showCustomAlert("❌ ስህተት", "ያስገቡት ማረጋገጫ ኮድ የተሳሳተ ነው!");
    }
}

